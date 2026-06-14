# EVPN/VXLAN Fabric Lab

A leaf-spine VXLAN fabric built in Cisco Modeling Labs on a Dell R720
home lab, modeled on hyperscaler-style data center architecture per
[RFC 7938](https://datatracker.ietf.org/doc/html/rfc7938). The goal is
to learn the modern MP-BGP EVPN control plane and VXLAN data plane
end-to-end by configuring every device by hand.

## Status

- [x] Phase 0: BGP refresher on NX-OS
- [x] Phase 1: Underlay eBGP fabric with ECMP across spines
- [ ] Phase 2: Activate L2VPN EVPN address family on existing BGP sessions
- [ ] Phase 3: L2 VXLAN with EVPN type-2 routes, hosts in same VNI
- [ ] Phase 4: Asymmetric IRB for inter-VNI L3
- [ ] Phase 5: Symmetric IRB with transit VNI (production model)

## Topology

```
                    ┌──────────────┐         ┌──────────────┐
                    │   Spine1     │         │   Spine2     │
                    │   AS 65100   │         │   AS 65101   │
                    │ Lo0: 10.0.0.1│         │ Lo0: 10.0.0.2│
                    └──┬────────┬──┘         └──┬────────┬──┘
                       │        │               │        │
                       │        └───────┐ ┌─────┘        │
                       │                ╳ ╳              │
                       │        ┌───────┘ └─────┐        │
                       │        │               │        │
                    ┌──┴────────┴──┐         ┌──┴────────┴──┐
                    │    Leaf1     │         │    Leaf2     │
                    │   AS 65001   │         │   AS 65002   │
                    │Lo0: 10.0.0.11│         │Lo0: 10.0.0.12│
                    │Lo1:10.0.0.101│         │Lo1:10.0.0.102│
                    │   (VTEP)     │         │   (VTEP)     │
                    └──────┬───────┘         └──────┬───────┘
                           │ E1/3                   │ E1/3
                           │                        │
                       ┌───┴────┐               ┌───┴────┐
                       │ Host1  │               │ Host2  │
                       │ Alpine │               │ Alpine │
                       │VLAN 10 │               │VLAN 10 │
                       │.10/24  │               │.20/24  │
                       └────────┘               └────────┘
```

## Device Summary

| Device | ASN   | Lo0 (BGP RID)  | Lo1 (VTEP source) | Role             |
|--------|-------|----------------|-------------------|------------------|
| Spine1 | 65100 | 10.0.0.1/32    | —                 | Underlay routing |
| Spine2 | 65101 | 10.0.0.2/32    | —                 | Underlay routing |
| Leaf1  | 65001 | 10.0.0.11/32   | 10.0.0.101/32     | VTEP             |
| Leaf2  | 65002 | 10.0.0.12/32   | 10.0.0.102/32     | VTEP             |

## Underlay Links

eBGP sessions, IPv4 unicast address family. Spine end always `.1`,
leaf end always `.2`.

| Link            | Subnet       | Spine Side      | Leaf Side       |
|-----------------|--------------|-----------------|-----------------|
| Spine1 ↔ Leaf1  | 10.1.1.0/30  | E1/1: 10.1.1.1  | E1/1: 10.1.1.2  |
| Spine1 ↔ Leaf2  | 10.1.2.0/30  | E1/2: 10.1.2.1  | E1/1: 10.1.2.2  |
| Spine2 ↔ Leaf1  | 10.1.3.0/30  | E1/1: 10.1.3.1  | E1/2: 10.1.3.2  |
| Spine2 ↔ Leaf2  | 10.1.4.0/30  | E1/2: 10.1.4.1  | E1/2: 10.1.4.2  |

## BGP Session Matrix

Each leaf forms two eBGP sessions, one with each spine, peering on the
directly-connected P2P address (no `update-source loopback`, no
`ebgp-multihop`). Loopbacks are advertised as routes, not used as session
endpoints.

| Session                | Leaf  | Spine  | Peering Subnet | Address Families     |
|------------------------|-------|--------|----------------|----------------------|
| Leaf1 ↔ Spine1         | 65001 | 65100  | 10.1.1.0/30    | IPv4 (+ EVPN later)  |
| Leaf1 ↔ Spine2         | 65001 | 65101  | 10.1.3.0/30    | IPv4 (+ EVPN later)  |
| Leaf2 ↔ Spine1         | 65002 | 65100  | 10.1.2.0/30    | IPv4 (+ EVPN later)  |
| Leaf2 ↔ Spine2         | 65002 | 65101  | 10.1.4.0/30    | IPv4 (+ EVPN later)  |

## Design Decisions

**Per-device ASNs (RFC 7938).** Every router has a unique ASN rather
than sharing an ASN per layer. This avoids the AS_PATH loop issue that
would otherwise require `allowas-in` workarounds, and aligns with how
Microsoft, Meta, and other hyperscalers actually build their fabrics.

**eBGP everywhere, no OSPF/IS-IS underlay.** A single protocol for both
underlay and overlay (EVPN) means one set of operational tooling and
no protocol redistribution. Adding a new leaf is a local config change
on one device — no route reflectors to update.

**ECMP requires `bestpath as-path multipath-relax`.** By default, BGP
only considers paths equal-cost if their AS_PATHs match exactly. In a
per-device-ASN fabric, paths to the same destination via different
spines have different ASNs (e.g., `65100 65001` vs `65101 65001`), so
BGP installs only one path and you silently lose half your fabric
bandwidth. The `multipath-relax` command tells BGP to compare AS_PATH
*length* only, not content. Required on every device.

## Repository Contents

- `configs/` — per-device running configurations
- `topology/` — CML topology file and diagram source
- `notes/` — phase-by-phase lessons learned and troubleshooting writeups

## References

- [RFC 7938 — Use of BGP for Routing in Large-Scale Data Centers](https://datatracker.ietf.org/doc/html/rfc7938)
- [RFC 8365 — A Network Virtualization Overlay Solution Using EVPN](https://datatracker.ietf.org/doc/html/rfc8365)
- Dinesh Dutt, *BGP in the Data Center* (free from Nvidia)
- Dinesh Dutt, *Cloud Native Data Center Networking* (O'Reilly)
