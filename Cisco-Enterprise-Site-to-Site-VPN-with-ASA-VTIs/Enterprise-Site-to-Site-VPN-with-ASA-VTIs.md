# Cisco / Enterprise Site to Site VPN with ASA VTIs

**Table of contents**
* [Overview](#overview)
* [Topology](#topology)

## Overview

In this recipe, we enable a resilent enterprise site to site VPN solution using Cisco ASA's VTIs (virtual tunnel interface) and BGP. The resilency goal is to enable automatic failover when an external path fails. We rely on BGP's ability to identify a down peer and then withdraw routes.

## Topology

![alt text]({{ site.baseurl }}/docs/Cisco/Enterprise-Site-to-Site-VPN-with-ASA-VTIs.PNG "Lab Topology")

**Debugging the events in the failure scenario**

Path via `sw1(10.10.1.2) > fw1 > rtr1 > fw2 > sw2(10.10.2.2)` failed by adding a deny IP any ACL to rtr1 interface. BGP peer failed and then the connectivity recovered to the secondary path via `sw1(10.10.1.2) > fw1 > rtr2 > fw2 > sw2(10.10.2.2)`. The convergence is slow because there is no BFD support on this version of ASA, but satisfies our goal of automatic recovery.

```
sw1#ping 10.10.2.2 repeat 100000 
Type escape sequence to abort.
Sending 100000, 100-byte ICMP Echos to 10.10.2.2, timeout is 2 seconds:
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!.........U.U.U.U.U.U.U.U.!!!!!!!!!! # path failure and recovery
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!
Success rate is 91 percent (265/290), round-trip min/avg/max = 4/16/59 ms
```
**BGP debug events on fw1**

```
fw1# 
BGP: nbr_topo global 172.16.1.2 IPv4 Unicast:base (0x00007fc7f26092d0:1) Not scheduling for GR processing [Peer did not advertise GR cap]
BGP: tbl IPv4 Unicast:base Service reset requests
BGP: TX IPv4 Unicast Net global 10.10.2.0/24 Changed.
BGP: TX IPv4 Unicast Mem global 1 1 172.16.1.2 Resetting counters.
BGP: TX IPv4 Unicast Mem global 1 1 172.16.1.2 Ignoring dummy policy change.
BGP: TX IPv4 Unicast Mem global 1 1 172.16.1.2 Resetting counters.
BGP: TX IPv4 Unicast Mem global 1 1 172.16.1.2 Ignoring dummy policy change.
BGP: TX IPv4 Unicast Mem global 1 1 172.16.1.2 Changing state from ACTIVE to DOWN (session not established). # peer marked down
BGP: TX IPv4 Unicast Mem global 1 1 172.16.1.2 Removing from group (1 members left).
BGP: 172.16.1.2 reset due to Peer closed the session
BGP: TX IPv4 Unicast Mem global 172.16.1.2 State is DOWN (session not established).
BGP(0): Revise route installing 1 of 1 routes for 10.10.2.0 255.255.255.0 -> 172.16.1.18(global) to main IP table # installing new route
BGP: TX IPv4 Unicast Net global 10.10.2.0/24 RIB done.
BGP: TX IPv4 Unicast Tab RIB walk done version 12, added 1 topologies.
BGP: TX IPv4 Unicast Tab Executing.
BGP: TX IPv4 Unicast Wkr global 1 Cur Processing.
BGP: TX IPv4 Unicast Top global Appending nets from attr 0x00007fc7f24032e8.
BGP: TX IPv4 Unicast Wkr global 1 Cur Attr change from 0x0000000000000000 to 0x00007fc7f24032e8.
BGP: TX IPv4 Unicast Wkr global 1 Cur Net 10.10.2.0/24 Skipped.
BGP: TX IPv4 Unicast Top global No attributes with modified nets.
BGP: TX IPv4 Unicast Top global Added tail marker with version 12.
BGP: TX IPv4 Unicast Wkr global 1 Cur Reached marker with version 12.
BGP: TX IPv4 Unicast Top global No attributes with modified nets.
BGP: TX IPv4 Unicast Wkr global 1 Cur Done (end of list), processed 1 attr(s), 0/1 net(s), 0 pos.
BGP: TX IPv4 Unicast Grp global 1 Converged.
BGP: TX IPv4 Unicast Tab Processed 1 walker(s).
BGP: TX IPv4 Unicast Tab Generation completed.
BGP: TX IPv4 Unicast Top global Deleting first marker with version 11.
BGP: TX IPv4 Unicast Top global Collection reached marker 11 after 0 net(s).
BGP: TX IPv4 Unicast Top global Collection done on marker 12 after 1 net(s).
BGP: TX IPv4 Unicast Top global Collection done on marker 12 after 0 net(s).
```
**fw1 Configuration**

```
fw1# show run
: Saved

ASA Version 9.8(2) 
!
hostname fw1
!
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 192.168.1.2 255.255.255.0 
!
interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 10.10.1.1 255.255.255.0 
!
interface GigabitEthernet0/2
 nameif outside2
 security-level 0
 ip address 192.168.10.2 255.255.255.0 
!
interface Management0/0
 shutdown
 no nameif
 no security-level
 ip address 10.255.0.242 255.255.0.0 
!
interface Tunnel12
 nameif vti12
 ip address 172.16.1.1 255.255.255.252 
 tunnel source interface outside
 tunnel destination 192.168.2.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROFILE1
!
interface Tunnel13
 nameif vti13
 ip address 172.16.1.5 255.255.255.252 
 tunnel source interface outside
 tunnel destination 192.168.3.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROFILE1
!
interface Tunnel100
 nameif vti100
 ip address 172.16.1.17 255.255.255.252 
 tunnel source interface outside2
 tunnel destination 192.168.11.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROFILE1
!
nat (inside,outside) source dynamic any interface
access-group outside-in in interface outside
access-group inside-in in interface inside
router bgp 65000
 bgp log-neighbor-changes
 address-family ipv4 unicast
  neighbor 172.16.1.2 remote-as 65000
  neighbor 172.16.1.2 activate
  neighbor 172.16.1.2 next-hop-self
  neighbor 172.16.1.6 remote-as 65000
  neighbor 172.16.1.6 activate
  neighbor 172.16.1.6 next-hop-self
  neighbor 172.16.1.18 remote-as 65000
  neighbor 172.16.1.18 activate
  neighbor 172.16.1.18 next-hop-self
  network 10.10.1.0 mask 255.255.255.0
  no auto-summary
  no synchronization
 exit-address-family
!
router ospf 10
 network 10.10.1.0 255.255.255.255 area 0
 log-adj-changes
!
route outside 0.0.0.0 0.0.0.0 192.168.1.1 1
route outside2 192.168.11.2 255.255.255.255 192.168.10.1 1
crypto ipsec ikev1 transform-set SET1 esp-aes esp-sha-hmac 
crypto ipsec profile PROFILE1
 set ikev1 transform-set SET1
 set pfs group5
 responder-only
crypto ikev1 enable outside
crypto ikev1 enable outside2
crypto ikev1 policy 10
 authentication pre-share
 encryption aes-256
 hash sha
 group 5
 lifetime 86400
tunnel-group 192.168.1.2 type ipsec-l2l
tunnel-group 192.168.1.2 ipsec-attributes
 ikev1 pre-shared-key \*\*\*\*\*
!
fw1#  
```

**fw2 Configuration**

```
fw2# show running-config 
: Saved

ASA Version 9.8(2) 
!
hostname fw2

!
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 192.168.2.2 255.255.255.0 
!
interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 10.10.2.1 255.255.255.0 
!
interface GigabitEthernet0/2
 nameif outside2
 security-level 0
 ip address 192.168.11.2 255.255.255.0 
!
interface Management0/0
 shutdown
 no nameif
 no security-level
 ip address 10.255.0.243 255.255.0.0 
!
interface Tunnel12
 nameif vti12
 ip address 172.16.1.2 255.255.255.252 
 tunnel source interface outside
 tunnel destination 192.168.1.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROFILE1
!
interface Tunnel23
 nameif vti23
 ip address 172.16.1.9 255.255.255.252 
 tunnel source interface outside
 tunnel destination 192.168.3.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROFILE1
!
interface Tunnel100
 nameif vti100
 ip address 172.16.1.18 255.255.255.252 
 tunnel source interface outside2
 tunnel destination 192.168.10.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROFILE1
!
object network internal
 subnet 10.10.2.0 255.255.255.0
access-list inside-in extended permit ip any any 
access-list outside-in extended permit ip any any 
nat (inside,outside) source dynamic any interface
access-group outside-in in interface outside
access-group inside-in in interface inside
router bgp 65000
 bgp log-neighbor-changes
 address-family ipv4 unicast
  neighbor 172.16.1.1 remote-as 65000
  neighbor 172.16.1.1 activate
  neighbor 172.16.1.1 next-hop-self
  neighbor 172.16.1.10 remote-as 65000
  neighbor 172.16.1.10 activate
  neighbor 172.16.1.10 next-hop-self
  neighbor 172.16.1.17 remote-as 65000
  neighbor 172.16.1.17 activate
  neighbor 172.16.1.17 next-hop-self
  network 10.10.2.0 mask 255.255.255.0
  no auto-summary
  no synchronization
 exit-address-family
!
route outside 0.0.0.0 0.0.0.0 192.168.2.1 1
route outside2 192.168.10.2 255.255.255.255 192.168.11.1 1
crypto ipsec ikev1 transform-set SET1 esp-aes esp-sha-hmac 
crypto ipsec profile PROFILE1
 set ikev1 transform-set SET1
 set pfs group5
crypto ikev1 enable outside
crypto ikev1 enable outside2
crypto ikev1 policy 10
 authentication pre-share
 encryption aes-256
 hash sha
 group 5
 lifetime 86400
tunnel-group 192.168.1.1 type ipsec-l2l
tunnel-group 192.168.1.1 ipsec-attributes
 ikev1 pre-shared-key \*\*\*\*\*
```
