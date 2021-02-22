# 05 Underlay IS-IS PiM-SM cl vxlan 10.0.0.0/24

На устройствах R1, NX-SPINE и NX-LEAF включен PiM-SM

RP - Loopback-интерфейс на R1 10.120.0.254

IGP - протокол - ISIS взят из старой работы.

Cхема лабораторного стенда в Eve-NG:

![](PIM-SM-VxLAN.png)


Типовые конфигурации:

<details>
  <summary>Конфигурация SPINE</summary>
<pre><code>
# Features

feature pim
feature isis

# PiM-SM

ip pim rp-address 10.120.0.254 group-list 224.0.0.0/4
ip pim ssm range 232.0.0.0/8

#Loopback-интерфейс

interface loopback0
  ip address 10.120.1.1/32
  ip pim sparse-mode

#Интерфейс к R1

interface Ethernet1/1
  no switchport
  ip address 10.120.0.2/30
  isis network point-to-point
  isis circuit-type level-2
  ip router isis 1
  ip pim sparse-mode
  no shutdown

#Интерфейс к LEAF

interface Ethernet1/2
  no switchport
  mtu 9216
  medium p2p
  ip unnumbered loopback0
  no isis hello-padding always
  isis network point-to-point
  isis circuit-type level-1
  ip router isis 1
  ip pim sparse-mode
  no shutdown

#Настройка ISIS  

router isis 1
  net 49.0001.0101.2000.1001.00
  metric-style transition
  address-family ipv4 unicast
    distribute level-1 into level-2 all
    summary-address 10.120.1.0/24 level-2
    summary-address 10.120.128.0/17 level-2
    router-id loopback0
    advertise interface loopback0 level-1
</code></pre>
</details>


<details>
  <summary>Конфигурация LEAF</summary>
<pre><code>
# Features

feature pim
feature isis
feature vn-segment-vlan-based
feature nv overlay

# PiM-SM

ip pim rp-address 10.120.0.254 group-list 224.0.0.0/4
ip pim ssm range 232.0.0.0/8

#VLAN к клиенту

vlan 1,10
vlan 10
  vn-segment 12010
  
#VxLAN

interface nve1
  no shutdown
  source-interface loopback0
  member vni 12010 mcast-group 239.0.0.10


#Loopback-интерфейс

interface loopback0
  ip address 10.120.1.4/32
  ip pim sparse-mode

#Интерфейс к клиентам

interface Ethernet1/3
  switchport access vlan 10

#Интерфейс к SPINE

interface Ethernet1/2
  no switchport
  mtu 9216
  medium p2p
  ip unnumbered loopback0
  no isis hello-padding always
  ip router isis 1
  ip pim sparse-mode
  no shutdown

#Настройка ISIS

router isis 1
  net 49.0001.0101.2000.1004.00
  is-type level-1
  metric-style transition
  address-family ipv4 unicast
	router-id loopback0
    advertise interface loopback0 level-1
</code></pre>
</details>

Пинг между клиентами:

CL-1#ping 10.0.0.4
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.4, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 112/257/497 ms
CL-1#

Мультикаст маршруты RP:

(*, 239.0.0.10), 1w1d/00:03:17, RP 10.120.0.254, flags: S
  Incoming interface: Null, RPF nbr 0.0.0.0
  Outgoing interface list:
    Ethernet0/1, Forward/Sparse, 1w0d/00:02:43
    Ethernet0/2, Forward/Sparse, 1w1d/00:02:37
    Ethernet0/0, Forward/Sparse, 1w1d/00:03:17

(10.120.1.5, 239.0.0.10), 3d02h/00:03:00, flags: T
  Incoming interface: Ethernet0/2, RPF nbr 10.120.0.6
  Outgoing interface list:
    Ethernet0/1, Forward/Sparse, 3d02h/00:02:43

(10.120.1.4, 239.0.0.10), 3d02h/00:02:46, flags: T
  Incoming interface: Ethernet0/2, RPF nbr 10.120.0.6
  Outgoing interface list:
    Ethernet0/0, Forward/Sparse, 00:02:14/00:03:17
    Ethernet0/1, Forward/Sparse, 3d02h/00:02:43

(10.120.2.2, 239.0.0.10), 3d02h/00:02:39, flags: T
  Incoming interface: Ethernet0/1, RPF nbr 10.120.0.10
  Outgoing interface list:
    Ethernet0/0, Forward/Sparse, 3d02h/00:03:17
    Ethernet0/2, Forward/Sparse, 3d02h/00:02:37

(10.120.1.3, 239.0.0.10), 1w1d/00:02:32, flags: T
  Incoming interface: Ethernet0/2, RPF nbr 10.120.0.6
  Outgoing interface list:
    Ethernet0/1, Forward/Sparse, 1w0d/00:02:43

(*, 224.0.1.40), 1w1d/00:02:12, RP 10.120.0.254, flags: SJCL
  Incoming interface: Null, RPF nbr 0.0.0.0
  Outgoing interface list:
    Loopback0, Forward/Sparse, 1w1d/00:02:12

Мультикаст маршруты NX-LEAF-4

NX-LEAF-4# show ip mroute
IP Multicast Routing Table for VRF "default"

(*, 232.0.0.0/8), uptime: 1w0d, pim ip
  Incoming interface: Null, RPF nbr: 0.0.0.0
  Outgoing interface list: (count: 0)


(*, 239.0.0.10/32), uptime: 1w0d, nve ip pim
  Incoming interface: Ethernet1/2, RPF nbr: 10.120.2.1
  Outgoing interface list: (count: 1)
    nve1, uptime: 1w0d, nve


(10.120.1.3/32, 239.0.0.10/32), uptime: 00:01:08, ip pim mrib
  Incoming interface: Ethernet1/2, RPF nbr: 10.120.2.1
  Outgoing interface list: (count: 1)
    nve1, uptime: 00:01:08, mrib


(10.120.1.5/32, 239.0.0.10/32), uptime: 00:01:58, ip pim mrib
  Incoming interface: Ethernet1/2, RPF nbr: 10.120.2.1
  Outgoing interface list: (count: 1)
    nve1, uptime: 00:01:58, mrib


(10.120.2.2/32, 239.0.0.10/32), uptime: 1w0d, nve mrib ip pim
  Incoming interface: loopback0, RPF nbr: 10.120.2.2
  Outgoing interface list: (count: 1)
    Ethernet1/2, uptime: 3d02h, pim

Мультикаст маршруты NX-LEAF-3

NX-LEAF-3# show ip mroute
IP Multicast Routing Table for VRF "default"

(*, 232.0.0.0/8), uptime: 1w1d, pim ip
  Incoming interface: Null, RPF nbr: 0.0.0.0
  Outgoing interface list: (count: 0)


(*, 239.0.0.10/32), uptime: 1w1d, nve pim ip
  Incoming interface: Ethernet1/2, RPF nbr: 10.120.1.2
  Outgoing interface list: (count: 1)
    nve1, uptime: 1w1d, nve


(10.120.1.3/32, 239.0.0.10/32), uptime: 1w1d, pim mrib ip
  Incoming interface: Ethernet1/3, RPF nbr: 10.120.1.1
  Outgoing interface list: (count: 2)
    Ethernet1/2, uptime: 1w1d, pim
    nve1, uptime: 1w1d, mrib


(10.120.1.5/32, 239.0.0.10/32), uptime: 1w1d, nve mrib pim ip
  Incoming interface: loopback0, RPF nbr: 10.120.1.5
  Outgoing interface list: (count: 2)
    Ethernet1/3, uptime: 03:55:04, pim
    Ethernet1/2, uptime: 3d02h, pim


(10.120.2.2/32, 239.0.0.10/32), uptime: 00:00:08, ip pim mrib
  Incoming interface: Ethernet1/2, RPF nbr: 10.120.1.2
  Outgoing interface list: (count: 1)
    nve1, uptime: 00:00:08, mrib


