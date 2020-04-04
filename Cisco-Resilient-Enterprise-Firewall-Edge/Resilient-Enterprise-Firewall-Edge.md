# Cisco / Resilient Enterprise Firewall Edge 

## Overview

In this recipe, we enable a resilent enterprise edge using a firewall (Cisco ASA) and two network providers. With SLA monitor we can track the primary provider and failover (approximately 5 to 7 seconds using configurations provided) to the secondary provider without human intervention. 

## Topology

![alt text]({{ site.baseurl }}Resilient-Enterprise-Firewall-Edge.PNG "Enterprise Firewall Edge")

**Debugging the events in the failure scenario**

```
ce-fw1/act# debug ip routing default 
IP routing debugging is on
 for the default routing table
ce-fw1/act# debug sla monitor trace
IP SLA Monitor TRACE debugging for all operations is on
ce-fw1/act# debug track
ce-fw1/act# IP SLA Monitor(1) Scheduler: Starting an operation
IP SLA Monitor(1) echo operation: Sending an echo operation
IP SLA Monitor(1) echo operation: RTT=26 OK
IP SLA Monitor(1) echo operation: RTT=40 OK
IP SLA Monitor(1) echo operation: RTT=50 OK
IP SLA Monitor(1) Scheduler: Updating result
IP SLA Monitor(1) Scheduler: Starting an operation
IP SLA Monitor(1) echo operation: Sending an echo operation
IP SLA Monitor(1) echo operation: Timeout                                    <<<< SLA monitor probe failed
IP SLA Monitor(1) echo operation: Timeout
IP SLA Monitor(1) echo operation: Timeout
IP SLA Monitor(1) Scheduler: Updating result
Track: 1 Change #14 rtr 1, reachability Up->Down                             <<<< Track transitions to failed
RT: ip_route_delete 0.0.0.0 0.0.0.0 via 192.168.255.3, outside1              <<<< Primary route removed
 ha_cluster_synced 0  routetype 0
RT: del 0.0.0.0 via 192.168.255.3, static metric [1/0]
RT: delete network route to 0.0.0.0 0.0.0.0
RT: default path has been cleared
RT: default path is now 0.0.0.0 via 192.168.255.131NP-route: Delete-Output 0.0.0.0/0 hop_count:1 ,  via 0.0.0.0, outside1
NP-route: Delete-Input 0.0.0.0/0 hop_count:1 Distance:1 Flags:0X0 ,  via 0.0.0.0, outside1
NP-route: Add-Output 0.0.0.0/0 hop_count:1 ,  via 192.168.255.131, outside2  <<<< Add new route
NP-route: Add-Input 0.0.0.0/0 hop_count:1 Distance:254 Flags:0X0 ,  via 192.168.255.131, outside2
RT: IP SLA Monitor(1) Scheduler: Starting an operation
IP SLA Monitor(1) echo operation: Sending an echo operation
```

### Firewall Configuration (ce-fw1)

```
ASA Version 9.8(2) 
!
hostname ce-fw1
!
interface GigabitEthernet0/0
 nameif inside
 security-level 0
 ip address 10.1.0.1 255.255.255.0 standby 10.1.0.2 
!
interface GigabitEthernet0/1
 no nameif
 no security-level
 no ip address
!
interface GigabitEthernet0/1.10
 vlan 10
 nameif outside1
 security-level 100
 ip address 192.168.255.1 255.255.255.128 standby 192.168.255.2 
!
interface GigabitEthernet0/1.20
 vlan 20
 nameif outside2
 security-level 100
 ip address 192.168.255.129 255.255.255.128 standby 192.168.255.130 
!
interface GigabitEthernet0/2
 description LAN/STATE Failover Interface
!
interface Management0/0
 shutdown
 no nameif
 no security-level
 ip address 10.255.0.132 255.255.0.0 
!
! We need two objects to define NAT statements later
!
object network inside1
 subnet 10.1.0.0 255.255.255.0
object network inside2
 subnet 10.1.0.0 255.255.255.0
access-list inside-in extended permit ip any any 
pager lines 23
mtu inside 1500
mtu outside1 1500
mtu outside2 1500
failover
failover lan unit secondary
failover lan interface failover GigabitEthernet0/2
failover replication http
failover link failover GigabitEthernet0/2
failover interface ip failover 10.1.255.1 255.255.255.0 standby 10.1.255.2
!
! Assuming NAT. If so, then we need two NAT statements for each interface that may
! be used as the default route.
!
object network inside1
 nat (inside,outside1) dynamic interface
object network inside2
 nat (inside,outside2) dynamic interface
access-group inside-in in interface inside
!
! 1st route, primary, is tracked and if failed removed as default. Then lower metric
! default is installed into the routing table. 
!
route outside1 0.0.0.0 0.0.0.0 192.168.255.3 1 track 1
route outside2 0.0.0.0 0.0.0.0 192.168.255.131 254
!
! Monitor first hop primary provider interface every 10 seconds with 3 probe packets
!
sla monitor 1
 type echo protocol ipIcmpEcho 192.168.255.3 interface outside1
 num-packets 3
 frequency 10
sla monitor schedule 1 life forever start-time now
!
track 1 rtr 1 reachability
!
policy-map global_policy
 class inspection_default
  inspect ip-options 
  inspect netbios 
  inspect rtsp 
  inspect sunrpc 
  inspect tftp 
  inspect xdmcp 
  inspect dns preset_dns_map 
  inspect ftp 
  inspect h323 h225 
  inspect h323 ras 
  inspect rsh 
  inspect esmtp 
  inspect sqlnet 
  inspect sip  
  inspect skinny  
  inspect icmp 
```
