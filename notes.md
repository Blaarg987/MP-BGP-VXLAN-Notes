Day 1 6/13/2026:

- Understanding MP-BGP EVPN as the CONTROL PLANE rather than anything to do with the data plane, this confused me at first
- Getting used to BGP in general
- Understanding the spine leaf architecture and why companies like microsoft use different ASNs per device following RFC 7938, microsoft used unique ASN per TOR.

Claude says:
Why the choice matters — the AS_PATH loop trapHere's the core mechanic that drives the choice. BGP's loop prevention is AS_PATH: if a router sees its own ASN in the AS_PATH of an incoming UPDATE, it rejects the route as a loop. This is great for inter-domain routing, but it causes weird problems inside a fabric.

Challenge 1: Understanding NX-OS

I had a short stint working with IOS-XR at Conterra and it seems NXOS uses the same nested configuration, confusing when coming from pure IOS.

Topology:

- 2 Leaves
- 2 Spines

Phase 0/1: Underlay IP plan

Loopbacks:
  Spine1: 10.0.0.1/32
  Spine2: 10.0.0.2/32
  Leaf1:  10.0.0.11/32
  Leaf2:  10.0.0.12/32

P2P links (spine .1, leaf .2):
  Sp1-E1/1 10.1.1.1 <-> L1-E1/1 10.1.1.2  (10.1.1.0/30)
  Sp1-E1/2 10.1.2.1 <-> L2-E1/1 10.1.2.2  (10.1.2.0/30)
  Sp2-E1/1 10.1.3.1 <-> L1-E1/2 10.1.3.2  (10.1.3.0/30)
  Sp2-E1/2 10.1.4.1 <-> L2-E1/2 10.1.4.2  (10.1.4.0/30)