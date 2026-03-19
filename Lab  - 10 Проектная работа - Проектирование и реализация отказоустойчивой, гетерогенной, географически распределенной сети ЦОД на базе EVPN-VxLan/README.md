# Проектная работа
## Тема: Использование Route type 5 маршрутов в сетях фабрик
В этой статье мы сосредоточимся на сопоставлении VNI с VRF, что позволит нам создавать топологии L3VPN с использованием VXLAN. Я рассмотрю три способа построения L3VPN поверх VXLAN, от наименее масштабируемых до наиболее масштабируемых. Но сначала нам нужно рассмотреть некоторые базовые темы, так что давайте приступим! <br><br>

Использование VRF также гарантирует предотвращение утечки трафика из VPN-сети одного клиента в частную сеть другого. Кроме того, поскольку каждый клиент находится в своем собственном VRF, он может свободно использовать любую IP-адресацию по своему усмотрению без риска конфликта с другими клиентами.<br><br>

И так как мы выяснили из предыдущей лабораторной работы, глобально существует 3-и способа передать маршруты ipv4 из глобальной таблицы маршрутизации в EVPN route type 5. Ниже можно ознакомиться с данными способами.
Схема стенда. <br>
<img width="1717" height="871" alt="image" src="https://github.com/user-attachments/assets/21282b68-b454-4191-8767-5e01feb5c323" /> <br>

## 1. VXLAN L3VPN с использованием СТАТИЧЕСКИХ Маршрутов
1-й способ использовать статические маршруты (большие минусы, так как плохо масштабируемая история)
## 2. L3VPN c использованием VRF поверх BGP 
Хотя такая конфигурация масштабируется лучше, чем использование только статических маршрутов, мы по-прежнему полагаемся на статические маршруты для подключения VXLAN. <br>
Если маршрутизатор добавляется в VPN, все остальные маршрутизаторы в этой VPN должны добавить статический маршрут для этого нового маршрутизатора. <br>
Кроме того, конфигурация BGP выполняется для каждого VRF, поэтому возникают те же проблемы; любой новый маршрутизатор должен быть добавлен в качестве соседа BGP внутри VRF. По мере увеличения количества VRF и соседей, накладные расходы BGP будут расти.  <br>
Использование Route-reflector может снизить масштабируемость BGP. Существует ли более лучшая история?, Да, это EVPN.
## 3. L3VPN с EVPN-ном
Это наиболее масштабируемое решение для VXLAN L3VPN.
<br><br>  <br>
<br>
## Что такое VRF-система и зачем она нам нужна?
Технология L3VPN использует VRF для изоляции сети, поэтому вот краткое введение: <br>
<br>
VRF — это сокращение от Virtual Routing Forwarding (виртуальная пересылка маршрутов).<br>
При создании VRF внутри маршрутизатора создается виртуальная таблица маршрутизации.<br>
Затем вы можете назначить интерфейсы для VRF. Трафик, поступающий на этот интерфейс, будет маршрутизироваться в соответствии с маршрутами в этом конкретном VRF.<br>

И так я выбрал 2-й вариант с vrf-ами. <br>
И так как мы видели ниже, на EdgeRouter прилетели маршруты Route type 5 из нашей фабрики. <br>
пример вывода маршутов 5-го типа на Edge Router представлены ниже:

```
EdgeRouter#
gp evpn route-type ip-prefix ipv4
BGP routing table information for VRF default
Router identifier 5.5.5.6, local AS number 65504
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 65501:10100 ip-prefix 192.168.10.0/24
                                 10.1.0.1              -       100     0       65503 65500 65501 i
 * >      RD: 65502:10100 ip-prefix 192.168.10.0/24
                                 10.1.0.2              -       100     0       65503 65500 65502 i
 * >      RD: 65501:10100 ip-prefix 192.168.20.0/24
                                 10.1.0.1              -       100     0       65503 65500 65501 i
 * >      RD: 65502:10100 ip-prefix 192.168.20.0/24
                                 10.1.0.2              -       100     0       65503 65500 65502 i
 * >      RD: 65501:10200 ip-prefix 192.168.30.0/24
                                 10.1.0.1              -       100     0       65503 65500 65501 i
 * >      RD: 65502:10200 ip-prefix 192.168.30.0/24
                                 10.1.0.2              -       100     0       65503 65500 65502 i
 * >      RD: 65501:1 ip-prefix 192.168.112.0/24
                                 10.1.0.1              -       100     0       65503 65500 65501 i
 * >      RD: 65502:1 ip-prefix 192.168.112.0/24
                                 10.1.0.2              -       100     0       65503 65500 65502 i
 * >      RD: 65503:1 ip-prefix 192.168.113.0/24
                                 10.1.0.3              -       100     0       65503 i
EdgeRouter#
```
## Мои поиски решения по простому вопросу, как из IPV4 маршрута пришедшего на leaf, анонсировать его в bgp evpn route-type 5?
И так ниже пойдет описание сугубо личных переживаний, скажем так не для всех, но у кого такие же были проблемы, могут почитать тут:
 <details>
  <summary>Какие маршруты имеются на leaf3?</summary>
   
 ## Базовый ip маршрут
   
   ```
   leaf3#sho ip route

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
 C        192.168.100.3/32 is directly connected, Loopback100

leaf3#

```
## Просмотр vrf маршрутов

```
leaf3#sho ip route vrf all

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
 C        192.168.100.3/32 is directly connected, Loopback100


VRF: VRF_GOOGLE
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



VRF: v
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


! IP routing not enabled

VRF: vrf1
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

 B E      192.168.112.112/32 [200/0] via VTEP 10.1.0.1 VNI 666 router-mac 50:00:00:d7:ee:0b local-interface Vxlan1
                                     via VTEP 10.1.0.2 VNI 666 router-mac 50:00:00:cb:38:c2 local-interface Vxlan1
 B E      192.168.112.0/24 [200/0] via VTEP 10.1.0.1 VNI 666 router-mac 50:00:00:d7:ee:0b local-interface Vxlan1
                                   via VTEP 10.1.0.2 VNI 666 router-mac 50:00:00:cb:38:c2 local-interface Vxlan1
 C        192.168.113.0/24 is directly connected, Vlan113


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

```
## Просмотр bgp маршрутов

```
[Kaf3#sho ip bgp e
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65503
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      5.5.5.4/30             -                     -       -          -       0       i
 *        5.5.5.4/30             5.5.5.6               0       -          100     0       65504 i
          5.5.5.6/32             5.5.5.6               0       -          100     0       65504 i
 * >      8.8.8.0/24             5.5.5.6               0       -          100     0       65504 i
 * >      10.0.0.4/32            5.5.5.6               0       -          100     0       65504 i
 * >      10.0.3.0/31            -                     -       -          -       0       i
 * >      10.0.3.4/31            -                     -       -          -       0       i
 * >Ec    10.1.0.1/32            10.0.3.1              0       -          100     0       65500 65501 i
 *  ec    10.1.0.1/32            10.0.3.5              0       -          100     0       65500 65501 i
 * >Ec    10.1.0.2/32            10.0.3.1              0       -          100     0       65500 65502 i
 *  ec    10.1.0.2/32            10.0.3.5              0       -          100     0       65500 65502 i
 * >E     10.1.0.3/32            -                     -       -          -       0       i
 *  e     10.1.0.3/32            -                     -       -          -       0       i
 * >      10.1.1.1/32            10.0.3.1              0       -          100     0       65500 i
 * >      10.2.2.2/32            10.0.3.5              0       -          100     0       65500 i
 * >      192.168.100.3/32       -                     -       -          -       0       i
leaf3#
```
## Просмотр bgp route type-5
```
leaf3#sho bgp evpn route-type ip-prefix ipv4
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65503
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 65501:10100 ip-prefix 192.168.10.0/24
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 65501:10100 ip-prefix 192.168.10.0/24
                                 10.1.0.1              -       100     0       65500 65501 i
 * >Ec    RD: 65502:10100 ip-prefix 192.168.10.0/24
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 65502:10100 ip-prefix 192.168.10.0/24
                                 10.1.0.2              -       100     0       65500 65502 i
 * >Ec    RD: 65501:10100 ip-prefix 192.168.20.0/24
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 65501:10100 ip-prefix 192.168.20.0/24
                                 10.1.0.1              -       100     0       65500 65501 i
 * >Ec    RD: 65502:10100 ip-prefix 192.168.20.0/24
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 65502:10100 ip-prefix 192.168.20.0/24
                                 10.1.0.2              -       100     0       65500 65502 i
 * >Ec    RD: 65501:10200 ip-prefix 192.168.30.0/24
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 65501:10200 ip-prefix 192.168.30.0/24
                                 10.1.0.1              -       100     0       65500 65501 i
 * >Ec    RD: 65502:10200 ip-prefix 192.168.30.0/24
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 65502:10200 ip-prefix 192.168.30.0/24
                                 10.1.0.2              -       100     0       65500 65502 i
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
leaf3#
```

 </details>
<details> 
  <sammary>Конфигурация leaf3 (borderLeaf) перед тем как я собираюсь все сломать</sammary>
 
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
         redistribute connected
!
end
leaf3#
```
</details>
   
## Заключение: L3VPN с использованием статических маршрутов
Хотя конфигурация проста, масштабируемость оставляет желать лучшего. Каждый раз, когда маршрут добавляется или удаляется, его необходимо удалять со всех маршрутизаторов в VPN. Если в VPN 100 маршрутизаторов, это станет довольно утомительной задачей. Было бы лучше, если бы маршруты обменивались динамически с использованием протокола маршрутизации. Поэтому далее в статье мы рассмотрим, как улучшить масштабируемость, используя BGP внутри каждого VRF.
## L3VPN с BGP для каждого VRF
Вместо настройки статических маршрутов на всех маршрутизаторах в клиентской VPN было бы гораздо удобнее, если бы маршрутизатор автоматически объявлял маршруты клиента. <br><br>
Таким образом, хотя это решение лучше и обладает большей масштабируемостью, по-прежнему необходимы статические маршруты и BGP-соседства в каждом VRF. Можно ли сделать лучше? Давайте рассмотрим пример EVPN ниже, чтобы это выяснить.

## L3VPN с EVPN
Это наиболее масштабируемое решение для VXLAN L3VPN и, вероятно, именно та модель, которую вы захотите развернуть в своей сети.
Благодаря EVPN нам больше не нужны статические маршруты vtep. Для дальнейшего масштабирования этой сети можно развернуть маршрутизатор-рефлектор, с поддержкой семейств адресов BGP EVPN сделает его естественным маршрутизатором-рефлектором благодаря его центральному расположению в сети. В этом случае LEAF будут взаимодействовать с R1, а не друг с другом, что значительно повысит масштабируемость при наличии более 100 маршрутизаторов, поддерживающих BGP, в вашей сети.


