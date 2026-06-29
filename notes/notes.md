Day 1 6/13/2026:

## PHASE 1

- Understanding MP-BGP EVPN as the CONTROL PLANE rather than anything to do with the data plane, this confused me at first
- Getting used to BGP in general
- Understanding the spine leaf architecture and why companies like microsoft use different ASNs per device following RFC 7938, microsoft used unique ASN per TOR.


Challenge 1: Understanding NX-OS

I had a short stint working with IOS-XR at Conterra and it seems NXOS uses the same nested configuration, confusing when coming from pure IOS.

## Topology:

- 2 Leaves
- 2 Spines

Day 2 6/14/2026

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


## Phase 1 Troubleshooting Notes

-  I wired it up wrong, had the ports switched around causing some of the BGP sessions not to come up

- ECMP wasnt coming up, then I realized I needed   bestpath as-path multipath-relax,  On Cisco, By default, BGP only considers paths "equal" for ECMP if they have identical AS_PATH attributes — same length and same ASNs in the same positions.

-   Understanding the iBGP full mesh problem:

When I first learned about BGP, I had trouble realizing the difference between iBGP and eBGP, until I fully understood the AS_PATH loop prevention mechanism and the impact it would have if every device was in the same ASN. If the devices are in the same ASN, the route that is propogated within the network would be registered as a loop UNLESS every single router forms an iBGP session with every other router (ie full mesh), this is obviously not feasable in large networks as the sessions needed would exponentially grow for every other router. Thats also when the need for route refelctors clicked for me, we could designate one (or two) routers as the only devices allowed to propogate these routes, effectively solving the AS_PATH loop problem. Although, this comes with its own set of problems, control plane single points of failure if not redundant, suboptimal path selection due to hjow iBGP path selection differs from eBGP, etc.

Hyperscalers solved this with RFC 7938 published by Meta in 2010 - 2013 (I was far from being a network engineer then).


My next logical question was why not just use a link state protocol like OSPF? Well some people do, but when I looked further into it I realized that one of the philosophys of SONiC was "BGP Everywhere" and thats why most dont, and I still dont know why as I am not familiar with SONiC (I will be learning it soon). I decided to stick with eBGP for my lab to follow RFC 7938. If anyone could help me understand this please do.

  ## Why eBGP-per-device in the underlay

iBGP has a full-mesh requirement (or needs route reflectors), which
doesn't scale to hundreds of leaves. eBGP with per-device ASNs avoids
this because:
- Every adjacent device is "external" by definition
- AS_PATH loop prevention works naturally
- No re-advertisement restrictions
- Adding a leaf = configure local device only, no RR config
- One protocol for underlay AND overlay (we'll use BGP for EVPN too)

Cost: each device needs a unique ASN, which requires planning, and
ECMP needs multipath-relax (see below).

## ECMP and multipath-relax

BGP by default treats two paths as multipath-equal only if their
AS_PATHs are *identical* (same length AND same ASNs). In a
per-device-ASN fabric, every path through a different spine has
different ASNs, so paths tie on length but differ in content.
Result: BGP installs only one path, silently halving fabric bandwidth.

Fix: `bestpath as-path multipath-relax` under the BGP address family.
This tells BGP "ignore AS_PATH content for multipath, only compare
length." With this command, paths with same-length AS_PATHs are
considered equal-cost and both install with `maximum-paths N`.

Required on every device in a per-device-ASN fabric. SONiC and
Cumulus default it on. NX-OS does not.

## BGP best-path selection (memorize this)

1. WEIGHT (Cisco proprietary)
2. LOCAL_PREF
3. Locally-originated
4. AS_PATH length
5. ORIGIN type
6. MED
7. eBGP > iBGP
8. IGP metric to next-hop
9. Oldest (some platforms)
10. Lowest router-id
11. Lowest neighbor IP



https://datatracker.ietf.org/doc/html/rfc7938)





6/28/2026

## Phase 2 Notes


Life happened and this project sat for a bit. Picking it back up and configuring the fabric to advertise the L2VPN EVPN address family on the existing eBGP sessions.

When I first learned BGP I was taught it was for IPv4 routes, which is what RFC 4271 (the original BGP-4 spec) actually specifies. RFC 4760 (Multiprotocol Extensions for BGP-4) added the MP_REACH_NLRI attribute, which carries an (AFI, SAFI) tag identifying what kind of reachability info is inside. EVPN is AFI 25, SAFI 70. This is the same protocol and the same sessions, BGP is now set up to advertise additional payloads, in this case that would be MAC addresses for VXLAN.

I have always found that learning networking technologies from the original specifications first is the best way to learn them, because when you learn the original technology, you learn WHY these improvements were made and WHAT problems they solve. Without doing it this way, you end up confused about why we do things the way we do in present day.

In this phase, I am just setting up the L2VPN EVPN Address Family, in phase 3 I will actually start sharing MAC addresses between peers and test reachability between two hosts.

## Phase 3a Notes

In phase 3 I configured the VLANs, loopbacks for the NVE interface, BGP reachability for Loopback 1, the NVE and ingress-replication to bypass multicast and the EVPN settings and route descriptors / route targets

One problem I ran into is when I checked the Pfxreceived using "show bgp l2vpn evpn neighbors 10.1.4.1 advertised-routes" on leaf 2 it showed 0, which was strange. Upon doing some research I realized it was due to the spines dropping the RT because they did not have anything to associate it with since they are just underlay transit. I added "retain route-target all" to the l2vpn evpn address family bgp configuration on both spines to explicitly tell them not to drop any route targets.

## Phase 3b Notes

Well the leafs are learning the hosts MACS and sharing them via BGP, but they still cannot ping each other. Its 9 AM and I have work in the morning so I am going to have to work on this again another day. I am updating the configs and notes. 