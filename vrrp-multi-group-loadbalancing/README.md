### Labs built using:
* [PingMyNetwork Sandbox](https://pingmynetwork.com/profile/doer)
---
# VRRP Multi-Group Load Balancing with Tracking
![VRRP01](../imgs/VRRP01.png)
#### 1). Topology Overview
- R1, CORE1, and CORE2 run OSPF, CORE1 and CORE2 SVIs are advertised into OSPF
- CORE1 and CORE2 are connected to each other by an LACP port-channel (F0/2 and F0/3) running as a Layer 3 link on `10.1.0.0/30`
- ASW1 and ASW2 are access switches, and each is connected to both CORE1 and CORE2 using EtherChannel `Trunk` links.
- Hosts are placed in four VLANs shown above
- The default gateway of each PC is the VRRP virtual IP of its VLAN (for example `10.1.10.1` for VLAN 10).
- CORE1 and CORE2 hold the SVIs shown in IPv4 Addressing Label
---
#### 2). Configuration of VRRP and STP
- Make the VRRP Master and the STP root bridge the same switch for each VLAN, so Layer 2 traffic and Layer 3 traffic follow the same path and no traffic has to cross the inter-core link to reach its gateway. Both cores are also actively used, so one is not sitting idle as a backup.

| VRRP Group | VLANs  | CORE1                 | CORE2                 |
| ---------- | ------ | --------------------- | --------------------- |
| 12         | 10, 20 | Priority 101 (Master) | Priority 100 (Backup) |
| 34         | 30, 40 | Priority 100 (Backup) | Priority 101 (Master) |
* CORE1:
```
spanning-tree vlan 10,20 priority 0
interface Vlan10
 vrrp 12 ip 10.1.10.1
 vrrp 12 priority 101
interface Vlan20
 vrrp 12 ip 10.1.20.1
 vrrp 12 priority 101
interface Vlan30
 vrrp 34 ip 10.1.30.1
interface Vlan40
 vrrp 34 ip 10.1.40.1
```
* CORE2:
```
spanning-tree vlan 30,40 priority 0
interface Vlan10
 vrrp 12 ip 10.1.10.1
interface Vlan20
 vrrp 12 ip 10.1.20.1
interface Vlan30
 vrrp 34 ip 10.1.30.1
 vrrp 34 priority 101
interface Vlan40
 vrrp 34 ip 10.1.40.1
 vrrp 34 priority 101
```
---
#### 3). Configuring Tracking
* CORE1:
```
ip sla 1
 icmp-echo 10.1.11.1 source-interface FastEthernet0/1
  frequency 5
ip sla schedule 1 life forever start-time now
track 1 ip sla 1 reachability
interface range vlan10,vlan20
 vrrp 12 track 1 decrement 2
```
* CORE2:
```
ip sla 1
 icmp-echo 10.1.12.1 source-interface FastEthernet0/1
  frequency 5
ip sla schedule 1 life forever start-time now
track 1 ip sla 1 reachability
interface range vlan30,vlan40
 vrrp 34 track 1 decrement 2
```
* An IP SLA ICMP probe from each CORE switch to R1 is tracked, and when the link to R1 fails, the VRRP priority of the Master CORE drops by `2`, which makes it `99` (less than the default of `100` on the peer), so the other CORE takes over as the gateway.
---
#### 4). Analyzing Failures
* NOTE: VLAN10, VLAN20 is used as an example here
* 1). When CORE1 goes down
* ![VRRP02](../imgs/VRRP02.png)
* ![VRRP04](../imgs/VRRP04.png)
* 2). When any link from CORE1 -> R1's g0/0 is down
* ![VRRP03](../imgs/VRRP03.png)
* ![VRRP05](../imgs/VRRP05.png)