# VXLAN. Multihoming

## Цель:
Настроить отказоустойчивое подключение клиентов с использованием EVPN Multihoming <br>
Схема стенда:
<img width="1166" height="871" alt="image" src="https://github.com/user-attachments/assets/8f848c80-219a-4329-be8d-47a0cf2d40fe" />
 <br>
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
### 2.4. Определим, кто является назначенным пересыльщиком (Designated Forwarder, DF) для канала EVPN AA Port-Channel 
В сегменте Ethernet EVPN AA только один участник ES выбирается в качестве назначенного пересыльщика (DF). DF отвечает за пересылку трафика BUM подключенному нижестоящему устройству. По умолчанию все участники ES выполняют операцию модуля для равномерного выбора DF на основе полученных маршрутов сегмента Ethernet (EVPN Type-4). Ниже показаны полученные маршруты EVPN Type-4.
__leaf1__
```
leaf1#show bgp evpn route-type ethernet-segment
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65501
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.0.1:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.1
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.2:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.2
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.2
                                 10.1.0.2              -       100     0       65500 65502 i
leaf1#
```
__leaf2__
```
leaf2#show bgp evpn route-type ethernet-segment
BGP routing table information for VRF default
Router identifier 10.1.0.2, local AS number 65502
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.1.0.1:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.1
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.1
                                 10.1.0.1              -       100     0       65500 65501 i
 * >      RD: 10.1.0.2:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.2
                                 -                     -       -       0       i
```
__leaf3__
```
leaf3#show bgp evpn route-type ethernet-segment
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65503
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.1.0.1:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.1
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.1
                                 10.1.0.1              -       100     0       65500 65501 i
 * >Ec    RD: 10.1.0.2:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.2
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:1 ethernet-segment 0034:0000:0000:0000:0001 10.1.0.2
                                 10.1.0.2              -       100     0       65500 65502 i
leaf3#
```
__show bgp evpn route-type ethernet-segment esi 0034:0000:0000:0000:0001 detail__
```
leaf2#
ail bgp evpn route-type ethernet-segment esi 0034:0000:0000:0000:0001 det
BGP routing table information for VRF default
Router identifier 10.1.0.2, local AS number 65502
BGP routing table entry for ethernet-segment 0034:0000:0000:0000:0001 10.1.0.1, Route Distinguisher: 10.1.0.1:1
 Paths: 2 available
  65500 65501
    10.1.0.1 from 10.2.2.2 (10.2.2.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:03:04:00:00:05
  65500 65501
    10.1.0.1 from 10.1.1.1 (10.1.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:03:04:00:00:05
BGP routing table entry for ethernet-segment 0034:0000:0000:0000:0001 10.1.0.2, Route Distinguisher: 10.1.0.2:1
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:03:04:00:00:05
```
__На стороне Leaf3__
```
leaf3#show bgp evpn route-type ethernet-segment esi 0034:0000:0000:0000:0001 det
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65503
BGP routing table entry for ethernet-segment 0034:0000:0000:0000:0001 10.1.0.1, Route Distinguisher: 10.1.0.1:1
 Paths: 2 available
  65500 65501
    10.1.0.1 from 10.2.2.2 (10.2.2.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:03:04:00:00:05
  65500 65501
    10.1.0.1 from 10.1.1.1 (10.1.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:03:04:00:00:05
BGP routing table entry for ethernet-segment 0034:0000:0000:0000:0001 10.1.0.2, Route Distinguisher: 10.1.0.2:1
 Paths: 2 available
  65500 65502
    10.1.0.2 from 10.2.2.2 (10.2.2.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:03:04:00:00:05
  65500 65502
    10.1.0.2 from 10.1.1.1 (10.1.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:03:04:00:00:05
leaf3#
```
__На leaf1__
```
leaf1#
leaf1#show bgp evpn route-type ethernet-segment esi 0034:0000:0000:0000:0001 det
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65501
BGP routing table entry for ethernet-segment 0034:0000:0000:0000:0001 10.1.0.1, Route Distinguisher: 10.1.0.1:1
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:03:04:00:00:05
BGP routing table entry for ethernet-segment 0034:0000:0000:0000:0001 10.1.0.2, Route Distinguisher: 10.1.0.2:1
 Paths: 2 available
  65500 65502
    10.1.0.2 from 10.2.2.2 (10.2.2.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:03:04:00:00:05
  65500 65502
    10.1.0.2 from 10.1.1.1 (10.1.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:03:04:00:00:05
leaf1#
```
### sho bgp  evpn instance
__leaf1__
```
leaf1#sho bgp  evpn instance
EVPN instance: VLAN 112
  Route distinguisher: 0:0
  Route target import: Route-Target-AS:65500:1112
  Route target export: Route-Target-AS:65500:1112
  Service interface: VLAN-based
  Local VXLAN IP address: 10.1.0.1
  VXLAN: enabled
  MPLS: disabled
  Local ethernet segment:
    ESI: 0034:0000:0000:0000:0001
      Interface: Port-Channel1
      Mode: all-active
      State: up
      ES-Import RT: 00:03:04:00:00:05
      DF election algorithm: modulus
      Designated forwarder: 10.1.0.1
      Non-Designated forwarder: 10.1.0.2
leaf1#
```
__Leaf2__
```
leaf2#sho bgp  evpn instance
EVPN instance: VLAN 112
  Route distinguisher: 0:0
  Route target import: Route-Target-AS:65500:1112
  Route target export: Route-Target-AS:65500:1112
  Service interface: VLAN-based
  Local VXLAN IP address: 10.1.0.2
  VXLAN: enabled
  MPLS: disabled
  Local ethernet segment:
    ESI: 0034:0000:0000:0000:0001
      Interface: Port-Channel1
      Mode: all-active
      State: up
      ES-Import RT: 00:03:04:00:00:05
      DF election algorithm: modulus
      Designated forwarder: 10.1.0.1
      Non-Designated forwarder: 10.1.0.2
leaf2#
```
__Leaf3__
```
leaf3#sho bgp  evpn instance
EVPN instance: VLAN 113
  Route distinguisher: 0:0
  Route target import: Route-Target-AS:65500:1113
  Route target export: Route-Target-AS:65500:1113
  Service interface: VLAN-based
  Local VXLAN IP address: 10.1.0.3
  VXLAN: enabled
  MPLS: disabled
leaf3#
```
## 2.4. На leaf3 проверем таблицу IMET, чтобы убедиться, что leaf1 и leaf2 обнаружены в оверлейной сети. 
Маршрут __Inclusive Multicast Ethernet Tag (IMET)__ — это способ, которым VTEP объявляет о своем участии в данной службе Layer 2 или сегменте VXLAN. Он также известен как __маршрут EVPN Type-3__. Другие коммутаторы Leaf получают этот маршрут, оценивают __RT__, чтобы проверить, есть ли у них соответствующая конфигурация, и, если есть, импортируют объявляющий VTEP в свой список рассылки для трафика BUM. Обратите внимание, что __это делается для каждой VLAN отдельно__.…
Комманда: __show bgp evpn route-type imet__
```
leaf3#show bgp evpn route-type imet
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65503
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.1.0.1:112 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65500 65501 i
 *  ec    RD: 10.1.0.1:112 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65500 65501 i
 * >Ec    RD: 10.1.0.2:112 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:112 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65500 65502 i
leaf3#
```
## 2.4.1. Посмотрим детальнее IMET (Inclusive Multicast Ethernet Tag)
Команда: show bgp evpn route-type imet rd 10.1.0.1:112 detail
```
leaf3#show bgp evpn route-type imet rd 10.1.0.1:112 detail
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65503
BGP routing table entry for imet 10.1.0.1, Route Distinguisher: 10.1.0.1:112
 Paths: 2 available
  65500 65501
    10.1.0.1 from 10.2.2.2 (10.2.2.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan
      VNI: 1112
      PMSI Tunnel: Ingress Replication, MPLS Label: 1112, Leaf Information Required: false, Tunnel ID: 10.1.0.1
  65500 65501
    10.1.0.1 from 10.1.1.1 (10.1.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan
      VNI: 1112
      PMSI Tunnel: Ingress Replication, MPLS Label: 1112, Leaf Information Required: false, Tunnel ID: 10.1.0.1
leaf3#
```
 Это (replication flood vtep list is:   112 10.1.0.2)  можно узнать из команды ниже:
 ```  
leaf1# sho interfaces vxlan 1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback1 and is active with 10.1.0.1
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
  Static VRF to VNI mapping is
   [vrf1, 666]
  Headend replication flood vtep list is:
   112 10.1.0.2
  Shared Router MAC is 0000.0000.0000
  ```
## 2.5 На leaf 3 проверим плоскость управления EVPN на наличие соответствующего MAC/IP хоста. 

Мы видим MAC s1-host1 __(5000.001b.5e8d)__ несколько раз в плоскости управления из-за нашей резервной архитектуры MLAG и ECMP. Оба leaf1 и leaf2 имеют подключения от s1-host1 в VLAN 112, следовательно, будут генерировать эти __маршруты EVPN типа 2__ для своего MAC __после обнаружения хоста__. Затем каждый из них отправляет этот маршрут на резервные магистральные коммутаторы (или серверы маршрутизации EVPN), которые обеспечивают путь ECMP к хосту.
<br><br>
Команда: __show bgp evpn route-type mac-ip__
Для наглядности решил оставить вывод команды для leaf1/2 и посмотрите на leaf3, куда приходят резервные маршруты от leaf1/2
<br>
__Leaf1__
  ```
leaf1#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65501
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.0.1:112 mac-ip 5000.001b.5e8d
                                 -                     -       -       0       i
 * >      RD: 10.1.0.1:112 mac-ip 5000.001b.5e8d 192.168.112.112
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.2:112 mac-ip 5000.001b.5e8d 192.168.112.112
                                 10.1.0.2              -       100     0       65500 65502 i
 *  ec    RD: 10.1.0.2:112 mac-ip 5000.001b.5e8d 192.168.112.112
                                 10.1.0.2              -       100     0       65500 65502 i
leaf1#
 ```
__Leaf2__
 ```
leaf2#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.1.0.2, local AS number 65502
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
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
leaf2#
 ```
Соответсвенно на Leaf3 мы видим резервные маршруты
 ```
leaf3#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65503
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
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
leaf3#
 ```
## 2.5.1. Посмотрим более детально 
 ```
leaf3#show bgp evpn route-type mac-ip  vni 1112 detail
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65503
BGP routing table entry for mac-ip 5000.001b.5e8d, Route Distinguisher: 10.1.0.1:112
 Paths: 2 available
  65500 65501
    10.1.0.1 from 10.2.2.2 (10.2.2.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan
      VNI: 1112 ESI: 0034:0000:0000:0000:0001
  65500 65501
    10.1.0.1 from 10.1.1.1 (10.1.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan
      VNI: 1112 ESI: 0034:0000:0000:0000:0001
BGP routing table entry for mac-ip 5000.001b.5e8d 192.168.112.112, Route Distinguisher: 10.1.0.1:112
 Paths: 2 available
  65500 65501
    10.1.0.1 from 10.2.2.2 (10.2.2.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:666 Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:d7:ee:0b EvpnNdFlags:pflag
      VNI: 1112 L3 VNI: 666 ESI: 0034:0000:0000:0000:0001
  65500 65501
    10.1.0.1 from 10.1.1.1 (10.1.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:1:666 Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:d7:ee:0b EvpnNdFlags:pflag
      VNI: 1112 L3 VNI: 666 ESI: 0034:0000:0000:0000:0001
BGP routing table entry for mac-ip 5000.001b.5e8d 192.168.112.112, Route Distinguisher: 10.1.0.2:112
 Paths: 2 available
  65500 65502
    10.1.0.2 from 10.2.2.2 (10.2.2.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:666 Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:cb:38:c2
      VNI: 1112 L3 VNI: 666 ESI: 0034:0000:0000:0000:0001
  65500 65502
    10.1.0.2 from 10.1.1.1 (10.1.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:1:666 Route-Target-AS:65500:1112 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:cb:38:c2
      VNI: 1112 L3 VNI: 666 ESI: 0034:0000:0000:0000:0001
leaf3#
 ```
## 2.6. На leaf3 проверим плоскость управления EVPN на наличие сигналов EVPN AA связанных с s1-host1.
Выше мы видели, что __маршруты типа 2 содержали значение ESI__. Затем мы можем определить все VTEP, являющиеся членами этого __ES, проверив маршруты автоматического обнаружения, или маршруты EVPN типа 1__.<br><br>
Команда:      show bgp evpn route-type auto-discovery <br><br>
```
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
leaf3#
```
## 2.7. На leaf1 проверим таблицу BGP, чтобы убедиться, что сети "арендаторов/хостов" на leaf3 были изучены в оверлейной сети.
В приведенном ниже выводе показаны изученные маршруты IP-префиксов из EVPN. Они называются __маршрутами EVPN типа 5__. Аналогично маршрутам __типа 2 и типа 3__, __другие VTEP оценивают RT, чтобы проверить наличие соответствующей конфигурации, и, если она есть, импортируют содержащийся префикс в свою таблицу маршрутизации VRF__. <br><br>

В подробном выводе мы можем увидеть конкретные маршруты от leaf3,  Мы видим информацию о RT, MAC-адресе маршрутизатора EVPN. Ниже представлен основной вывод, касающийся сетИ 192.168.112.0/24
```
leaf3#show bgp evpn route-type ip-prefix ipv4
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
leaf3#
```
Мы видим информацию о RT, MAC-адресе маршрутизатора EVPN (совместно используемом с leaf3) и VNI уровня L3
```
leaf3#show bgp evpn route-type ip-prefix ipv4 detail
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65503
BGP routing table entry for ip-prefix 192.168.112.0/24, Route Distinguisher: 65501:1
 Paths: 2 available
  65500 65501
    10.1.0.1 from 10.2.2.2 (10.2.2.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:666 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:d7:ee:0b
      VNI: 666
  65500 65501
    10.1.0.1 from 10.1.1.1 (10.1.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:1:666 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:d7:ee:0b
      VNI: 666
BGP routing table entry for ip-prefix 192.168.112.0/24, Route Distinguisher: 65502:1
 Paths: 2 available
  65500 65502
    10.1.0.2 from 10.2.2.2 (10.2.2.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:666 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:cb:38:c2
      VNI: 666
  65500 65502
    10.1.0.2 from 10.1.1.1 (10.1.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:1:666 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:cb:38:c2
      VNI: 666
BGP routing table entry for ip-prefix 192.168.113.0/24, Route Distinguisher: 65503:1
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best, redistributed (Connected)
      Extended Community: Route-Target-AS:1:666 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:72:8b:31
      VNI: 666
```
## 2.8. Проверка локальных ARP/MAC
```
leaf2#sho mac address-table dynamic
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
 112    5000.001b.5e8d    DYNAMIC     Po1        1       1:32:49 ago
4094    5000.0072.8b31    DYNAMIC     Vx1        1       1 day, 14:09:25 ago
4094    5000.00d7.ee0b    DYNAMIC     Vx1        1       5 days, 12:22:13 ago
Total Mac Addresses for this criterion: 3

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
```
__На Leaf3__
```
leaf3#sho mac address-table dynamic
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
4094    5000.00cb.38c2    DYNAMIC     Vx1        1       4 days, 7:20:48 ago
4094    5000.00d7.ee0b    DYNAMIC     Vx1        1       4 days, 7:20:48 ago
Total Mac Addresses for this criterion: 2
```
## 2.9. На leaf1 проверим плоскость данных VXLAN на наличие MAC-адреса
Напомним, что выше маршрут EVPN типа 2 для  был связан с ESI, и наши маршруты EVPN типа 1 показали, что leaf1 и leaf2 являются членами этого ES. Таким образом, мы видим два возможных пункта назначения для этого MAC-адреса хоста. Затем команда show l2rib output mac позволяет нам увидеть информацию о VTEP в оборудовании, показывающую, какая балансировка нагрузки будет происходить. Наконец, мы можем проверить путь ECMP к удаленному VTEP leaf3 через Spine 1 и Spine2.
```
leaf1#
how vxlan address-table evpn
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
4094  5000.0072.8b31  EVPN      Vx1  10.1.0.3         1       1 day, 14:12:46 ago
4094  5000.00cb.38c2  EVPN      Vx1  10.1.0.2         1       5 days, 12:25:35 ago
Total Remote Mac Addresses for this criterion: 2

leaf2#
how vxlan address-table evpn
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
 112  5000.001b.5e8d  EVPN      Vx1  0.0.0.0          1       1:36:28 ago
4094  5000.0072.8b31  EVPN      Vx1  10.1.0.3         1       1 day, 14:13:03 ago
4094  5000.00d7.ee0b  EVPN      Vx1  10.1.0.1         1       5 days, 12:25:52 ago
Total Remote Mac Addresses for this criterion: 3

leaf3#
how vxlan address-table evpn
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
4094  5000.00cb.38c2  EVPN      Vx1  10.1.0.2         1       4 days, 7:25:04 ago
4094  5000.00d7.ee0b  EVPN      Vx1  10.1.0.1         1       4 days, 7:25:04 ago
Total Remote Mac Addresses for this criterion: 2
```
## 2.10 Просмотр Ip route
__leaf1__
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
 B E      192.168.113.0/24 [200/0] via VTEP 10.1.0.3 VNI 666 router-mac 50:00:00:72:8b:31 local-interface Vxlan1

```
__leaf3__
```
 leaf3#
leaf3#sho ip route vrf vrf1

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

 B E      192.168.112.112/32 [200/0] via VTEP 10.1.0.2 VNI 666 router-mac 50:00:00:cb:38:c2 local-interface Vxlan1
                                     via VTEP 10.1.0.1 VNI 666 router-mac 50:00:00:d7:ee:0b local-interface Vxlan1
 B E      192.168.112.0/24 [200/0] via VTEP 10.1.0.2 VNI 666 router-mac 50:00:00:cb:38:c2 local-interface Vxlan1
                                   via VTEP 10.1.0.1 VNI 666 router-mac 50:00:00:d7:ee:0b local-interface Vxlan1
 C        192.168.113.0/24 is directly connected, Vlan113

leaf3#
```
## 3. Просмотр информации Wireshark
При выполнении команды ping c S1-host1, leaf1 послал в сторону Spine 1 и Spine2, BGP UPDATE MEssage Route type 2 (MAC Advertisement), пример ниже:
<img width="1118" height="1782" alt="image" src="https://github.com/user-attachments/assets/1b2c4eb4-4fbc-4e5e-bde3-c9819efe3414" />
<br>
## 3.1. Просмотр где какие Mac адреса отображаются при передачи трафика
<img width="1608" height="449" alt="image" src="https://github.com/user-attachments/assets/63d25382-62bc-48e5-b94e-2fb7adfe4ec2" /> <br>
На рисунке выше представлен пример пинга выполненного с хоста S1-host1 до pc3. Трейс снимался на leaf2. Прошу обратить внимание какие MAC адреса присутвуют на Underlay и Overlay уровнях.<br> <br>
нА РИСУНКЕ ниже представлен Replay ответ от 113 до 112. Прошу обратить внимание на мак адреса и самостоятельно проанализировать где и на что меняются мак адреса.<br>
<img width="1599" height="422" alt="image" src="https://github.com/user-attachments/assets/6258f5dd-0b26-4bd5-81af-5c9bbf5b7b4f" />
## 4. Проверка работы Multihoming
### 4.1. Проверка имитирующая отказ одного интерфейса хоста
И Так с хоста pc3 (192.168.113.113) я запустил беЗконечный (проверочные слова __БеЗ конца__ Ибо __Бес конца__ не этично и не правильно) пинг хоста S1-Host1 (192.168.112.112). Методом просмотра трейсов, выяснил путь прохождения трафика. Тафик шел, через leaf2, на S1-host1 трафик приходил на интерфейс eth8. Далее на хосте отключаем Eth8, имитируя тем самым отказ порта хоста, смотрим что получилось:<br>
<img width="1576" height="1644" alt="image" src="https://github.com/user-attachments/assets/32b78277-d65d-4321-93aa-b437ba21aec1" />
<br>
Как мы видим, leaf2, не получив ICMP Replay пакета, понимает, что что-то тут не ладно, и отправляет в сторону Спайнов BGP Update Withdrawn сообщение <br>
Далее, мы видим от Spine 1, на leaf2 приходит BGP UPdate, с указанием какой IP адрес использовать <br>
<img width="1580" height="1368" alt="image" src="https://github.com/user-attachments/assets/04dd95c8-93a3-4297-9e15-88b1e3bce312" /> <br>
Далее, для MAC адреса хоста S1-host1, leaf2 отправляет в сторону spine1, MAC Advertisement:<br>
<img width="1134" height="1348" alt="image" src="https://github.com/user-attachments/assets/a32561ff-af22-40ad-8848-a4ef4c6ea281" /> <br>
Давайте взглянем на это с другой стороны, что происходило на стороне Leaf3, в момент когда отключили порт на хосте 1? Со спайна 1 прилетел BGP Update Withdrawn Meaasage. <br>
Каритинка представлена снизу:<br>
<img width="1668" height="1090" alt="image" src="https://github.com/user-attachments/assets/5af72854-f6a9-4a20-a6da-4bfe70edf68c" />
<br>
Далее, лиф 3 и спайн 1 еще обменивались сообщениями, я их не стал приводить, вот крайнее сообщение, после которого в пакетах пинга, поменялись IP адреса назначения, вместа лиф2 был указан лиф1. <br>
И так BGP Withdrawn от спайн 1<br>
<img width="1150" height="1356" alt="image" src="https://github.com/user-attachments/assets/61692e4a-8a8d-4647-b82f-903fd4759c79" /> <br>









