# Настройка VxLAN. L3VNI
Цель:
Настроить маршрутизацию в рамках Overlay между клиентами.


Описание/Пошаговая инструкция выполнения домашнего задания:
В этой самостоятельной работе мы ожидаем, что вы самостоятельно:

Настроите каждого клиента в своем VNI
Настроите маршрутизацию между клиентами.
Зафиксируете в документации - план работы, адресное пространство, схему сети, конфигурацию устройств


ho bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65501
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.1.0.3:112 mac-ip 0050.7966.6805
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



leaf1#sho vxlan vtep
Remote VTEPS for Vxlan1:

VTEP           Tunnel Type(s)
-------------- --------------
10.1.0.2       flood, unicast
10.1.0.3       flood

Total number of remote VTEPS:  2





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


## IP Адрес pc2-1
VPCS> show ip

NAME        : VPCS[1]
IP/MASK     : 192.168.112.112/24
GATEWAY     : 192.168.112.1
DNS         :
MAC         : 00:50:79:66:68:08
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500

VPCS>
leaf2#
leaf2#
