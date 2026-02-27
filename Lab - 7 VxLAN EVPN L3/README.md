# Настройка VxLAN. L3VNI
__Цель:__ <br>
Настроить маршрутизацию в рамках Overlay между подсетями VLAN, используя:
- __VxLAN EVPN__ как Control-plane
- __Symmetric IRB__ (L2VNI для каждого VLAN + выделенный L3VNI для VRF <br><br>
__План работ:__ <br>
- Настроить каждого клиента в своем VNI
- Настроить маршрутизацию между клиентами
Схема стенда снизу:
<img width="1419" height="918" alt="image" src="https://github.com/user-attachments/assets/c7295b7c-d77d-44c4-a71e-046a72faf794" /> <br>

## Архитектура
- Underlay: eBGP — обеспечивает IP reachability между VTEP 
- Overlay: BGP EVPN
- Data plane: VxLAN (UDP/4789)
- VTEP: Leaf-коммутаторы (Loopback1 используется как VTEP IP)
## L3VNI (VRF → VNI)
Для symmetric IRB используется выделенный VRF и отдельный VNI:

- VRF  -vrf1|
- L3VNI	 - 666
-  RT (EVPN) - 1:666 

Leaf’ы экспортируют connected-prefix’ы (SVI подсети) в EVPN как __Type-5__ маршруты.
## Anycast Default Gateway
Для клиентов в каждом VLAN используется Anycast GW на Leaf:

VLAN	Anycast GW IP <br>
112	192.168.112.254 <br> 
113	192.168.113.254 <br>
На всех Leaf задан общий виртуальный MAC:
```
ip virtual-router mac-address 02:00:00:00:00:00
```
## Просмотр диагностических команд
Начну с того, что все уже настроено, и давайте посмотрим на выводы команд, и что интересного мы можем  увидеть, по сравнению с предыдущей лабораторной работой. Пример конфигурации устройств, представлен в эпилоге данной статьи. Рисунок схемы лабораторного стенда, представлен снизу:
## Просмотр маршрутов mac-ip на устройстве leaf1
```
sho bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65501
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.1.0.3:112 mac-ip 0050.7966.`**6805`**
                                 10.1.0.3              -       100     0       65500 65503 i
 *  ec    RD: 10.1.0.3:112 mac-ip 0050.7966.6805
                                 10.1.0.3              -       100     0       65500 65503 i
 * >      RD: 10.1.0.1:112 mac-ip 0050.7966.6807
                                 -                     -       -       0       i
 * >      RD: 10.1.0.1:112 mac-ip 0050.7966.6807 192.168.112.2
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.2:112 mac-ip 0050.7966.6808
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:112 mac-ip 0050.7966.6808
                                 10.1.0.2              -       100     0       65500 65502 i
 * >Ec    RD: 10.1.0.2:112 mac-ip 0050.7966.6808 192.168.112.3
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:112 mac-ip 0050.7966.6808 192.168.112.3
                                 10.1.0.2              -       100     0       65500 65502 i
 * >Ec    RD: 10.1.0.2:112 mac-ip 0050.7966.6808 192.168.112.112
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:112 mac-ip 0050.7966.6808 192.168.112.112
                                 10.1.0.2              -       100     0       65500 65502 i
 * >      RD: 10.1.0.1:113 mac-ip 0050.7966.680a
                                 -                     -       -       0       i
 * >      RD: 10.1.0.1:113 mac-ip 0050.7966.680a 192.168.113.2
                                 -                     -       -       0       i
leaf1#
```
## MAC адрес pc3-1  и разбор вывода bgp evpn route-type mac-ip
```
PC3-1> show ip
MAC         : 00:50:79:66:68:05
```
<img width="1212" height="198" alt="image" src="https://github.com/user-attachments/assets/6fa64875-4a06-4e19-a966-3bb687cc5326" />
И так картинку выше (это часть картинки из общего вывода с __leaf1__ ) я понимаю следующем образом, до mac адреса хоста (*68:05), который живет за leaf3, существует 2-ва маршрута, через AS 65500 и конечная точка AS65503, но понять через какой Spine пойдет трафик из данной команды не получится, и вообще пока не понятно можно ли это узнать не залезая в FIB. <br><br>
Ниже еще пример для других адресов
А если посмотреть чуть по ниже, например MAC адрес (*6808)(это хост pc2-1) для него значится аж 6-ть записей, которые живут за leaf2 (10.1.0.2) смотреть снизу
Почему так?(за одним mac, два разных IP) мне лично не совсем понятно, возможно Я менял IP на хосте, и они остались в буфере, я уже не помню, другого объяснения не могу не нахожу.

```
pc2-1> sho ip
NAME        : pc2-1
MAC         : 00:50:79:66:68:08
```

<img width="1215" height="415" alt="image" src="https://github.com/user-attachments/assets/98686f20-bf56-4938-9e31-989a72feaf30" /> <br>
## Просмотр интерфейса Vxlan1
<br>

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
    [112, 1112]       [113, 1113]
  Dynamic VLAN to VNI mapping for 'evpn' is
    [4094, 666]
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is
   [vrf1, 666]
  Headend replication flood vtep list is:
   112 10.1.0.2        10.1.0.3
   113 10.1.0.2
  Shared Router MAC is 0000.0000.0000
leaf1#
```
<br>

## Просмотр BGP evpn ip в.4 маршрутов
```
leaf1#sho bgp evpn route-type ip-prefix ipv4
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65501
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 65501:1 ip-prefix 192.168.112.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 65502:1 ip-prefix 192.168.112.0/24
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 65502:1 ip-prefix 192.168.112.0/24
                                 10.1.0.2              -       100     0       65500 65502 i
 * >      RD: 65501:1 ip-prefix 192.168.113.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 65502:1 ip-prefix 192.168.113.0/24
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 65502:1 ip-prefix 192.168.113.0/24
                                 10.1.0.2              -       100     0       65500 65502 i
leaf1#
```
## Просмотр BGP EVPN маршрутов MAC-IP с Leaf2
```
leaf2#sho bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.1.0.2, local AS number 65502
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.1.0.1:112 mac-ip 0050.7966.6807
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:112 mac-ip 0050.7966.6807
                                 10.1.0.1              -       100     0       65500 65501 i
 * >Ec    RD: 10.1.0.1:112 mac-ip 0050.7966.6807 192.168.112.2
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:112 mac-ip 0050.7966.6807 192.168.112.2
                                 10.1.0.1              -       100     0       65500 65501 i
 * >      RD: 10.1.0.2:112 mac-ip 0050.7966.6808
                                 -                     -       -       0       i
 * >      RD: 10.1.0.2:112 mac-ip 0050.7966.6808 192.168.112.3
                                 -                     -       -       0       i
 * >      RD: 10.1.0.2:112 mac-ip 0050.7966.6808 192.168.112.112
                                 -                     -       -       0       i
leaf2#
```
## Просмотр Vxlan vtep на leaf1
```

leaf1#sho vxlan vtep
Remote VTEPS for Vxlan1:

VTEP           Tunnel Type(s)
-------------- --------------
10.1.0.2       flood, unicast
10.1.0.3       flood

Total number of remote VTEPS:  2
```
## Просмотр Vxlan vtep на leaf2
```

leaf2#sho vxlan vtep
Remote VTEPS for Vxlan1:

VTEP           Tunnel Type(s)
-------------- --------------
10.1.0.1       flood, unicast
10.1.0.3       flood

Total number of remote VTEPS:  2

VPCS> ping 192.168.112.112

84 bytes from 192.168.112.112 icmp_seq=1 ttl=64 time=99.696 ms
84 bytes from 192.168.112.112 icmp_seq=2 ttl=64 time=91.628 ms
84 bytes from 192.168.112.112 icmp_seq=3 ttl=64 time=67.951 ms
84 bytes from 192.168.112.112 icmp_seq=4 ttl=64 time=53.291 ms
84 bytes from 192.168.112.112 icmp_seq=5 ttl=64 time=43.411 ms

VPCS> show ip

NAME        : VPCS[1]
IP/MASK     : 192.168.112.2/24
GATEWAY     : 192.168.112.1
DNS         :
MAC         : 00:50:79:66:68:07
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500
```


## Просмотр mac Адреса pc2-1
```
pc2-1> show ip

NAME        : VPCS[1]
IP/MASK     : 192.168.112.112/24
GATEWAY     : 192.168.112.1
```
__MAC         : 00:50:79:66:68:08__

## Просмотр vrf routing on Leaf2
```
leaf2#sho ip route vrf vrf1

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

 C        192.168.112.0/24 is directly connected, Vlan112
 C        192.168.113.0/24 is directly connected, Vlan113
```

## Просмотр vrf маршрутизации на leaf1
```
leaf1#sho ip route vrf vrf1

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

 C        192.168.112.0/24 is directly connected, Vlan112
 C        192.168.113.0/24 is directly connected, Vlan113

leaf1#
```
## Просмотр vrf маршрутов после пинга с pc1-1 pc2-2
```
pc1-1> ping 192.168.113.3

84 bytes from 192.168.113.3 icmp_seq=1 ttl=63 time=162.578 ms
84 bytes from 192.168.113.3 icmp_seq=2 ttl=63 time=67.794 ms
84 bytes from 192.168.113.3 icmp_seq=3 ttl=63 time=91.946 ms
84 bytes from 192.168.113.3 icmp_seq=4 ttl=63 time=121.316 ms
84 bytes from 192.168.113.3 icmp_seq=5 ttl=63 time=85.178 ms

pc1-1> sho ip

NAME        : pc1-1[1]
IP/MASK     : 192.168.112.2/24
GATEWAY     : 192.168.112.1
DNS         :
MAC         : 00:50:79:66:68:07

leaf1#
how ip route vrf vrf1
C        192.168.112.0/24 is directly connected, Vlan112
 B E      192.168.113.3/32 [200/0] via VTEP 10.1.0.2 VNI 666 router-mac 50:00:00:cb:38:c2 local-interface Vxlan1
 C        192.168.113.0/24 is directly connected, Vlan113
```
Видим, что добавился маршрут между до pc2-1
<img width="1238" height="143" alt="image" src="https://github.com/user-attachments/assets/91a85ec9-83c2-4074-8b0a-3cb4131a6f2c" />


```
leaf1#sho bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65501
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.0.1:112 mac-ip 0050.7966.6807
                                 -                     -       -       0       i
 * >      RD: 10.1.0.1:112 mac-ip 0050.7966.6807 192.168.112.2
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.2:113 mac-ip 0050.7966.680b 192.168.113.3
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:113 mac-ip 0050.7966.680b 192.168.113.3
                                 10.1.0.2              -       100     0       65500 65502 i
leaf1#
```
## Пингуем и смотрим Смотрим дампы пинга
<img width="925" height="380" alt="image" src="https://github.com/user-attachments/assets/c8c08dc0-02d9-4459-bed9-798a400005bd" /> <br>
И так с хоста pc1-1 ping-уем хост pc2-1, который находится за leaf2
```
PC1-1> ping 192.168.112.112

84 bytes from 192.168.112.112 icmp_seq=1 ttl=64 time=88.546 ms
84 bytes from 192.168.112.112 icmp_seq=2 ttl=64 time=112.694 ms
84 bytes from 192.168.112.112 icmp_seq=3 ttl=64 time=49.601 ms
84 bytes from 192.168.112.112 icmp_seq=4 ttl=64 time=60.305 ms
84 bytes from 192.168.112.112 icmp_seq=5 ttl=64 time=44.472 ms

PC1-1> show ip

NAME        : PC1-1
IP/MASK     : 192.168.112.2/24
GATEWAY     : 192.168.112.1
MAC         : 00:50:79:66:68:07
```
Как мы видим из картинки выше, для IP адрессации используются адрес интерфейса  Loopback1, который был создан специально для Vxlan Overlay сети. Но vrf EVPN L3 не используется. Ping проходит за счет L2 Evpn.

## Проверка на leaf1
```
leaf1#
ho bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65501
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.0.1:112 mac-ip 0050.7966.6807
                                 -                     -       -       0       i
 * >      RD: 10.1.0.1:112 mac-ip 0050.7966.6807 192.168.112.2
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.2:112 mac-ip 0050.7966.6808
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:112 mac-ip 0050.7966.6808
                                 10.1.0.2              -       100     0       65500 65502 i
 * >Ec    RD: 10.1.0.2:112 mac-ip 0050.7966.6808 192.168.112.3
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:112 mac-ip 0050.7966.6808 192.168.112.3
                                 10.1.0.2              -       100     0       65500 65502 i
 * >Ec    RD: 10.1.0.2:112 mac-ip 0050.7966.6808 192.168.112.112
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:112 mac-ip 0050.7966.6808 192.168.112.112
                                 10.1.0.2              -       100     0       65500 65502 i
leaf1#
```
## Просмотр BGP EVPN route mac-ip детально c Leaf1
```
leaf1#sho bgp evpn route-type mac-ip detail
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65501
BGP routing table entry for mac-ip 0050.7966.6807, Route Distinguisher: 10.1.0.1:112
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan
      VNI: 1112 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip 0050.7966.6807 192.168.112.2, Route Distinguisher: 10.1.0.1:112
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: Route-Target-AS:1:666 Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan
      VNI: 1112 L3 VNI: 666 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip 0050.7966.6808, Route Distinguisher: 10.1.0.2:112
 Paths: 2 available
  65500 65502
    10.1.0.2 from 10.2.2.2 (10.2.2.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan
      VNI: 1112 ESI: 0000:0000:0000:0000:0000
  65500 65502
    10.1.0.2 from 10.1.1.1 (10.1.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan
      VNI: 1112 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip 0050.7966.6808 192.168.112.3, Route Distinguisher: 10.1.0.2:112
 Paths: 2 available
  65500 65502
    10.1.0.2 from 10.2.2.2 (10.2.2.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:666 Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:cb:38:c2
      VNI: 1112 L3 VNI: 666 ESI: 0000:0000:0000:0000:0000
  65500 65502
    10.1.0.2 from 10.1.1.1 (10.1.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:1:666 Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:cb:38:c2
      VNI: 1112 L3 VNI: 666 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip 0050.7966.6808 192.168.112.112, Route Distinguisher: 10.1.0.2:112
 Paths: 2 available
  65500 65502
    10.1.0.2 from 10.2.2.2 (10.2.2.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:666 Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:cb:38:c2
      VNI: 1112 L3 VNI: 666 ESI: 0000:0000:0000:0000:0000
  65500 65502
    10.1.0.2 from 10.1.1.1 (10.1.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:1:666 Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:cb:38:c2
      VNI: 1112 L3 VNI: 666 ESI: 0000:0000:0000:0000:0000
leaf1#
```
Теперь мы видим, что до pc2-1 (mac *6808) c leaf1, существует 2-ва маршрута, через spine1 и spine2, ниже подсветил фломастером:
<img width="1610" height="405" alt="image" src="https://github.com/user-attachments/assets/c830b4d7-f621-442a-bb1a-1e92c3970910" />
<img width="1150" height="496" alt="image" src="https://github.com/user-attachments/assets/53178607-d742-41fd-8128-8272c5a6c190" />
## Информация с leaf3
Снизу представлена информация об интерфейсе Vxlan1, он "поднят", для своей работы и использует IP адрес Loopback 1 (10.1.0.3), для расспространение mac информации о 112 VLAN использует следующие ip адреса Vtep-ов :  10.1.0.1        10.1.0.2 <br><br>
```
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback1 and is active with 10.1.0.3
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is
    [112, 1112]
  Dynamic VLAN to VNI mapping for 'evpn' is
    [4094, 666]
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is not configured
  Headend replication flood vtep list is:
   112 10.1.0.1        10.1.0.2
  Shared Router MAC is 0000.0000.0000
leaf3#
```
## Просмотр имеющихся VNI на leaf3
```
leaf3#
leaf3# sho vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI        VLAN       Source       Interface       802.1Q Tag
---------- ---------- ------------ --------------- ----------
1112       112        static       Ethernet3       untagged
                                   Ethernet4       untagged
                                   Vxlan1          112

VNI to dynamic VLAN Mapping for Vxlan1
VNI       VLAN       VRF       Source
--------- ---------- --------- ------------
666       4094                 evpn
```
# Разбор Wireshark
## Разбирать будем  описанный выше пример пинга с хоста pc1-1 ping-уем хост pc2-1, который находится за leaf2
И так с Pc1-1(192.168.112.2) выполняем ping 192.168.113.3 (pc2-2)
Трейс снимался на leaf1 (eth1)
<img width="1251" height="306" alt="image" src="https://github.com/user-attachments/assets/bc0b2e4e-3a25-4240-b873-43fafb6b95ba" />
После пинга мы видим на уровне VxLAN бродкаст домена, широковещательное сообщение ARP, кто есть такой 192.168.113.2 ?
Данный Мегафрейм с Vxlan содержанием отправляется от leaf1 к Spine1 и вероятно к Spine2, но трафик я снимал только на eth1 leaf1, поэтому я этого не узнаю).
И так вот этим широковещательным запросом, __Кто-ТЫ такой 192.168.113.2???__ достучались до нашего испытуемого подопечного, естественно он безприкословно ответил, даже не подозревая сколько сверху еще добавиться заголовков, в формате описанном в rfc9135 секция 7.1, картинка ниже:
<img width="1560" height="1122" alt="image" src="https://github.com/user-attachments/assets/3a305ffb-7510-4bbf-ad26-d8103636d853" />
Leaf1 отправляет Gratius ARP replay в сторону spine1, согласно RFC 9135/9136, пример ниже:
<img width="1343" height="813" alt="image" src="https://github.com/user-attachments/assets/d48e1940-85fe-4bdf-95a0-ba432ab1fc18" />

# Конфигурация устройств
## Leaf1
```
leaf1#sho run
! Command: show running-config
! device: leaf1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname leaf1
!
spanning-tree mode mstp
!
vlan 1
   name Host_Network
!
vlan 2
   name 2
!
vlan 112-113
!
vrf instance vrf1
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
   switchport access vlan 112
!
interface Ethernet4
   description to pc 1-2
   switchport access vlan 113
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback1
   description VTEP
   ip address 10.1.0.1/32
!
interface Management1
!
interface Vlan112
   vrf vrf1
   ip address 192.168.112.1/24
   ip virtual-router address 192.168.112.254/24
!
interface Vlan113
   vrf vrf1
   ip address 192.168.113.1/24
   ip virtual-router address 192.168.113.254/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 112 vni 1112
   vxlan vlan 113 vni 1113
   vxlan vrf vrf1 vni 666
!
ip virtual-router mac-address 02:00:00:00:00:00
!
ip routing
ip routing vrf vrf1
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
      network 10.1.0.1/32
   !
   vrf vrf1
      rd 65501:1
      route-target import evpn 1:666
      route-target export evpn 1:666
      redistribute connected
```
## Leaf2
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
vrf instance vrf1
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
   vrf vrf1
   ip address 192.168.112.2/24
   ip virtual-router address 192.168.112.254
!
interface Vlan113
   vrf vrf1
   ip address 192.168.113.2/24
   ip virtual-router address 192.168.113.254
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 112 vni 1112
   vxlan vlan 113 vni 1113
   vxlan vrf vrf1 vni 666
!
ip virtual-router mac-address 02:00:00:00:00:00
!
ip routing
ip routing vrf vrf1
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
   vrf vrf1
      rd 65502:1
      route-target import evpn 1:666
      route-target export 1:666
      redistribute connected
```
## Leaf3
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
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback1
   description VTEP
   ip address 10.1.0.3/32
!
interface Management1
!
interface Vlan112
   ip address 192.168.112.1/24
   ip virtual-router address 192.168.112.254/24
!
interface Vlan113
   ip address 192.168.113.2/24
   ip virtual-router address 192.168.113.254/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 112 vni 1112
   vxlan learn-restrict any
!
ip virtual-router mac-address 02:00:00:00:00:00
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
```
## Spine 1
```
spine1#
spine1#sho run
! Command: show running-config
! device: spine1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
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
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback1
   description IP for underlay -Router-ID
   ip address 10.1.1.1/32
!
interface Management1
!
ip routing
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
end
spine1#
```
## Spine 2
```
spine2#sho run
! Command: show running-config
! device: spine2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname spine2
!
spanning-tree mode mstp
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
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback1
   description Overlay loopback
   ip address 10.2.2.2/32
!
interface Management1
!
ip routing
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
!
end
spine2#
```
