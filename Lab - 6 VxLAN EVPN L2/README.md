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
Как говорится, учиться надо на ошибках, так вот, все нижеследующее до пункта 3, неверно!. Я решил это оставить, чтобы было на чем учиться))). Так, что прошу сразу переходить
к  __Пункт 3 - Настройка!!!__.
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
# Пункт 3 - Настройка
Еще раз практика, или как завещал товарищь Владимир Ильич - Учиться! Учиться и еще раз Учиться!
В данном разделе я опишу, что и как я поменял в схеме сети:
1. Я избавился от loopback интерфесов на Leaf коммутаторах, на которых ранее пытался сделать UNDERLAY BGP сеть. Вместо loopbcak интерфейсов используются настоящие физические для построения и анонсирования.
2. На leaf коммутаторах loopback адреса используются для построения EVPN BGP.
3. Ниже обновленная схема представлена на рисунке.
<img width="1512" height="924" alt="image" src="https://github.com/user-attachments/assets/105c1fa1-4ddc-4a4e-9ca8-c79249c59332" />



## 3.1. Настройки  Spine коммутаторов
### 3.1.1. Spine1
``` arista
!
service routing protocols model multi-agent
!
hostname spine1
!
spanning-tree mode mstp
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
peer-filter fleaf-asn
   1 match as-range 65500-65600 result accept
!
router bgp 65500
   router-id 10.1.1.1
   no bgp default ipv4-unicast
   timers bgp 3 9
   neighbor SPINE-EVPN peer group
   neighbor SPINE-EVPN update-source Loopback1
   neighbor SPINE-EVPN ebgp-multihop 3
   neighbor SPINE-EVPN send-community standard extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY out-delay 0
   neighbor UNDERLAY send-community extended
   neighbor 10.0.1.0 peer group UNDERLAY
   neighbor 10.0.1.0 remote-as 65501
   neighbor 10.0.2.0 peer group UNDERLAY
   neighbor 10.0.2.0 remote-as 65502
   neighbor 10.0.3.0 peer group UNDERLAY
   neighbor 10.0.3.0 remote-as 65503
   neighbor 10.1.0.1 peer group SPINE-EVPN
   neighbor 10.1.0.1 remote-as 65501
   neighbor 10.1.0.2 peer group SPINE-EVPN
   neighbor 10.1.0.2 remote-as 65502
   neighbor 10.1.0.3 peer group SPINE-EVPN
   neighbor 10.1.0.3 remote-as 65503
   !
   address-family evpn
      neighbor SPINE-EVPN activate
      neighbor SPINE-EVPN next-hop-unchanged
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.1.1.1/32
!
```
## 3.1.2 Основные настройки Spine 2
```
spine2#
!
interface Ethernet1
   description Peer-to-peer link to leaf-1
   no switchport
   ip address 10.0.1.5/31
!
interface Ethernet2
   description Peer-to-peer link to leaf-2<br>
   no switchport
   ip address 10.0.2.5/31
!
interface Ethernet3
   description Peer-to-peer link to leaf-3<br>
   no switchport
   ip address 10.0.3.5/31
!
interface Loopback1
   description Overlay loopback
   ip address 10.2.2.2/32
!
peer-filter fleaf-asn
   1 match as-range 65500-65600 result accept
!
router bgp 65500
   router-id 10.2.2.2
   no bgp default ipv4-unicast
   timers bgp 3 9
   neighbor SPINE-EVPN peer group
   neighbor SPINE-EVPN update-source Loopback1
   neighbor SPINE-EVPN ebgp-multihop 3
   neighbor SPINE-EVPN send-community standard extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY out-delay 0
   neighbor UNDERLAY send-community extended
   neighbor 10.0.1.4 peer group UNDERLAY
   neighbor 10.0.1.4 remote-as 65501
   neighbor 10.0.2.4 peer group UNDERLAY
   neighbor 10.0.2.4 remote-as 65502
   neighbor 10.0.3.4 peer group UNDERLAY
   neighbor 10.0.3.4 remote-as 65503
   neighbor 10.1.0.1 peer group SPINE-EVPN
   neighbor 10.1.0.1 remote-as 65501
   neighbor 10.1.0.2 peer group SPINE-EVPN
   neighbor 10.1.0.2 remote-as 65502
   neighbor 10.1.0.3 peer group SPINE-EVPN
   neighbor 10.1.0.3 remote-as 65503
   !
   address-family evpn
      neighbor SPINE-EVPN activate
      neighbor SPINE-EVPN next-hop-unchanged
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.2.2.2/32
```
## 3.2. Настройка Leaf коммутаторов
### 3.2.1. Настройка leaf1
```
leaf1#
service routing protocols model multi-agent
vlan 1
   name Host_Network
!
interface Ethernet1
   description Peer-to-peer link to Spine-1
   no switchport
   ip address 10.0.1.0/31
!
interface Ethernet2
   description Peer-to-peer link to Spine-2
   no switchport
   ip address 10.0.1.4/31
!
interface Ethernet3
   description -=Direction to host=-
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
   vxlan vlan 1 vni 1112
!
router bgp 65501
   router-id 10.1.0.1
   no bgp default ipv4-unicast
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
   neighbor UNDERLAY send-community extended
   neighbor 10.0.1.1 peer group UNDERLAY
   neighbor 10.0.1.5 peer group UNDERLAY
   neighbor 10.1.1.1 peer group SPINE-EVPN
   neighbor 10.2.2.2 peer group SPINE-EVPN
   !
   vlan 1
      rd auto
      route-target both 65500:1112
      redistribute learned
   !
   address-family evpn
      neighbor SPINE-EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.1.0.1/32
```
### 3.2.2. Настройка leaf2
```
service routing protocols model multi-agent
!
vlan 112
   name Host_network_Vlan_112
!
interface Ethernet1
   description Peer-to-peer link to Spine-1
   no switchport
   ip address 10.0.2.0/31
!
interface Ethernet2
   description Peer-to-peer link to Spine-2
   no switchport
   ip address 10.0.2.4/31
!
interface Ethernet3
   description to host 112 VLAN
   switchport access vlan 112
!
interface Loopback1
   description VTEP
   ip address 10.1.0.2/32
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 112 vni 1112
!
router bgp 65502
   router-id 10.1.0.2
   no bgp default ipv4-unicast
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
   neighbor UNDERLAY send-community extended
   neighbor 10.0.2.1 peer group UNDERLAY
   neighbor 10.0.2.5 peer group UNDERLAY
   neighbor 10.1.1.1 peer group SPINE-EVPN
   neighbor 10.2.2.2 peer group SPINE-EVPN
   !
   vlan 112
      rd auto
      route-target both 65500:1112
      redistribute learned
   !
   address-family evpn
      neighbor SPINE-EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.1.0.2/32
!
```
### 3.2.3. Настройка leaf3
```
! device: leaf3 (vEOS-lab, EOS-4.29.2F)
!
service routing protocols model multi-agent
!
vlan 3
   name 3
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
   switchport access vlan 3
!
interface Loopback1
   description VTEP
   ip address 10.1.0.3/32
!
interface Vlan3
   ip address 192.168.3.1/24
!
router bgp 65503
   router-id 10.1.0.3
   no bgp default ipv4-unicast
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
   neighbor UNDERLAY send-community extended
   neighbor 10.0.3.1 peer group UNDERLAY
   neighbor 10.0.3.5 peer group UNDERLAY
   neighbor 10.1.1.1 peer group SPINE-EVPN
   neighbor 10.2.2.2 peer group SPINE-EVPN
   !
   address-family evpn
      neighbor SPINE-EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.1.0.3/32
```
## 4. Процесс установления BGP EVPN сессии
И так дорогие мои мы дошли до самой магии, у нас уже есть установившиеся eBGP сессия, бегают пакетики туда-сюда. И вот Leaf1 становится инициатором в установке EVPN BGP сессии и соответсвенно, отправляет своим соседам (Spine 1 и 2) __BGP Open Message__:
Leaf1, d OPEN сообщении сообщает номер своей AS 65501, BGP Identifier 10.1.0.1, ниже пример:
<img width="922" height="2094" alt="image" src="https://github.com/user-attachments/assets/0d7725af-e963-4a06-b007-10fc2882af6f" />
Естественно аналогичным образом Spine1,2 отвечают с указанием своей AS и BGP Identifier.
После черего происходит обмен keepalive-ами. Далее Leaf1 пробует установить EVPN BGP сессию, и в этот раз  посылает ~~OPEN~~ Update Message, в котором содержится следующая ключевая информация:
в NLRI содержится Route Distinguisher, Path Attribute - Extended Communitties->Route target: 65500:1112, Type tunnel: Vxlan Encapsulation, снизу картинка:
<img width="1044" height="2780" alt="image" src="https://github.com/user-attachments/assets/9823a405-8462-4657-9f1a-849387da255d" />
Через некоторое время leaf 1, получил BGP UPDATE от spine1, в котором указывались атрибуты для построения BGP EVPN туннеля с Leaf2, картинка снизу:
<img width="1040" height="1588" alt="image" src="https://github.com/user-attachments/assets/5236eac9-dfe5-4e96-b0a7-fafcd959c6b2" />
## 5. Проверка
Пингую с PC2-1
```
pc2-1> ping 192.168.1.2

*192.168.112.1 icmp_seq=1 ttl=64 time=11.009 ms (ICMP type:3, code:0, Destination network unreachable)
*192.168.112.1 icmp_seq=2 ttl=64 time=10.219 ms (ICMP type:3, code:0, Destination network unreachable)
*192.168.112.1 icmp_seq=3 ttl=64 time=10.621 ms (ICMP type:3, code:0, Destination network unreachable)
*192.168.112.1 icmp_seq=4 ttl=64 time=12.900 ms (ICMP type:3, code:0, Destination network unreachable)
*192.168.112.1 icmp_seq=5 ttl=64 time=11.092 ms (ICMP type:3, code:0, Destination network unreachable)

pc2-1> ping 192.168.112.3

84 bytes from 192.168.112.3 icmp_seq=1 ttl=64 time=53.597 ms
84 bytes from 192.168.112.3 icmp_seq=2 ttl=64 time=71.608 ms
84 bytes from 192.168.112.3 icmp_seq=3 ttl=64 time=49.277 ms
84 bytes from 192.168.112.3 icmp_seq=4 ttl=64 time=55.580 ms
84 bytes from 192.168.112.3 icmp_seq=5 ttl=64 time=59.190 ms

pc2-1>
```
Одновременно получаю с Leaf-1 вот такой Update
<img width="928" height="959" alt="image" src="https://github.com/user-attachments/assets/26b08d5c-0c4f-4732-b653-702b7c7689fc" />
Gratius ARP-шки бегают когда пытаюсь первый раз попробовать с кем-то с новым связь построить, вот ниж пример:
<img width="939" height="721" alt="image" src="https://github.com/user-attachments/assets/9dae54bc-a5a6-4ae0-bb8a-9541b625490c" />
### Проверка Evpn BGP маршрутов
```
leaf2#sho bgp evpn
BGP routing table information for VRF default
Router identifier 10.1.0.2, local AS number 65502
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.1.0.1:1 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:1 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65500 65501 i
 * >Ec    RD: 10.1.0.1:2 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:2 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65500 65501 i
 * >      RD: 10.1.0.2:112 imet 10.1.0.2
                                 -                     -       -       0       i
 * >      RD: 10.1.0.2:113 imet 10.1.0.2
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.3:112 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65500 65503 i
 *  ec    RD: 10.1.0.3:112 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65500 65503 i
leaf2#
```
## Просмотр Vxlan Vtep
```
leaf2# show vxlan vtep detail
Remote VTEPS for Vxlan1:

VTEP           Learned Via         MAC Address Learning       Tunnel Type(s)
-------------- ------------------- -------------------------- --------------
10.1.0.1       control plane       control plane              flood
10.1.0.3       control plane       control plane              flood

Total number of remote VTEPS:  2
```
## Провекрка появления mac адреса после пинга
Пингуем, т.к. до пинга таблица мак адресов девственно чиста.
```
pc2-1> ping 192.168.112.3

192.168.112.3 icmp_seq=1 timeout
84 bytes from 192.168.112.3 icmp_seq=2 ttl=64 time=45.718 ms
84 bytes from 192.168.112.3 icmp_seq=3 ttl=64 time=92.772 ms
```

Теперь проверяем появился ли MAC адрес на Leaf2

```
sho vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
 112  0050.7966.6805  EVPN      Vx1  10.1.0.3         1       0:00:04 ago
Total Remote Mac Addresses for this criterion: 1

```

После пинга мы видим появился мак акдрес PC3-1 в таблице vxlan leaf2

## Проверка mac адреса pc3-1
```
pc3-1> show ip

NAME        : pc3-1[1]
IP/MASK     : 192.168.112.3/24
GATEWAY     : 192.168.112.1
MAC         : 00:50:79:66:68:05
pc3-1>
```
## Посмотрим что-за маршруты есть поподробней
```
leaf2#sho bgp evpn route-type imet detail
BGP routing table information for VRF default
Router identifier 10.1.0.2, local AS number 65502
BGP routing table entry for imet 10.1.0.1, Route Distinguisher: 10.1.0.1:1
 Paths: 2 available
  65500 65501
    10.1.0.1 from 10.1.1.1 (10.1.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan
      VNI: 1112
      PMSI Tunnel: Ingress Replication, MPLS Label: 1112, Leaf Information Required: false, Tunnel ID: 10.1.0.1
  65500 65501
    10.1.0.1 from 10.2.2.2 (10.2.2.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan
      VNI: 1112
      PMSI Tunnel: Ingress Replication, MPLS Label: 1112, Leaf Information Required: false, Tunnel ID: 10.1.0.1
BGP routing table entry for imet 10.1.0.1, Route Distinguisher: 10.1.0.1:2
 Paths: 2 available
  65500 65501
    10.1.0.1 from 10.2.2.2 (10.2.2.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:65500:1222 TunnelEncap:tunnelTypeVxlan
      VNI: 1222
      PMSI Tunnel: Ingress Replication, MPLS Label: 1222, Leaf Information Required: false, Tunnel ID: 10.1.0.1
  65500 65501
    10.1.0.1 from 10.1.1.1 (10.1.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:65500:1222 TunnelEncap:tunnelTypeVxlan
      VNI: 1222
      PMSI Tunnel: Ingress Replication, MPLS Label: 1222, Leaf Information Required: false, Tunnel ID: 10.1.0.1
BGP routing table entry for imet 10.1.0.2, Route Distinguisher: 10.1.0.2:112
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan
      VNI: 1112
      PMSI Tunnel: Ingress Replication, MPLS Label: 1112, Leaf Information Required: false, Tunnel ID: 10.1.0.2
BGP routing table entry for imet 10.1.0.2, Route Distinguisher: 10.1.0.2:113
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: Route-Target-AS:65500:1113 TunnelEncap:tunnelTypeVxlan
      VNI: 1113
      PMSI Tunnel: Ingress Replication, MPLS Label: 1113, Leaf Information Required: false, Tunnel ID: 10.1.0.2
BGP routing table entry for imet 10.1.0.3, Route Distinguisher: 10.1.0.3:112
 Paths: 2 available
  65500 65503
    10.1.0.3 from 10.1.1.1 (10.1.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan
      VNI: 1112
      PMSI Tunnel: Ingress Replication, MPLS Label: 1112, Leaf Information Required: false, Tunnel ID: 10.1.0.3
  65500 65503
    10.1.0.3 from 10.2.2.2 (10.2.2.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan
      VNI: 1112
      PMSI Tunnel: Ingress Replication, MPLS Label: 1112, Leaf Information Required: false, Tunnel ID: 10.1.0.3
leaf2#
```
## Просмотр суммарной информации evpng bgp:
```
leaf2# sho bgp evpn summary
BGP summary information for VRF default
Router identifier 10.1.0.2, local AS number 65502
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.1.1.1 4 65500         149289    149343    0    0    4d10h Estab   3      3
  10.2.2.2 4 65500         149219    149275    0    0    4d10h Estab   3      3
```
## Просмотр bgp с leaf3
```
leaf3#sho bgp summary
BGP summary information for VRF default
Router identifier 10.1.0.3, local AS number 65503
Neighbor          AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
-------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.0.3.1       65500 Established   IPv4 Unicast            Negotiated              3          3
10.0.3.5       65500 Established   IPv4 Unicast            Negotiated              3          3
10.1.1.1       65500 Established   L2VPN EVPN              Negotiated              6          6
10.2.2.2       65500 Established   L2VPN EVPN              Negotiated              6          6
leaf3#
```
## Просмотр BGP EVPN на spine2
```
spine2#sho bgp evpn
BGP routing table information for VRF default
Router identifier 10.2.2.2, local AS number 65500
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.0.1:1 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65501 i
 * >      RD: 10.1.0.1:2 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65501 i
 * >      RD: 10.1.0.2:112 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65502 i
 * >      RD: 10.1.0.2:113 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65502 i
 * >      RD: 10.1.0.3:112 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65503 i
spine2#
```
## Просмотр bgp на spine2
```
spine2#sho bgp summary
BGP summary information for VRF default
Router identifier 10.2.2.2, local AS number 65500
Neighbor          AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
-------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.0.1.4       65501 Established   IPv4 Unicast            Negotiated              1          1
10.0.2.4       65502 Established   IPv4 Unicast            Negotiated              1          1
10.0.3.4       65503 Established   IPv4 Unicast            Negotiated              1          1
10.1.0.1       65501 Established   L2VPN EVPN              Negotiated              2          2
10.1.0.2       65502 Established   L2VPN EVPN              Negotiated              2          2
10.1.0.3       65503 Established   L2VPN EVPN              Negotiated              1          1
```

## Просмотр интерфейса Vxlan
```
leaf2#
ho interfaces vxlan 1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback1 and is active with 10.1.0.2
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is
    [112, 1112]       [113, 1113]
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is not configured
  Headend replication flood vtep list is:
   112 10.1.0.1        10.1.0.3
  Shared Router MAC is 0000.0000.0000
leaf2#
```
## Пример конфига Leaf2
```
leaf2#sho run
! Command: show running-config
! device: leaf2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname leaf2
!
spanning-tree mode mstp
!
vlan 112
   name Host_network_Vlan_112
!
vlan 113
!
interface Ethernet1
   description Peer-to-peer link to Spine-1
   no switchport
   ip address 10.0.2.0/31
!
interface Ethernet2
   description Peer-to-peer link to Spine-2
   no switchport
   ip address 10.0.2.4/31
!
interface Ethernet3
   description to host 112 VLAN
   switchport access vlan 112
!
interface Ethernet4
   description to pc-host 2-2
   switchport access vlan 113
!
interface Ethernet5
   description -=Direction to hosts=-
   no switchport
   ip address 192.168.2.1/24
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback1
   description VTEP
   ip address 10.1.0.2/32
!
interface Management1
!
interface Vlan112
   ip address 192.168.112.1/24
!
interface Vlan113
   ip address 192.168.113.1/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 112 vni 1112
   vxlan vlan 113 vni 1113
!
ip routing
!
router bgp 65502
   router-id 10.1.0.2
   no bgp default ipv4-unicast
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
   neighbor UNDERLAY send-community extended
   neighbor 10.0.2.1 peer group UNDERLAY
   neighbor 10.0.2.5 peer group UNDERLAY
   neighbor 10.1.1.1 peer group SPINE-EVPN
   neighbor 10.2.2.2 peer group SPINE-EVPN
   !
   vlan 112
      rd auto
      route-target both 65500:1112
      redistribute learned
   !
   vlan 113
      rd auto
      route-target both 65500:1113
      redistribute learned
   !
   address-family evpn
      neighbor SPINE-EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.1.0.2/32
!
end
leaf2#
```
## Пример конфига lEAF3
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
vlan 112
   name 112
!
vlan 113
   name 113
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
   switchport access vlan 112
!
interface Ethernet4
   switchport access vlan 112
!
interface Loopback1
   description VTEP
   ip address 10.1.0.3/32
!
interface Management1
!
interface Vlan112
   ip address 192.168.112.1/24
!
interface Vlan113
   ip address 192.168.113.2/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 112 vni 1112
!
ip routing
!
router bgp 65503
   router-id 10.1.0.3
   no bgp default ipv4-unicast
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
   neighbor UNDERLAY send-community extended
   neighbor 10.0.3.1 peer group UNDERLAY
   neighbor 10.0.3.5 peer group UNDERLAY
   neighbor 10.1.1.1 peer group SPINE-EVPN
   neighbor 10.2.2.2 peer group SPINE-EVPN
   !
   vlan 112
      rd auto
      route-target both 65500:1112
      redistribute learned
   !
   vlan 113
      rd auto
      route-target both 65500:1113
      redistribute learned
   !
   address-family evpn
      neighbor SPINE-EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.1.0.3/32
!
end
leaf3#
```
Кто дочитал, тот молодец:) Всем пока.
