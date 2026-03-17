# VxLAN. Оптимизация таблиц маршрутизации
## Цели занятия
- разобрать __EVPN route-type 5__ и его применение;
- настроить __route-type__ для оптимизации маршрутизации.

## Краткое содержание
- разберем передачу маршрутной инфомарции EVPN type-5 и настройку EVPN для передачи маршрутной информации через type-5 анонсы.
## Cхема стенда
<img width="1717" height="871" alt="image" src="https://github.com/user-attachments/assets/70667e94-2921-4b76-927a-86884e000260" /> <br>
как видно из схемы,  к предыдущей топологии мы "присобачили" собаке хвост справа, называется "EdgeRouter"
<br>
## Просмотр Route type 5 маршрутов, на новом подключенном устройстве
```
EdgeRouter#show bgp evpn route-type ip-prefix ipv4
BGP routing table information for VRF default
Router identifier 5.5.5.6, local AS number 65504
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 65501:1 ip-prefix 192.168.112.0/24
                                 10.1.0.1              -       100     0       65503 65500 65501 i
 * >      RD: 65502:1 ip-prefix 192.168.112.0/24
                                 10.1.0.2              -       100     0       65503 65500 65502 i
 * >      RD: 65503:1 ip-prefix 192.168.113.0/24
                                 10.1.0.3              -       100     0       65503 i
EdgeRouter#
```
<details>
  <summary>EdgeRouter конфигурация</summary>

  ```
EdgeRouter#sho run
! Command: show running-config
! device: EdgeRouter (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname EdgeRouter
!
spanning-tree mode mstp
!
vrf instance VRF_GOOGLE
   rd 65504:1
!
interface Ethernet1
!
interface Ethernet2
!
interface Ethernet3
   description rouiting port to google
   no switchport
   ip address 8.8.8.254/24
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
   description Interconnect to BorderLeaf3
   no switchport
   ip address 5.5.5.6/30
!
interface Loopback1
   ip address 10.0.0.4/32
!
interface Management1
!
interface Vlan101
!
ip routing
ip routing vrf VRF_GOOGLE
!
router bgp 65504
   router-id 5.5.5.6
   no bgp default ipv4-unicast
   timers bgp 3 9
   neighbor test peer group
   neighbor test remote-as 65503
   neighbor test next-hop-self
   neighbor test update-source Ethernet8
   neighbor test route-reflector-client
   neighbor test send-community standard extended
   neighbor 5.5.5.5 peer group test
   redistribute connected
   !
   address-family evpn
      neighbor test activate
   !
   address-family ipv4
      neighbor test activate
      network 5.5.5.6/32
      network 8.8.8.8/32
      network 10.3.3.3/32
   !
   vrf VRF_GOOGLE
      rd 65504:1
      route-target import evpn 65504:1
      route-target export evpn 65504:1
      network 8.8.8.8/32
!
end
EdgeRouter#

```

</details>

  <details>
  <summary>Leaf3 конфигурация</summary>
  

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
vlan 4000
   name BGP-INTERCONNECT
!
vrf instance VRF_GOOGLE
   rd 65503:100
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
interface Loopback100
   description For BGP between VRF
   ip address 192.168.100.3/32
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
interface Vlan4000
   description BGP interconnect between default and VRF_GOOGLE
   vrf VRF_GOOGLE
   ip address 10.254.254.2/30
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
ip routing vrf VRF_GOOGLE
no ip routing vrf v
ip routing vrf vrf1
ip routing vrf vrf2
!
ip prefix-list GOOGLE-NET seq 10 permit 8.8.8.0/24
!
route-map IMPORT-GOOGLE permit 10
   match ip address prefix-list GOOGLE-NET
!
route-map SELECT-GOOGLE permit 10
   match ip address prefix-list GOOGLE-NET
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
   neighbor 10.254.254.2 remote-as 65503
   neighbor 10.254.254.2 next-hop-self
   neighbor 10.254.254.2 update-source Vlan4000
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
      network 8.8.8.0/24
      network 10.1.0.3/32
      redistribute connected
   !
   vrf VRF_GOOGLE
      rd 65503:100
      route-target import evpn 65503:100
      route-target export evpn 65503:100
      neighbor 10.254.254.1 remote-as 65503
      neighbor 10.254.254.1 update-source Vlan4000
      !
      address-family ipv4
         bgp route install-map IMPORT-GOOGLE
         neighbor 10.254.254.1 activate
         network 8.8.8.0/24
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
leaf3#


```

  </details>

  И так как мы видели выше, на EdgeRouter прилетели маршруты Route type 5 из нашей фабрики.
  <br>
  Давайте, попробуем завернуть теперь маршруты полученные leaf3 от RouterEdge  обратно в фабрику как __Route Type 5 - маршруты__ <br>
  
Ниже представлен вывод глобальной таблицы маршрутизации на leaf3 <br>

```
sho ip route

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
```
Меня интересует в частности сеть 8.8.8.0.24 за которой находится хост 8.8.8.8
Маршрут до 8.8.8.0/24 уже есть на leaf3 через BGP от EdgeRouter.
```
leaf3#ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 72(100) bytes of data.
80 bytes from 8.8.8.8: icmp_seq=1 ttl=63 time=208 ms
80 bytes from 8.8.8.8: icmp_seq=2 ttl=63 time=207 ms
80 bytes from 8.8.8.8: icmp_seq=3 ttl=63 time=255 ms
```
Требуется "за leaking" этот маршрут в EVPN route type 5 на leaf3, чтобы другие листья (leaf1, leaf2) увидели его как Route type 5. <br>

Глобально существует 3-и способа это сделать.
## 1. VXLAN L3VPN с использованием СТАТИЧЕСКИХ Маршрутов
1-й способ использовать статические маршруты (большие минусы, так как плохо масштабируемая история)
## 2. L3VPN c использованием VRF поверх BGP 
Хотя такая конфигурация масштабируется лучше, чем использование только статических маршрутов, мы по-прежнему полагаемся на статические маршруты для подключения VXLAN. Если маршрутизатор добавляется в VPN, все остальные маршрутизаторы в этой VPN должны добавить статический маршрут для этого нового маршрутизатора. Кроме того, конфигурация BGP выполняется для каждого VRF, поэтому возникают те же проблемы; любой новый маршрутизатор должен быть добавлен в качестве соседа BGP внутри VRF. По мере увеличения количества VRF и соседей, накладные расходы BGP будут расти. Использование Route-reflector может снизить масштабируемость BGP. Существует ли более лучшая история?, Да, это EVPN.
## 3. L3VPN с EVPN-ном
Это наиболее масштабируемое решение для VXLAN L3VPN.
<br><br>
Забегая вперед, скажу ни один из 3-х способов у меня не заработал)
И это вылевается в очень интересную историю. Предлагаю сменить тему моей проектной работы: "Проектирование и реализация отказоустойчивой, гетерогенной, географически распределенной сети ЦОД на базе EVPN-VXLAN" на новую тему, посвященную использованию Route type 5 маршрутов в сетях фабрик.  <br>
<br>

Благодарю за внимание!

<details>
  <summary>leaf3 конфигурация изначальная</summary>
       
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
 
  






   
