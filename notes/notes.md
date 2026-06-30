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

Well the leafs are learning the hosts MACS and sharing them via BGP, but they still cannot ping each other. Its 9 PM and I have work in the morning so I am going to have to work on this again another day. I am updating the configs and notes. 



06/29/2026

I tried to figure out why BUM traffic isn't working for this L2VPN. I verified:

1. Both MAC address tables are dynamically populated with both hosts
2. The control plane is working as both type-2 and type-3 routes populated
3. The hosts cannot resolve ARP

This indicates a platform level bug with replicated BUM traffic. Everything else is working so I am just going to move forward with phase 4.


## Phase 4 Notes

In this phase I am configuring Integrated Routing and Bridging using EVPN VXLAN with anycast gateways to try and mimic a true datacenter topology. 

Since L2VPN BUM replication did not work I am changing the plan slightly by doing the following:

1. Moving host 2 to a new VLAN (VLAN 20) with IP 192.168.20.20 and using a gateway of 192.168.20.1 &adding a new VNI 10020 for VLAN 20
2. Both leaves get SVIs for both VLANs with anycast addresses


So at first I was confused about the difference between asymetric and symetric IRB but after doing some more reading I understand why we use symetric IRB these days. In Asymetric when a host wants to talk to another host in a different subnet via an anycast gateway the leaf node has to ARP for the MAC of the destination IP, this causes unnesecary traffic to each leaf node participating in that VNI. Additionally, in symetric MPBGP includes the MAC+IP information in the type 2 route type so ARP is unnecesarry, making everything just a routing decision until it gets to the target leaf node. Also, symetric IRB makes use of the type 5 route, which is IP prefix information, making BGP tables smaller which makes the memory use more efficient. 




I was having some trouble understanding the terminology hierarchy in MPBGP so I asked claude to summarize it for me and this is what it came up with. I am Pasting the entire summary here as it helped me understand the structure as I am still new to BGP:

The hierarchy

BGP UPDATE
├── Withdrawn routes (legacy IPv4 section, usually empty in MP-BGP)
├── Path attributes
│   ├── ORIGIN
│   ├── AS_PATH
│   ├── NEXT_HOP (for legacy IPv4 section, often unused)
│   ├── LOCAL_PREF
│   ├── COMMUNITIES (regular)
│   ├── EXTENDED_COMMUNITIES   ← Route Target, MAC Mobility, Router MAC live here
│   └── MP_REACH_NLRI
│       ├── AFI = 25 (L2VPN)
│       ├── SAFI = 70 (EVPN)
│       ├── Next-hop (the VTEP IP)
│       └── NLRI bytes ← THE EVPN ROUTE LIVES HERE
│           ├── Route Type (1 byte: 2 for MAC/IP, 3 for IMET, 5 for prefix, etc.)
│           ├── Length
│           └── Route-type-specific fields
└── (NLRI section, legacy IPv4, usually empty in MP-BGP)
So when you see in show bgp l2vpn evpn:
*>l[2]:[0]:[0]:[48]:[5254.0019.3ef6]:[0]:[0.0.0.0]/216
That whole bracketed string IS the EVPN NLRI, parsed and displayed by NX-OS in human-readable form. Inside the actual UPDATE message, those brackets are binary bytes:

[2] = route type byte (type-2)
[0] = Ethernet Segment Identifier (zero = no multihoming)
[0] = Ethernet Tag ID
[48] = MAC address length in bits
[5254.0019.3ef6] = the MAC bytes
[0]:[0.0.0.0] = IP address length (zero) and IP (no IP because length is zero)
The trailing /216 = total NLRI length in bits (Note: 216 bits / 8 = 27 bytes — that's the entire encoded type-2 NLRI structure)

Different route types have different field layouts within their NLRI:

Type-2 NLRI: RD, ESI, Ethernet Tag, MAC length, MAC, IP length, IP, MPLS label (sometimes)
Type-3 NLRI: RD, Ethernet Tag, IP length, Originating Router IP
Type-5 NLRI: RD, ESI, Ethernet Tag, IP prefix length, IP prefix, GW IP, MPLS label/VNI

But all of them are encoded as bytes inside the MP_REACH_NLRI attribute. The first byte after the length field is always the route type, which tells the parser which field layout to use.
How NX-OS displays it vs. what's on the wire
What you see:
Route Distinguisher: 10.0.0.11:32777    (L2VNI 10010)
*>l[2]:[0]:[0]:[48]:[5254.0019.3ef6]:[0]:[0.0.0.0]/216
                      10.0.0.101                        100      32768 i
What's actually in the BGP UPDATE message:
BGP UPDATE:
  Path Attributes:
    ORIGIN: IGP
    AS_PATH: (empty, local origination)
    LOCAL_PREF: 100
    EXTENDED_COMMUNITIES:
      Route Target: 65000:10010
      Encapsulation Type: VXLAN
    MP_REACH_NLRI:
      AFI: 25
      SAFI: 70
      Next-hop: 10.0.0.101
      NLRI:
        Route Type: 2
        Length: 33 (or whatever the byte count is)
        Route Distinguisher: 10.0.0.11:32777
        ESI: 0
        Ethernet Tag: 0
        MAC Length: 48
        MAC: 52:54:00:19:3e:f6
        IP Length: 0
        (no IP, since length is 0)
        MPLS Label / VNI: 10010


I also asked claude to compare and contrast the structure between an UPDATE message container EVPN information and a regular IPv4 Unicast update, it gave me information on the differences between the original BGP format and the newer MPBGP format (RFC 4760) which really helped me understand the changed on a deeper level:

The legacy IPv4 unicast UPDATE format (RFC 4271)
The original BGP UPDATE message has this fixed structure:
BGP UPDATE Message:
├── Withdrawn Routes Length (2 bytes)
├── Withdrawn Routes (variable) ← IPv4 prefixes being withdrawn
├── Total Path Attribute Length (2 bytes)
├── Path Attributes (variable)
│   ├── ORIGIN
│   ├── AS_PATH
│   ├── NEXT_HOP (4 bytes: IPv4 address)
│   ├── (optional: MED, LOCAL_PREF, COMMUNITIES, etc.)
│   └── (note: no MP_REACH_NLRI present in pure legacy)
└── Network Layer Reachability Information (variable) ← IPv4 prefixes being advertised
Notice three things about this structure:

The NLRI sits OUTSIDE the path attributes, at the very end of the message
The NLRI is implicitly IPv4 — the format is (prefix length, prefix bytes) and the parser assumes IPv4
NEXT_HOP is its own dedicated path attribute containing 4 bytes of IPv4 address

A concrete example. Imagine R1 in AS 100 advertising 10.0.0.0/24 to R2, with itself as next-hop 192.168.1.1:
BGP UPDATE:
├── Withdrawn Routes Length: 0
├── Path Attribute Length: 23
├── Path Attributes:
│   ├── ORIGIN: IGP (3 bytes total including TLV header)
│   ├── AS_PATH: [100] (7 bytes)
│   └── NEXT_HOP: 192.168.1.1 (7 bytes)
└── NLRI:
    └── 10.0.0.0/24 (4 bytes: length byte + 3 prefix bytes)
The parser knows the NLRI bytes mean IPv4 because there's nowhere else they could mean.
The MP-BGP format (RFC 4760)
MP-BGP doesn't change the UPDATE message structure — it adds two new path attribute types and moves the reachability information inside one of them:
BGP UPDATE Message:
├── Withdrawn Routes Length (2 bytes) — typically 0 in MP-BGP
├── Withdrawn Routes (variable) — typically empty
├── Total Path Attribute Length (2 bytes)
├── Path Attributes (variable)
│   ├── ORIGIN
│   ├── AS_PATH
│   ├── (LOCAL_PREF, communities, etc.)
│   ├── EXTENDED_COMMUNITIES (for VPN/EVPN: RT, Router MAC, etc.)
│   └── MP_REACH_NLRI ← reachability info moves INSIDE here
│       ├── AFI (2 bytes): which address family
│       ├── SAFI (1 byte): which subsequent address family
│       ├── Next-Hop Length (1 byte)
│       ├── Next-Hop (variable: format depends on AFI)
│       ├── Reserved (1 byte)
│       └── NLRI bytes (variable: format depends on (AFI, SAFI))
└── NLRI (typically empty in MP-BGP)
So R1 advertising 10.0.0.0/24 via MP-BGP looks like:
BGP UPDATE:
├── Withdrawn Routes Length: 0
├── Path Attribute Length: 32
├── Path Attributes:
│   ├── ORIGIN: IGP
│   ├── AS_PATH: [100]
│   └── MP_REACH_NLRI:
│       ├── AFI: 1 (IPv4)
│       ├── SAFI: 1 (unicast)
│       ├── Next-Hop Length: 4
│       ├── Next-Hop: 192.168.1.1
│       ├── Reserved: 0
│       └── NLRI: 10.0.0.0/24
└── NLRI: (empty)
Same information, different location. Notice:

NLRI is now inside the MP_REACH_NLRI attribute
Next-hop is also inside MP_REACH_NLRI (no separate NEXT_HOP attribute needed for the IPv4 routes in this message)
The (AFI=1, SAFI=1) tag is explicit — the parser doesn't have to assume the NLRI is IPv4; the tag says so

And the EVPN version
Same UPDATE structure, just with different (AFI, SAFI) and different NLRI byte layout:
BGP UPDATE:
├── Withdrawn Routes Length: 0
├── Path Attribute Length: ~80
├── Path Attributes:
│   ├── ORIGIN: IGP
│   ├── AS_PATH: [65001]
│   ├── EXTENDED_COMMUNITIES:
│   │   ├── Route Target: 65000:10010
│   │   └── Encapsulation Type: VXLAN
│   └── MP_REACH_NLRI:
│       ├── AFI: 25 (L2VPN)
│       ├── SAFI: 70 (EVPN)
│       ├── Next-Hop Length: 4
│       ├── Next-Hop: 10.0.0.101 (VTEP IP)
│       ├── Reserved: 0
│       └── NLRI:
│           └── Type-2 route:
│               ├── Route Type: 2
│               ├── Length: 33
│               ├── RD: 10.0.0.11:32777
│               ├── ESI: 0
│               ├── Ethernet Tag: 0
│               ├── MAC Length: 48
│               ├── MAC: 52:54:00:19:3e:f6
│               ├── IP Length: 0
│               └── VNI: 10010
└── NLRI: (empty)
Three structural observations to make this concrete:
1. Path attribute structure is identical for both. ORIGIN, AS_PATH, etc. exist in both messages and mean the same thing. They apply to whatever reachability info is in the message.
2. The only thing that changes about reachability between IPv4 and EVPN is the (AFI, SAFI) tag and the NLRI bytes. Everything else — the BGP state machine, the TCP session, the path attribute model — is identical.
3. Extended communities are where EVPN-specific metadata lives. Route Target, Router MAC, MAC Mobility, Encapsulation Type — these are path attributes, ride alongside the NLRI, and tell receivers what to do with the route.
Side-by-side view

BGP UPDATE has a fixed top-level structure: withdrawn routes section, path attributes section, NLRI section

Path attributes is a list of (type, length, value) TLVs
MP_REACH_NLRI is one of those TLVs — type number 14

Inside MP_REACH_NLRI are the AFI, SAFI, next-hop, and the actual reachability bytes (which are parsed based on the AFI+SAFI tag)

All other path attributes (AS_PATH, ORIGIN, communities, etc.) live as siblings to MP_REACH_NLRI in the path attributes section

For EVPN, the extended communities path attribute carries the EVPN-specific metadata (RT, router MAC, MAC mobility, etc.)



Given all this, I am going to have to skip phase 4 and proceed directly to phase 5 and configure symetric IRB because on CML the dataplane is broken so ARP cannot resolve, hence asymetric IRB will not work.



## Phase 5 Notes


I got the config up and the data plane / control plane working but I still cannot ping, possibly another limitation of CML?


Output of VRF route table:

Leaf1# show ip route vrf TENANT-A 
IP Route Table for VRF "TENANT-A"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

192.168.10.0/24, ubest/mbest: 1/0, attached
    *via 192.168.10.1, Vlan10, [0/0], 00:10:34, direct
192.168.10.1/32, ubest/mbest: 1/0, attached
    *via 192.168.10.1, Vlan10, [0/0], 00:10:34, local
192.168.20.0/24, ubest/mbest: 1/0, attached
    *via 192.168.20.1, Vlan20, [0/0], 00:10:27, direct
192.168.20.1/32, ubest/mbest: 1/0, attached
    *via 192.168.20.1, Vlan20, [0/0], 00:10:28, local
192.168.20.20/32, ubest/mbest: 1/0
    *via 10.1.1.1%default, [20/0], 00:01:21, bgp-65001, external, tag 65100,    segid: 50000 tunnelid: 0xa010101 encap: VXLAN  <--- This route here confirms it has been learned


Leaf1# show ip arp vrf TENANT-A

Flags: * - Adjacencies learnt on non-active FHRP router
       + - Adjacencies synced via CFSoE
       # - Adjacencies Throttled for Glean
       CP - Added via L2RIB, Control plane Adjacencies
       PS - Added via L2RIB, Peer Sync
       RO - Re-Originated Peer Sync Entry
       D - Static Adjacencies attached to down interface

IP ARP Table for context TENANT-A
Total number of entries: 1
Address         Age       MAC Address     Interface       Flags
192.168.10.10   00:00:38  5254.0019.3ef6  Vlan10                   <--- We have ARP 
Leaf1# show forwarding route 192.168.20.20 vrf TENANT-A

slot  1
=======


IPv4 routes for table TENANT-A/base

------------------+-----------------------------------------+----------------------+-----------------+-----------------
Prefix            | Next-hop                                | Interface            | Labels          | Partial Install 
------------------+-----------------------------------------+----------------------+-----------------+-----------------
192.168.20.20/32     10.1.1.1           nve1        vni:   50000     


Everything looks like it should be working but once again its late and I need to go to sleep. Will continue this another time.