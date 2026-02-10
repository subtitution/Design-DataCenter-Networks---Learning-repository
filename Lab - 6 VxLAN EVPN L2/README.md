# VxLAN L2 VNI <br>
Цель работы: 
Настроить Overlay на основе VxLAN EVPN для L2 связанности между клиентами.

## План работы
- Вспомнить теорию и основные термины
- Настроить BGP peering между Leaf и Spine в AF (Adress Family) l2vpn evpn
- Настроить связанность между клиентами в первой зоне и убедиться в её наличии

## Список терминов и сокращений:
- __NVO__ - __Network Virtualization Overlay__ - оверлейная сеть
- __NVE__ - Network Virtualization Edge. Туннельный интерфейс инкапсуляции/декапсуляции фреймов
- __VTEP__ - __VxLAN Tunnel End Point__ - устройство, которое занимается инкапсуляцией/декапсуляцией фреймов  (обычно leaf)
- __VNI__ - Virtual Network Identifier - vtnrf МчДФТ инкапсуляции, определяющая Layer 2 домен в оверлей сети
- __EVI__ - EVPN Instance - логический свитч в EVPN домене
- __MAC-VRF__ - Virtual Routing and Forwarding table для MAC адресов

# 1. Теория
## Типы маршрутов EVPN

### Основные операци
- __Type-2 Route__: Host Advertisement Route (Для анонса информации о подключенных хостах)
- __Type-3 Route__: Inclusive Multicast Ethernet TAG Route (IMET) - Для работы с BUM (Ingress Replication)
  
### Multi-homing
- __Type-4 Route__ - Ethernet Segment Route (Для выбора Designed Forwarding (Кто будет управлять BUM трафиком))
- __Type-1 Route__ - Ethernet Auto-Discovery Route (Для объявления Ethernet Segment Identifier (__ESI__) (Для конвергенции и балансировки))

### Связь с внешним миром
- __Type-5 Route__ - IP-prefix route advertisement (Для анонса внешних маршрутов в фабрику)

### Multicast
- __Type-6,7,8__ - Для распространения PIM, IGMP Leave/Join по фабрике
# 2. Практика - Настройка
Как говорится, учиться надо на ошибках, так вот, все нижеследующее до пункта 3, неверно!. Я решил специально решил это оставить, чтобы было на чем учиться))). Так, что кому интересно, как правильно настроить, можно смело сразу переходить к (## Пункт 3).
<img width="1053" height="599" alt="image" src="https://github.com/user-attachments/assets/bc6f1747-3d19-4fed-8168-db814ec7b5b2" />

## 2.1. Настройка leaf1
```
leaf1(config)#service routing protocols model multi-agent
! Change will take effect only after switch reboot
!
service routing protocols model multi-agent
!
interface Loopback0
   description for Underlay network
   ip address 10.0.0.1/32
!
interface Loopback1
   description VTEP
   ip address 10.1.0.1/32
!
interface Vlan1
   ip address 192.168.1.1/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 1 vni 1
!
ip routing
!
router bgp 65501
   router-id 10.0.0.1
   timers bgp 3 9
   maximum-paths 2 ecmp 2
   neighbor SPINE-EVPN peer group
   neighbor SPINE-EVPN remote-as 65500
   neighbor SPINE-EVPN update-source Loopback1
   neighbor SPINE-EVPN ebgp-multihop 3
   neighbor SPINE-EVPN send-community standard extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65500
   neighbor UNDERLAY out-delay 0
   neighbor UNDERLAY update-source Loopback0
   neighbor UNDERLAY send-community extended
   neighbor 10.0.1.1 peer group UNDERLAY
   neighbor 10.0.1.1 remote-as 65500
   neighbor 10.0.1.1 send-community extended
   neighbor 10.0.1.5 peer group UNDERLAY
   neighbor 10.0.1.5 remote-as 65500
   neighbor 10.0.1.5 send-community extended
   neighbor 10.1.1.2 peer group SPINE-EVPN
   neighbor 10.2.2.2 peer group SPINE-EVPN
   !
   vlan 1
      rd auto
      route-target both 1:1
      redistribute learned
   !
   address-family evpn
      neighbor SPINE-EVPN activate
      neighbor 10.0.1.1 activate
      neighbor 10.0.1.5 activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.0.0.1/32
!
end
leaf1#
```
## Настройка Leaf2
```
leaf2#sho run

service routing protocols model ribd

vlan 112
   name Host_network_Vlan_112

interface Loopback0
   description IP for underlay -Router-ID
   ip address 10.0.0.2/32
!
interface Loopback1
   description VTEP
   ip address 2.2.2.2/32
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 112 vni 112
   vxlan learn-restrict any
!
router bgp 65502
   maximum-paths 2 ecmp 2
   neighbor SPINE peer group
   neighbor SPINE send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65500
   neighbor UNDERLAY out-delay 0
   neighbor 10.0.2.1 peer group UNDERLAY
   neighbor 10.0.2.5 peer group UNDERLAY
   neighbor 10.1.1.2 peer group SPINE
   neighbor 10.2.2.2 peer group SPINE
   !
   vlan 112
      rd 2.2.2.2:112
      route-target both 112:112
      redistribute learned
   !
   address-family evpn
      neighbor SPINE activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.0.0.2/32
```
## Настройка Spine 1
```
service routing protocols model multi-agent
!
interface Ethernet1
   description Peer-to-peer link to leaf-1
   no switchport
   ip address 10.0.1.1/31
!
interface Ethernet2
   description Peer-to-peer link to leaf-2
   no switchport
   ip address 10.0.2.1/31
!
interface Ethernet3
   description Peer-to-peer link to leaf-3
   no switchport
   ip address 10.0.3.1/31
!
interface Loopback1
   description IP for underlay -Router-ID
   ip address 10.1.1.1/32
!
interface Loopback2
   description to EVPN peer
   ip address 10.1.1.2/32

peer-filter fleaf-asn
   1 match as-range 65500-65600 result accept
!
router bgp 65500
   router-id 10.1.1.1
   no bgp default ipv4-unicast
   timers bgp 3 9
   bgp listen range 10.0.0.0/16 peer-group UNDERLAY peer-filter fleaf-asn
   neighbor SPINE-EVPN peer group
   neighbor SPINE-EVPN remote-as 65503
   neighbor SPINE-EVPN update-source Loopback2
   neighbor SPINE-EVPN ebgp-multihop 3
   neighbor SPINE-EVPN send-community standard extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY out-delay 0
   neighbor UNDERLAY send-community extended
   neighbor 2.2.2.2 peer group SPINE-EVPN
   neighbor 10.1.0.1 peer group SPINE-EVPN
   !
   address-family evpn
      neighbor SPINE-EVPN activate
      neighbor SPINE-EVPN next-hop-unchanged
      no neighbor 10.0.0.1 activate
      no neighbor 10.0.0.2 activate
      no neighbor 10.0.0.3 activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.1.1.1/32
```
## Диагностические выводы команд
```
spine1#
p bgp1#
BGP routing table information for VRF default
Router identifier 10.1.1.1, local AS number 65500
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.0.0.2/32            10.0.2.0              0       -          100     0       65502 i
 * >      10.0.0.3/32            10.0.3.0              0       -          100     0       65503 i
 * >      10.1.1.1/32            -                     -       -          -       0       i
```
## Проверка BGP EVPN соседей
```
spine1#sho bgp evpn summary
BGP summary information for VRF default
Router identifier 10.1.1.1, local AS number 65500
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  2.2.2.2  4 65503              0         0    0    0 02:42:32 Active
  10.1.0.1 4 65503              0         0    0    0 02:43:30 Active
```
## Проверка EVPN маршрутов
```
spine1#sho bgp evpn
BGP routing table information for VRF default
Router identifier 10.1.1.1, local AS number 65500
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
spine1#sho bgp evpn detail
BGP routing table information for VRF default
Router identifier 10.1.1.1, local AS number 65500
spine1#
```
## Leaf01
```
leaf1#sho interfaces vxlan 1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback1 and is active with 10.1.0.1
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is
    [1, 1]
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is not configured
  Shared Router MAC is 0000.0000.0000
leaf1#
```
## Проверка VTEP
```
leaf1#sho vxlan vtep
Remote VTEPS for Vxlan1:

VTEP       Tunnel Type(s)
---------- --------------

Total number of remote VTEPS:  0
leaf1#
```
## Проверка BGP EVPN маршрутов
```
leaf1#sho bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65501
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.0.1:1 imet 10.1.0.1
                                 -                     -       -       0       i
leaf1#
leaf1#sho bgp evpn route-type i?
imet  ip-prefix

leaf1#sho bgp evpn route-type imet detail
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65501
BGP routing table entry for imet 10.1.0.1, Route Distinguisher: 10.0.0.1:1
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: Route-Target-AS:1:1 TunnelEncap:tunnelTypeVxlan
      VNI: 1
      PMSI Tunnel: Ingress Replication, MPLS Label: 1, Leaf Information Required: false, Tunnel ID: 10.1.0.1
leaf1#
```
# Leaf2
```
leaf2#
[Kw interfaces vxlan 112
Vxlan1 is down, line protocol is down (notconnect)
  Hardware is Vxlan
  Source interface is Loopback0 and is active with 10.0.0.2
  Listening on UDP port 4789
  Replication/Flood Mode is not initialized yet
  Remote MAC learning via Datapath
  VNI mapping to VLANs
  Static VLAN to VNI mapping is
    [112, 112]
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is not configured
  Shared Router MAC is 0000.0000.0000
leaf2#

leaf2#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP       Tunnel Type(s)
---------- --------------

Total number of remote VTEPS:  0
leaf2#


leaf2#sho bgp evpn route-type imet detail
% Not supported
leaf2#
```
# Пункт 3. Еще раз практика, или как завещал товарищь Владимир Ильич - Учиться! Учиться и еще раз Учиться!
В данном разделе я опишу, что и как я поменял в схеме сети:
1. Я избавился от loopback интерфесов на Leaf коммутаторах, на которых ранее пытался сделать UNDERLAY BGP сеть. Вместо loopbcak интерфейсов используются настоящие физические для построения и анонсирования.
2. На leaf коммутаторах loopback адреса используются для построения EVPN BGP.
3. Ниже обновленная схема представлена на рисунке.
