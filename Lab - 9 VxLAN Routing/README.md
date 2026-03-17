# VxLAN. Оптимизация таблиц маршрутизации
## Цели занятия
- разобрать __EVPN route-type 5__ и его применение;
- настроить __route-type__ для оптимизации маршрутизации.

## Краткое содержание
- разберем передачу маршрутной инфомарции EVPN type-5 и настройку EVPN для передачи маршрутной информации через type-5 анонсы.
ho ip route

VRF: default
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 C        5.5.5.4/30 is directly connected, Ethernet8
 B E      8.8.8.0/24 [200/0] via 5.5.5.6, Ethernet8
 B E      10.0.0.4/32 [200/0] via 5.5.5.6, Ethernet8
 C        10.0.3.0/31 is directly connected, Ethernet1
 C        10.0.3.4/31 is directly connected, Ethernet2
 B E      10.1.0.1/32 [200/0] via 10.0.3.1, Ethernet1
                              via 10.0.3.5, Ethernet2
 B E      10.1.0.2/32 [200/0] via 10.0.3.1, Ethernet1
                              via 10.0.3.5, Ethernet2
 C        10.1.0.3/32 is directly connected, Loopback1
 B E      10.1.1.1/32 [200/0] via 10.0.3.1, Ethernet1
 B E      10.2.2.2/32 [200/0] via 10.0.3.5, Ethernet2

leaf3#
gp evpn summary | grep 5.5.5.6
  5.5.5.6  4 65504          35942     39671    0   19 00:01:21 Estab   0      0
leaf3#
 ip route vrf vrf2

VRF: vrf2
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set


leaf3#
gp evpn route-type ip-prefix ipv4
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65503
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 65501:1 ip-prefix 192.168.112.0/24
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 65501:1 ip-prefix 192.168.112.0/24
                                 10.1.0.1              -       100     0       65500 65501 i
 * >Ec    RD: 65502:1 ip-prefix 192.168.112.0/24
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 65502:1 ip-prefix 192.168.112.0/24
                                 10.1.0.2              -       100     0       65500 65502 i
 * >      RD: 65503:1 ip-prefix 192.168.113.0/24
                                 -                     -       -       0       i

<details>
  <summary>leaf3 конфигурация</summary>
       
```
leaf3#sho run


! Command: show running-config




! device: leaf3 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname leaf3
!
spanning-tree mode mstp
!
vlan 3
   name 3
!
vlan 113
   name 113
!
vrf instance v
!
vrf instance vrf1
!
vrf instance vrf2
   description VRF ROute-6 IP Advertise Option A BGP to Edgge Router
!
interface Ethernet1
   description Peer-to-peer link to Spine-1
   no switchport
   ip address 10.0.3.0/31
!
interface Ethernet2
   description Peer-to-peer link to Spine-2
   no switchport
   ip address 10.0.3.4/31
!
interface Ethernet3
   description downlink to host
   switchport access vlan 113
!
interface Ethernet4
   switchport access vlan 112
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
   description to EDGE router
   no switchport
   ip address 5.5.5.5/30
!
interface Loopback1
   description VTEP
   ip address 10.1.0.3/32
!
interface Management1
!
interface Vlan101
   vrf vrf2
   ip address 10.10.10.2/29
!
interface Vlan113
   description Host Network 113
   vrf vrf1
   ip address 192.168.113.1/24
   ip virtual-router address 192.168.113.254/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 101 vni 101
   vxlan vlan 112 vni 1112
   vxlan vrf vrf1 vni 666
   vxlan learn-restrict any
!
ip virtual-router mac-address 02:00:00:00:00:00
!
ip routing
no ip routing vrf v
ip routing vrf vrf1
ip routing vrf vrf2
!
router bgp 65503
   router-id 10.1.0.3
   no bgp default ipv4-unicast
   timers bgp 3 9
   rd auto
   maximum-paths 3 ecmp 3
   neighbor SPINE-EVPN peer group
   neighbor SPINE-EVPN remote-as 65500
   neighbor SPINE-EVPN update-source Loopback1
   neighbor SPINE-EVPN ebgp-multihop 3
   neighbor SPINE-EVPN send-community standard extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65500
   neighbor UNDERLAY out-delay 0
   neighbor UNDERLAY send-community extended
   neighbor test peer group
   neighbor test remote-as 65504
   neighbor test update-source Ethernet8
   neighbor 5.5.5.6 peer group test
   neighbor 10.0.3.1 peer group UNDERLAY
   neighbor 10.0.3.5 peer group UNDERLAY
   neighbor 10.1.1.1 peer group SPINE-EVPN
   neighbor 10.2.2.2 peer group SPINE-EVPN
   !
   vlan 101
      rd auto
      route-target both 65503:101
      redistribute learned
   !
   vlan 113
      rd auto
      route-target both 65500:1113
      redistribute learned
   !
   address-family evpn
      neighbor SPINE-EVPN activate
      neighbor test activate
      neighbor 5.5.5.6 activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      neighbor test activate
      network 10.1.0.3/32
      redistribute connected
   !
   vrf vrf1
      rd 65503:1
      route-target import evpn 1:666
      route-target export evpn 1:666
      redistribute connected
   !
   vrf vrf2
      rd 65504:1
      route-target import evpn 2:666
      route-target import evpn 65504:1
      route-target export evpn 2:666
      route-target export evpn 65504:1
      neighbor 5.5.5.6 remote-as 65504
      redistribute connected
      !
      address-family ipv4
         neighbor 5.5.5.6 activate
!
end
```
</details>
 
  

Маршрут до 8.8.8.0/24 уже есть на leaf3 через BGP от EdgeRouter. Нужно "за leaking" этот маршрут в EVPN route type 5 на leaf3, чтобы другие листья (leaf1, leaf2) увидели его.

На leaf3 нужно сделать следующее:

Создать VRF (у вас уже есть vrf2, но судя по выводу, маршрут 8.8.8.0/24 находится в глобальной таблице)

Настроить импорт маршрута из глобальной таблицы в VRF

Настроить экспорт этого маршрута в EVPN route type 5

Вот конфигурация для leaf3:




   
