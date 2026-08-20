# Site-to-Site VPN

**Tools used:** GNS3, Cisco IOSv, VPCS
**Skills demonstrated:** IPsec (IKEv1) site-to-site VPN configuration, crypto ACLs, static routing, VPN troubleshooting

---

## 1. Objective

Simulate two branch offices connected over a WAN link, with an IPsec site-to-site VPN tunnel encrypting all traffic between them — the standard way two physical office locations securely share a private network over the public internet.

**Requirements I set for this build:**
- Two independent branch LANs, each behind its own router
- A simulated WAN link between the branches (public-style addressing)
- An IPsec tunnel that encrypts only traffic between the two branch LANs, verified with real SA/counter output, not just a successful ping

---

## 2. Diagram

![Topology](Topology.SVG)

**Layout:** `PC-A — Switch-A — Router-A === (WAN link) === Router-B — Switch-B — PC-B`

Two branch routers connected directly over a simulated WAN link, each with a local switch and PC representing that branch's internal LAN.

---

## 3. What I Built

### IP Addressing Plan

| Segment | Subnet | Notes |
|---|---|---|
| Branch-A LAN | 10.1.1.0/24 | Gateway 10.1.1.1 |
| Branch-B LAN | 10.2.2.0/24 | Gateway 10.2.2.1 |
| WAN link (A↔B) | 203.0.113.0/30 | Router-A: .1, Router-B: .2 (simulating public IPs) |

### IPsec Configuration (3 building blocks)

**1. ISAKMP (IKE Phase 1)** — how the two routers authenticate each other and protect their negotiation:
```
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key VpnSecretKey123 address <peer WAN IP>
```

**2. IPsec Transform Set (IKE Phase 2)** — how the actual data is encrypted:
```
crypto ipsec transform-set TS esp-aes 256 esp-sha256-hmac
 mode tunnel
```

**3. Interesting traffic ACL + Crypto map** — defines which traffic gets encrypted and binds the policy to an interface. Mirrored on both routers (source/destination swapped):

Router-A:
```
ip access-list extended VPN-TRAFFIC
 permit ip 10.1.1.0 0.0.0.255 10.2.2.0 0.0.0.255

crypto map VPNMAP 10 ipsec-isakmp
 set peer 203.0.113.2
 set transform-set TS
 match address VPN-TRAFFIC

interface GigabitEthernet0/0
 crypto map VPNMAP
```

Router-B:
```
ip access-list extended VPN-TRAFFIC
 permit ip 10.2.2.0 0.0.0.255 10.1.1.0 0.0.0.255

crypto map VPNMAP 10 ipsec-isakmp
 set peer 203.0.113.1
 set transform-set TS
 match address VPN-TRAFFIC

interface GigabitEthernet0/0
 crypto map VPNMAP
```

**Static routes** (required for each router to know how to reach the remote LAN across the WAN link — see Problems section):
```
! Router-A
ip route 10.2.2.0 255.255.255.0 203.0.113.2

! Router-B
ip route 10.1.1.0 255.255.255.0 203.0.113.1
```

### Why the interesting-traffic ACL matters

The ACL isn't an access-control gate — it's a **traffic classifier**. Packets matching it get picked up by the crypto map, encrypted, and sent through the tunnel; packets that don't match route normally, unencrypted, as if the VPN doesn't exist. Both routers have to agree on what "interesting traffic" is — Router-A defines LAN-A→LAN-B as interesting, Router-B must mirror that as LAN-B→LAN-A. If the two ACLs don't match exactly, the tunnel can fail to negotiate at all, or come up but only protect traffic asymmetrically — a well-known real-world IPsec misconfiguration.

---

## 4. Problems I Hit and How I Fixed Them

**Missing static routes.** After configuring ISAKMP, the transform set, and the crypto maps, the first ping from PC-A to PC-B failed with a "Destination host unreachable" from Router-A's own gateway (10.1.1.1) — meaning the packet never even left Router-A. Checking `show crypto isakmp sa` confirmed no negotiation had started at all.

The cause: the WAN link between the routers was directly connected and working, but neither router had a route telling it *how to reach the other side's LAN* through that link. VPN configuration alone doesn't create IP routing — the router still needs to know 10.2.2.0/24 is reachable via 203.0.113.2 before it can even attempt to forward (and therefore encrypt) traffic toward it. Adding static routes on both routers (`ip route 10.2.2.0 255.255.255.0 203.0.113.2` on Router-A, and the mirror on Router-B) resolved it immediately — the tunnel negotiated on the next ping and came up `QM_IDLE`/`ACTIVE`.

**Interface identification.** Before applying the crypto map, I double-checked which physical interface on each router actually faced the WAN link (Gi0/0 on both routers in this topology) rather than assuming based on interface numbering alone — applying a crypto map to the wrong interface would have silently prevented the tunnel from ever triggering.

---

## 5. What I'd Do Differently at Scale

- **PFS (Perfect Forward Secrecy):** current config shows `PFS (Y/N): N` in the IPsec SA output. At scale/production, I'd enable PFS (`set pfs group14` on the crypto map) so each new IPsec SA generates a fresh Diffie-Hellman exchange — meaning a compromise of one session's keys doesn't expose past or future sessions.
- **Point-to-point doesn't scale past a few sites.** This design is one static tunnel between exactly two routers. With more than a handful of branch offices, I'd move to a hub-and-spoke or full-mesh design using **DMVPN** or a dedicated VPN concentrator, rather than manually configuring a separate point-to-point tunnel for every pair of sites.
- **Pre-shared key → certificate-based authentication.** A shared pre-shared key is fine for a lab; in production I'd move to certificate-based IKE authentication, which scales better and avoids a single shared secret across sites.
- **Routing:** static routes work for two sites but don't scale — with more sites I'd run a dynamic routing protocol over the tunnels (e.g., via DMVPN with EIGRP/OSPF, or route-based VPN with BGP) instead of static routes per site pair.

---

## 6. Verification Output

**`show crypto isakmp sa`** (Router-A):
```
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
203.0.113.2     203.0.113.1     QM_IDLE           1001 ACTIVE
IPv6 Crypto ISAKMP SA
```
`QM_IDLE` / `ACTIVE` confirms Phase 1 (ISAKMP) completed successfully and the tunnel is up and idle, ready to protect traffic.

**`show crypto ipsec sa`** (Router-A):
```
interface: GigabitEthernet0/0
    Crypto map tag: VPNMAP, local addr 203.0.113.1
   protected vrf: (none)
   local  ident (addr/mask/prot/port): (10.1.1.0/255.255.255.0/0/0)
   remote ident (addr/mask/prot/port): (10.2.2.0/255.255.255.0/0/0)
   current_peer 203.0.113.2 port 500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 9, #pkts encrypt: 9, #pkts digest: 9
    #pkts decaps: 9, #pkts decrypt: 9, #pkts verify: 9
    #send errors 0, #recv errors 0
     local crypto endpt.: 203.0.113.1, remote crypto endpt.: 203.0.113.2
     current outbound spi: 0xA03EA07E(2688458878)
     PFS (Y/N): N, DH group: none
     inbound esp sas:
      spi: 0x7AF7DB33(2063063859)
        transform: esp-256-aes esp-sha256-hmac
        in use settings ={Tunnel, }
        Status: ACTIVE(ACTIVE)
     outbound esp sas:
      spi: 0xA03EA07E(2688458878)
        transform: esp-256-aes esp-sha256-hmac
        in use settings ={Tunnel, }
        Status: ACTIVE(ACTIVE)
```
Matching non-zero `encaps`/`decaps` counters on both inbound and outbound ESP SAs confirm the tunnel is actively encrypting and decrypting real traffic between PC-A and PC-B, not just idle/negotiated.

---

## Setup / How to Reproduce

1. Open in GNS3 (project file included in this repo).
2. Start all nodes, wait for both routers to fully boot.
3. Router configs are in the `configs/` folder — apply via console or paste directly.
4. Set VPCS IPs: PC-A `10.1.1.20 255.255.255.0 10.1.1.1`, PC-B `10.2.2.20 255.255.255.0 10.2.2.1`.
5. Ping PC-A → PC-B to trigger tunnel negotiation, then verify with `show crypto isakmp sa` and `show crypto ipsec sa`.
