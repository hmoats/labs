# Big Switch / Simple 3 tenant design lab

**Table of contents**
* [Overview](#overview)
* [Topology](#topology)
* [Configuration](#configuration)
* [Debugging](#debugging)
  * [Route tables](#route-tables)

---

## Overview

In this lab we build a simple design with the following hierarchy

- Tenant 1 (red)
  - Purpose: service tenant
  - Segments: app, db
  - Security posture: secure DB network and expose only Internet services in app segment
- Tenant 2 (external)
  - Purpose: external edge ingress/egress 
  - Segments: dmz
  - Security posture: edge filtering
- Tenant 3 (mgmt)
  - Purpose: management, monitoring and bastion hosts
  - Segments: mgmt
  - Security posture: allow 192.168.1.1 to bastion hosts. Allow bastion host to red tenant via TCP 80, 443 and 22

## Topology

![Topology](/Simple-3-tenant-design-lab.PNG)

## Configuration

```
! # The configuration assumes the base spine and leaf configurations are complete. 
! #
! # First we need to create interface groups.
! 
! interface-group
interface-group H0
  description 10.0.0.2
  member switch R1L1 interface R1L1-eth5
  member switch R1L2 interface R1L2-eth5

interface-group H1
  description 10.0.1.2
  member switch R2L1 interface R2L1-eth5
  member switch R2L2 interface R2L2-eth5

interface-group H2
  description 10.0.2.2
  member switch R2L1 interface R2L1-eth6
  member switch R2L2 interface R2L2-eth6

interface-group R1
  description 10.0.3.2
  member switch R3L1 interface R3L1-eth6
  member switch R3L2 interface R3L2-eth6

! tenant
tenant external
  !
  logical-router
    apply policy-list acl-external
    !
    ! # The tenant router needs a default route to the physical router
    route 0.0.0.0/0 next-hop next-hop-group default-route-external
    interface segment dmz
      ip address 10.0.3.1/24
    !
    interface tenant system
      import-route
    !
    next-hop-group default-route-external
      ip 10.0.3.2
    policy-list acl-external
      !
      ! # Allow external access to the app segment 
      10 permit proto tcp any to tenant red segment app port 80
      20 permit proto tcp any to tenant red segment app port 443
      !
      ! # We need to allow the return traffic from the app network. At the time
      ! # of this lab there is no way to filter on a range of ports of TCP flags.
      30 permit tenant red segment app to any
      999 deny any to any
  segment dmz
    member interface-group R1 vlan untagged

tenant mgmt
  !
  logical-router
    apply policy-list acl-mgmt
    route 0.0.0.0/0 next-hop tenant system
    interface segment mgmt
      ip address 10.0.2.1/24
    !
    interface tenant system
      import-route
    policy-list acl-mgmt
      !
      ! # Allow ping to 
      10 permit proto icmp any to any
      20 permit tenant red to tenant mgmt
      30 permit tenant external to tenant mgmt
      40 permit proto tcp tenant mgmt to tenant red segment app port 80
      50 permit proto tcp tenant mgmt to tenant red segment app port 443
      60 permit proto tcp tenant mgmt to tenant red segment db port 22
      999 deny any to any
  !
  segment mgmt
    description mgmt
    member interface-group H2 vlan untagged

tenant red
  !
  logical-router
    apply policy-list acl-red
    route 0.0.0.0/0 next-hop tenant system
    interface segment app
      ip address 10.0.0.1/24
    interface segment db
      ip address 10.0.1.1/24
    !
    interface tenant system
      import-route
    policy-list acl-red
      10 permit proto icmp any to any
      20 permit tenant mgmt to tenant red
      30 permit tenant external to tenant red
      40 permit proto tcp any to tenant red segment app port 80
      50 permit proto tcp any to tenant red segment app port 443
      60 permit proto tcp tenant red segment app port 80 to any
      70 permit proto tcp tenant red segment app port 443 to any
      80 permit proto tcp tenant red segment app to tenant red segment db port 3306
      90 permit proto tcp tenant red segment db port 3306 to tenant red segment app
      999 deny any to any
  !
  segment app
    description app
    member interface-group H0 vlan untagged
  !
  segment db
    description db
    member interface-group H1 vlan untagged

tenant system
  !
  logical-router
    apply policy-list acl-system
    route 0.0.0.0/0 next-hop tenant external
    !
    interface tenant external
      export-route
    !
    interface tenant mgmt
      export-route
    !
    interface tenant red
      export-route
    policy-list acl-system
      10 permit proto icmp any to any
      20 permit any to tenant red segment app
      30 permit tenant red segment app to any
      40 permit tenant mgmt to tenant red
      50 permit tenant red to tenant mgmt
      60 permit tenant external to tenant red
      70 permit tenant red to tenant external
      80 permit tenant external to tenant mgmt
      90 permit tenant mgmt to tenant external
      999 deny any to any
```

## Debugging

### Route tables

```
lab1# show tenant red logical-router route
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ Route Table ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# Tenant Name Dest Cidr   Preference Type      Protocol Next Hop Tenant Next Hop            Next Hop IP Status
-|-----------|-----------|----------|---------|--------|---------------|-------------------|-----------|--------|
1 red         0.0.0.0/0   1          static             system          Tenant iface system             Active
2 red         10.0.0.0/24 0          connected          red             Segment Iface app               Active
3 red         10.0.1.0/24 0          connected          red             Segment Iface db                Active
4 red         10.0.2.0/24 0          imported           system                                          Active
5 red         10.0.3.0/24 0          imported           system                                          Active
6 red         10.0.3.2/32 0          imported           system                                          Inactive

lab1# show tenant mgmt logical-router route
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ Route Table ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# Tenant Name Dest Cidr   Preference Type      Protocol Next Hop Tenant Next Hop            Next Hop IP Status
-|-----------|-----------|----------|---------|--------|---------------|-------------------|-----------|--------|
1 mgmt        0.0.0.0/0   1          static             system          Tenant iface system             Active
2 mgmt        10.0.0.0/24 0          imported           system                                          Active
3 mgmt        10.0.1.0/24 0          imported           system                                          Active
4 mgmt        10.0.2.0/24 0          connected          mgmt            Segment Iface mgmt              Active
5 mgmt        10.0.3.0/24 0          imported           system                                          Active
6 mgmt        10.0.3.2/32 0          imported           system                                          Inactive

lab1# show tenant external logical-router route
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ Route Table  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# Tenant Name Dest Cidr   Preference Type      Protocol Next Hop Tenant Next Hop               Next Hop IP Status
-|-----------|-----------|----------|---------|--------|---------------|----------------------|-----------|------|
1 external    0.0.0.0/0   1          static             external        default-route-external 10.0.3.2    Active
2 external    10.0.0.0/24 0          imported           system                                             Active
3 external    10.0.1.0/24 0          imported           system                                             Active
4 external    10.0.2.0/24 0          imported           system                                             Active
5 external    10.0.3.0/24 0          connected          external        Segment Iface dmz                  Active
6 external    10.0.3.2/32 0          host               external        Segment Iface dmz                  Active

lab1# show tenant system logical-router route
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ Route Table  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# Tenant Name Dest Cidr   Preference Type     Protocol Next Hop Tenant Next Hop              Next Hop IP Status
-|-----------|-----------|----------|--------|--------|---------------|---------------------|-----------|------|
1 system      0.0.0.0/0   1          static            external        Tenant iface external             Active
2 system      10.0.0.0/24 0          imported          red                                               Active
3 system      10.0.1.0/24 0          imported          red                                               Active
4 system      10.0.2.0/24 0          imported          mgmt                                              Active
5 system      10.0.3.0/24 0          imported          external                                          Active
6 system      10.0.3.2/32 0          imported          external                                          Active

```

### Test path

```
```
