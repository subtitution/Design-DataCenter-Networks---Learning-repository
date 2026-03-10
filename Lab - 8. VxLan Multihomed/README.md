# VXLAN. Multihoming

## Цель:
Настроить отказоустойчивое подключение клиентов с использованием EVPN Multihoming
## Проверка предварительной базовой работоспособности 
На leaf1 выполним команду: ``` show ip route ```
```
 C        10.0.1.0/31 is directly connected, Ethernet1
 C        10.0.1.4/31 is directly connected, Ethernet2
 C        10.1.0.1/32 is directly connected, Loopback1
 B E      10.1.0.2/32 [200/0] via 10.0.1.1, Ethernet1
                              via 10.0.1.5, Ethernet2
 B E      10.1.0.3/32 [200/0] via 10.0.1.1, Ethernet1
                              via 10.0.1.5, Ethernet2
 B E      10.1.1.1/32 [200/0] via 10.0.1.1, Ethernet1
 B E      10.2.2.2/32 [200/0] via 10.0.1.5, Ethernet2
```
<img width="596" height="103" alt="image" src="https://github.com/user-attachments/assets/ca39f740-72b3-4fad-8268-bb9f11e15078" /> <br>
По выводу команды, мы понимаем, что Leaf2/3 доступны через spine 1/2. <br>
## 1. Настройка
### 1.1. Настройка плоскости данных VxLAN
#### 1.1.1. Настроим loopback интерфейс 
VTEP, в отличие от MLAG VTEP, в EVPN все VTEP имеют уникальный IP-адрес. Позже мы увидим, чем отличаются отказоустойчивость и балансировка нагрузки в этой конфигурации. 
```
interface Loopback1
   description VTEP
   ip address 10.1.0.1/32
```
#### 1.1.2. Настроим интерфейс vxlan1, указав loopback1 в качестве источника
vxlan1 - это логический интерфейс, который будет обеспечивать функции инкапсуляции и декапсуляции заголовков VXLAN <br>
Команда (__vxlan source-interface Loopback1__):
```
interface Vxlan1
   vxlan source-interface Loopback1
```
### 1.2. Настройка службы EVPN Layer 2 на leaf1 коммутаторе
#### 1.2.1. Добавим локальные Vlan, с идентификаторами 112, 113
Команда:
```
vlan 112-113
```
#### 1.2.2. Сопоставим Vlan 2-го уровня c соответсвующими VNI
Команда:
```
interface Vxlan1
   vxlan vlan 112 vni 1112
   vxlan vlan 113 vni 1113
```
#### 1.2.3. Добавим конфигурацию mac-vrf EVPN для VLAN 112-113
Здесь мы настраиваем сервис на основе VLAN с использованием EVPN. Он состоит из двух компонентов. Первый — это __идентификатор маршрута (RD)__, который идентифицирует маршрутизатор (или коммутатор доступа), __инициирующий маршруты EVPN__. Его можно задать вручную в формате Номер:Номер, например, Loopback0:Идентификатор VLAN, или, как мы делаем в этом случае, позволить EOS автоматически выделить его.
<br><br>
Второй параметр — это __целевой маршрут (RT)__. RT используется коммутаторами доступа в сети для определения необходимости импорта объявленного маршрута в их локальные таблицы. Если они получают маршрут EVPN, они проверяют значение RT и смотрят, настроен ли у них соответствующий RT в BGP. Если да, то они импортируют маршрут в соответствующий mac-vrf (или VLAN). Если нет, то игнорируют маршрут.
<br><br>
Команды:
```
router bgp 65501
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
```
## 1.2. Настройка службы EVPN уровня 3 на Leaf1 коммутаторе
### 1.2.1. Создадим VRF, или логический экземпляр маршрутизации, для сети уровня 3 заказчика
Замечание: ``` В EOS по умолчанию VRF создаются с отключенной маршрутизацией между подсетями. Всегда включайте IP-маршрутизацию в определяемых пользователем VRF.```
Команды:
```
vrf instance vrf1
ip routing vrf vrf1
```
### 1.2.2. Создадим интерфейс SVI для Функции шлюза по умолчанию для хост-сети в качестве шлюза AnyCast Gateway
С помощью VXLAN мы можем использовать общий IP-адрес через Anycast Gateway. Это позволяет использовать один IP-адрес для всех коммутаторов без необходимости выделения дополнительных IP-адресов.
```
ip virtual-router mac-address 02:00:00:00:00:00
!
interface Vlan112
   description Host Network 112
   vrf vrf1
   ip address 192.168.112.1/24
   ip virtual-router address 192.168.112.254/24
!
interface Vlan113
   description Host Network 113
   vrf vrf1
   ip address 192.168.113.1/24
   ip virtual-router address 192.168.113.254/24
```
### 1.2.3. Произведем "Мапирование"(Соответсвие)  от англиского MAP, локального VRF Layer 3 с соответсвующим VNI 
__Ремарка:__ в данной работе я довольно сильно ощутил ту нелепость, и возможно даже новаторство и всю ту боль и трудность, писателей,  которую, вероятно, испытывали первые переводчики технических книг на русский. Возникает закономерный вопрос: стоит ли вообще переводить термины, или проще писать их «русскими буквами» (транслитом)? Прямой перевод часто звучит неестественно, режет слух и не несет той органичности, к которой мы привыкли в оригинале.  В общем это было такое маленькое лиическое отступление, которым, автор хотел поделиться с читателем.<br><br>

Для работы сервиса уровня 3 VRF требует наличия так называемого VNI уровня 3, который используется для маршрутизации VXLAN в симметричной конфигурации IRB между VTEP. Здесь подойдет любой уникальный идентификационный номер. <br>
Команда:
```
interface Vxlan1
    vxlan vrf vrf1 vni 666
```
### 1.2.4. Добавим конфигурацию IP VRF EVPN для VRF кастомера
Здесь мы настраиваем службу VRF уровня 3 с использованием EVPN. Она также использует уникальные значения __RD и RT__ . Они используются коммутаторами доступа для тех же целей, что и служба уровня 2. Разница заключается лишь в том, что маршруты импортируются. Если они получают __маршрут EVPN типа 5__, они проверяют значение RT и смотрят, настроен ли у них соответствующий RT для VRF. Если да, они импортируют маршрут в соответствующую таблицу маршрутизации VRF. Если нет, они игнорируют маршрут.<br>
Команды:
```
router bgp 65501
  rd auto
   !
   !
   vrf vrf1
      rd 65501:1
      route-target import evpn 1:666
      route-target export evpn 1:666
      redistribute connected
```
### 1.2.5. Настраиваем Port-Channel обращенный к хосту, а на русском это будет порт-канал EVPN, звучит на русском не привычно, и даже я бы сказал, звучит как новая технология
Здесь мы настраиваем идентификатор сегмента Ethernet ( ESI) , а также значение RT для сегмента Ethernet. Мы увидим, как плоскость управления EVPN использует их для согласования характеристик и состояния канала порта AA. Мы также настраиваем статический идентификатор системы LACP. Это необходимо для того, чтобы все члены сегмента Ethernet отображались как одна система LACP для нижестоящего устройства. Обратите внимание, что все эти значения должны совпадать для членов одного и того же сегмента Ethernet (или канала порта). <br>
Команды:
```
interface Port-Channel1
   description EVPN A-A DownLink S1-Host1-Eth7
   switchport trunk allowed vlan 112-113
   switchport mode trunk
     !
   interface Ethernet3
   description EVPN A-A Downlink -host
   switchport access vlan 112
   channel-group 1 mode active
```
## 1.2.6. Настроим аналогично leaf2
Пример настроек снизу:
```
```
## 2. Проверка 
Командой __show interfaces__ выполненной на leaf1, посмотрим состояние интерфейса Port-Channel1 <br>
```
Port-Channel1 is up, line protocol is up (connected)
  Hardware is Port-Channel, address is 5000.0001.0003
  Description: EVPN A-A DownLink S1-Host1-Eth7
   Active members in this channel: 1
  ... Ethernet3 , Full-duplex, 1Gb/s
  Up 4 days, 2 hours, 15 minutes, 3 seconds
  2 link status changes since last clear
```
Видим Channel 1 is UP - поднят(работает)!
### 2.1. Проверим bgp EVPN Leaf1
```
leaf1#sho bgp evpn
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65501
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.0.1:112 auto-discovery 0 0034:0000:0000:0000:0001
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.2:112 auto-discovery 0 0034:0000:0000:0000:0001
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:112 auto-discovery 0 0034:0000:0000:0000:0001
                                 10.1.0.2              -       100     0       65500 65502 i
 * >      RD: 10.1.0.1:1 auto-discovery 0034:0000:0000:0000:0001
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.2:1 auto-discovery 0034:0000:0000:0000:0001
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:1 auto-discovery 0034:0000:0000:0000:0001
                                 10.1.0.2              -       100     0       65500 65502 i
 * >      RD: 10.1.0.1:112 mac-ip 5000.001b.5e8d
                                 -                     -       -       0       i
 * >      RD: 10.1.0.1:112 mac-ip 5000.001b.5e8d 192.168.112.112
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.2:112 mac-ip 5000.001b.5e8d 192.168.112.112
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:112 mac-ip 5000.001b.5e8d 192.168.112.112
                                 10.1.0.2              -       100     0       65500 65502 i
 * >      RD: 10.1.0.1:112 imet 10.1.0.1
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.2:112 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:112 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65500 65502 i
 * >      RD: 10.1.0.1:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.1
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.2:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.2
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.2
                                 10.1.0.2              -       100     0       65500 65502 i
 * >      RD: 65501:1 ip-prefix 192.168.112.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 65502:1 ip-prefix 192.168.112.0/24
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 65502:1 ip-prefix 192.168.112.0/24
                                 10.1.0.2              -       100     0       65500 65502 i
 * >Ec    RD: 65503:1 ip-prefix 192.168.113.0/24
                                 10.1.0.3              -       100     0       65500 65503 i
 *  ec    RD: 65503:1 ip-prefix 192.168.113.0/24
                                 10.1.0.3              -       100     0       65500 65503 i
leaf1#
```
### 2.2. Проверим bgp EVPN Leaf2
```
leaf2#sho bgp evpn
BGP routing table information for VRF default
Router identifier 10.1.0.2, local AS number 65502
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.1.0.1:112 auto-discovery 0 0034:0000:0000:0000:0001
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:112 auto-discovery 0 0034:0000:0000:0000:0001
                                 10.1.0.1              -       100     0       65500 65501 i
 * >      RD: 10.1.0.2:112 auto-discovery 0 0034:0000:0000:0000:0001
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.1:1 auto-discovery 0034:0000:0000:0000:0001
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:1 auto-discovery 0034:0000:0000:0000:0001
                                 10.1.0.1              -       100     0       65500 65501 i
 * >      RD: 10.1.0.2:1 auto-discovery 0034:0000:0000:0000:0001
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.1:112 mac-ip 5000.001b.5e8d
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:112 mac-ip 5000.001b.5e8d
                                 10.1.0.1              -       100     0       65500 65501 i
 * >Ec    RD: 10.1.0.1:112 mac-ip 5000.001b.5e8d 192.168.112.112
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:112 mac-ip 5000.001b.5e8d 192.168.112.112
                                 10.1.0.1              -       100     0       65500 65501 i
 * >      RD: 10.1.0.2:112 mac-ip 5000.001b.5e8d 192.168.112.112
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.1:112 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:112 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65500 65501 i
 * >      RD: 10.1.0.2:112 imet 10.1.0.2
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.1:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.1
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.1
                                 10.1.0.1              -       100     0       65500 65501 i
 * >      RD: 10.1.0.2:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.2
                                 -                     -       -       0       i
 * >Ec    RD: 65501:1 ip-prefix 192.168.112.0/24
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 65501:1 ip-prefix 192.168.112.0/24
                                 10.1.0.1              -       100     0       65500 65501 i
 * >      RD: 65502:1 ip-prefix 192.168.112.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 65503:1 ip-prefix 192.168.113.0/24
                                 10.1.0.3              -       100     0       65500 65503 i
 *  ec    RD: 65503:1 ip-prefix 192.168.113.0/24
                                 10.1.0.3              -       100     0       65500 65503 i
leaf2#
```
### 2.3. Проверим bgp EVPN Leaf3
```
leaf3#sho bgp evpn
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65503
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.1.0.1:112 auto-discovery 0 0034:0000:0000:0000:0001
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:112 auto-discovery 0 0034:0000:0000:0000:0001
                                 10.1.0.1              -       100     0       65500 65501 i
 * >Ec    RD: 10.1.0.2:112 auto-discovery 0 0034:0000:0000:0000:0001
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:112 auto-discovery 0 0034:0000:0000:0000:0001
                                 10.1.0.2              -       100     0       65500 65502 i
 * >Ec    RD: 10.1.0.1:1 auto-discovery 0034:0000:0000:0000:0001
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:1 auto-discovery 0034:0000:0000:0000:0001
                                 10.1.0.1              -       100     0       65500 65501 i
 * >Ec    RD: 10.1.0.2:1 auto-discovery 0034:0000:0000:0000:0001
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:1 auto-discovery 0034:0000:0000:0000:0001
                                 10.1.0.2              -       100     0       65500 65502 i
 * >Ec    RD: 10.1.0.1:112 mac-ip 5000.001b.5e8d
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:112 mac-ip 5000.001b.5e8d
                                 10.1.0.1              -       100     0       65500 65501 i
 * >Ec    RD: 10.1.0.1:112 mac-ip 5000.001b.5e8d 192.168.112.112
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:112 mac-ip 5000.001b.5e8d 192.168.112.112
                                 10.1.0.1              -       100     0       65500 65501 i
 * >Ec    RD: 10.1.0.2:112 mac-ip 5000.001b.5e8d 192.168.112.112
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:112 mac-ip 5000.001b.5e8d 192.168.112.112
                                 10.1.0.2              -       100     0       65500 65502 i
 * >Ec    RD: 10.1.0.1:112 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:112 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65500 65501 i
 * >Ec    RD: 10.1.0.2:112 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:112 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65500 65502 i
 * >Ec    RD: 10.1.0.1:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.1
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.1
                                 10.1.0.1              -       100     0       65500 65501 i
 * >Ec    RD: 10.1.0.2:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.2
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.2
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
