# Настройка VxLAN. L3VNI
__Цель:__ <br>
Настроить маршрутизацию в рамках Overlay между подсетями VLAN, используя:
- __VxLAN EVPN__ как Control-plane
- __Symmetric IRB__ (L2VNI для каждого VLAN + выделенный L3VNI для VRF <br><br>
__План работ:__ <br>
- Настроить каждого клиента в своем VNI
- Настроить маршрутизацию между клиентами
Схема стенда снизу:
<img width="1419" height="918" alt="image" src="https://github.com/user-attachments/assets/c7295b7c-d77d-44c4-a71e-046a72faf794" />


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

<img width="1215" height="415" alt="image" src="https://github.com/user-attachments/assets/98686f20-bf56-4938-9e31-989a72feaf30" />
## Просмотр интерфейса Vxlan1
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
leaf1#
how ip route vrf vrf1

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



