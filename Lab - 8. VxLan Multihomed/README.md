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
Для корректной работы EVPN Multihoming (ESI) в топологии с разными AS (как у вас: 65501 и 65502) лифы должны обмениваться маршрутами __Type 4 (Ethernet Segment) и Type 1 (Auto-discovery)__.
Ниже шаблон конфигурации:<br>


### 1. Настройка Leaf1 (AS 65501)
```
bash
router bgp 65501
   ! Глобальная активация EVPN для соседей (Spine)
   address-family evpn
      neighbor SPINE-EVPN activate
   !
   vrf TENANT-1
      rd 65501:10100
      ! Важно: оба лифа должны импортировать RT друг друга или использовать общий
      route-target import evpn 65501:10100
      route-target import evpn 65502:10100  <-- Добавляем RT соседа
      route-target export evpn 65501:10100
```

### 2. Настройка Leaf2 (AS 65502)
   ```
bash
router bgp 65502
   address-family evpn
      neighbor SPINE-EVPN activate
   !
   vrf TENANT-1
      rd 65502:10100
      route-target import evpn 65502:10100
      route-target import evpn 65501:10100  <-- Добавляем RT соседа
      route-target export evpn 65502:10100
```

### 3. Проверка интерфейсов (ESI)
Убедитесь, что identifier и lacp system-id идентичны на обоих устройствах (у вас в конфиге они уже совпадают, это правильно):<br>
```
bash
interface Port-Channel10
   evpn ethernet-segment
      identifier 0034:0000:0000:0000:0001
   lacp system-id 1234.5678.0304

```
Почему это важно:<br>
В EVPN All-Active лифы выбирают Designated Forwarder (DF). Если лиф не видит соседа по BGP (пустой вывод show bgp evpn route-type ethernet-segment), он может:<br>
Считать себя единственным и блокировать трафик, чтобы не создать петлю.<br>
Не синхронизировать ARP-таблицу с соседом.<br><br>
Как проверить результат:<br>
После применения RT-импорта введите команду:<br>
show evpn ethernet-segment interface Port-Channel10<br>
В выводе вы должны увидеть:<br>
DF Role: Designated Forwarder (на одном) и Non-Designated Forwarder (на другом).<br>
Remote ES hosts: IP-адрес соседа (10.1.0.1 или 10.1.0.4).<br>




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
### Итог теста 4.1.
Переключение произошло мгновенно практически без потерь, что подтверждается снятым трейсом на конечной стороне.
### 4.2. Тестирование отказа BGP
И так, при отказе интерфейса на хосте, технология мультихоминг работает, в предыдущем тесте, мы это проверили, а давайте, попробуем положить BGP например на leaf1, давайте удалим BGP UNDERLAY пиры? Здорово я придумал?  и посмотрим, что произойдет? <br><br>
....................ПАРУ минут Спустя ............................. <br><br>
Вот собственно,  и ожидаемый результат, не работает в этом случае мультихоминг. К сожалению, я и так потратил довольно много времени на разбор данной лабы, и проверять поведение в конфигурации с горизонтальными линками (MLAG) я  не буду. <br>
<img width="718" height="383" alt="image" src="https://github.com/user-attachments/assets/bfeffb6e-8b8a-4846-90bb-e5976c08c631" /><br>
## 5 Итог и выводы
В ходе выполнения лабораторной работы мы успешно настроили и проверили отказоустойчивое подключение клиентов с использованием технологии EVPN Multihoming в фабрике VXLAN (*_кроме случая с падением BGP_). На основе проделанной работы можно сделать следующие ключевые выводы:

## 5.1. Принципиальные отличия от MLAG
EVPN Multihoming принципиально отличается от традиционного MLAG (MC-LAG) подходом к организации отказоустойчивости:
<br><br>
Уникальные VTEP IP-адреса — в отличие от MLAG, где два лифа маскируются под один VTEP с общим IP-адресом, в EVPN Multihoming каждый лиф имеет собственный уникальный IP-адрес источника VXLAN-туннеля.
<br><br>
Независимая работа — лифы не синхронизируют таблицы коммутации и маршрутизации через проприетарные протоколы (как MLAG), а полагаются на плоскость управления EVPN.
<br><br>
ECMP-балансировка — уникальные IP-адреса VTEP позволяют использовать ECMP для балансировки трафика между лифами как в восходящем, так и в нисходящем направлении.

## 5.2. Роль Ethernet Segment (ES) и Designated Forwarder (DF)
Ключевым элементом технологии является концепция сегмента Ethernet (ES):
<br><br>
__ESI (Ethernet Segment Identifier)__ — уникальный идентификатор, который связывает два лифа (leaf1 и leaf2) в единый логический сегмент для подключения нижестоящего устройства.
<br><br>
__Режим all-active__ — в нашей конфигурации использовался режим all-active, при котором оба лифа могут передавать трафик от хоста, что обеспечивает акти<br><br>вное использование обоих uplink-каналов.

Выборы DF — протокол DF (Designated Forwarder) автоматически определяет, какой из лифов отвечает за пересылку BUM-трафика (Broadcast, Unknown unicast, Multicast) в сегмент Ethernet. В нашем случае DF был выбран leaf1 (Designated forwarder: 10.1.0.1), а leaf2 находится в режиме Non-Designated forwarder для BUM-трафика.

## 5.3. Распределение служебных функций в плоскости управления EVPN
В ходе проверок мы наблюдали, как плоскость управления EVPN обеспечивает работу мультихоминга:
<br><br>
__Маршруты Type-1 (Auto-Discovery)__ — информируют удаленные VTEP о членах одного Ethernet Segment, создавая основу для резервирования.
<br><br>
__Маршруты Type-2 (MAC/IP)__ — содержат информацию о MAC-адресах и IP-адресах хостов, привязанную к конкретному ESI, что позволяет удаленным лифам видеть оба пути к хосту.
<br><br>
__Маршруты Type-3 (IMET)__ — обеспечивают доставку BUM-трафика между всеми участниками VXLAN-сегмента.
<br><br>
__Маршруты Type-4 (Ethernet Segment)__ — используются для выборов DF и синхронизации состояния сегмента между его участниками.
<br><br>
__Маршруты Type-5 (IP Prefix)__ — обеспечивают маршрутизацию между разными VNI на L3-уровне.
<br><br>
## 5.4. Балансировка нагрузки и отказоустойчивость
Проверка подтвердила, что:
<br><br>
Трафик к хосту за leaf1/leaf2 балансируется между двумя VTEP (10.1.0.1 и 10.1.0.2), о чем свидетельствуют записи в таблице маршрутизации leaf3: via VTEP 10.1.0.1 и via VTEP 10.1.0.2 для одного и того же префикса.
<br><br>
При отказе одного из лифов (например, leaf1) трафик автоматически перенаправляется через оставшийся лиф (leaf2) без необходимости сходимости протоколов маршрутизации на хосте — это обеспечивается механизмами EVPN и LACP.
<br><br>
Таблица MAC-адресов на leaf3 содержит записи для маршрутизаторов leaf1 и leaf2, что подтверждает возможность достижения хоста через оба пути.
<br><br>
## 5.5. Симметричная маршрутизация IRB с использованием L3 VNI
Настройка L3 VNI (в нашем случае VNI 666) для VRF vrf1 обеспечивает:
<br><br>
Возможность маршрутизации между разными VLAN (112 и 113) на любом лифе.
<br><br>
Симметричную модель IRB (Integrated Routing and Bridging), при которой как входящий, так и исходящий трафик проходит через VXLAN-инкапсуляцию с использованием единого L3 VNI.
<br><br>
Независимость расположения шлюза — благодаря Anycast Gateway с виртуальным MAC-адресом хосты всегда имеют один и тот же адрес шлюза независимо от того, через какой лиф они подключаются.
<br><br>
## 5.6. Практическая значимость
EVPN Multihoming представляет собой современное решение для построения отказоустойчивых сетей доступа, которое:
<br><br>
Устраняет необходимость в сложных проприетарных решениях типа MLAG.
<br><br>
Обеспечивает эффективное использование полосы пропускания за счет активного режима работы всех линков.
<br><br>
Масштабируется на большое количество лифов без увеличения сложности конфигурации.
<br><br>
Использует стандартизированные протоколы (BGP EVPN), что гарантирует совместимость оборудования разных вендоров.
<br><br>
Таким образом, цель лабораторной работы достигнута: мы подтвердили работоспособность отказоустойчивого подключения клиентов с использованием EVPN Multihoming и продемонстрировали ключевые механизмы, обеспечивающие его функционирование.
<br><br>
## 6. Конфигурация устройств
### 6.1. Spine
```
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
__Spine2__
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
### 6.3. Leaf
__leaf1__
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
vlan 112
!
vrf instance vrf1
!
interface Port-Channel1
   description EVPN A-A DownLink S1-Host1-Eth7
   switchport trunk allowed vlan 112-113
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0034:0000:0000:0000:0001
      route-target import 00:03:04:00:00:05
   lacp system-id 1234.5678.0304
!
interface Ethernet1
   description Peer-to-peer link to Spine-1
   no switchport
   ip address 10.0.1.0/31
!
interface Ethernet1/3
!
interface Ethernet2
   description Peer-to-peer link to Spine-2
   no switchport
   ip address 10.0.1.4/31
!
interface Ethernet3
   description EVPN A-A Downlink -host
   switchport trunk allowed vlan 112-113
   channel-group 1 mode active
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
   description Host Network 112
   vrf vrf1
   ip address 192.168.112.1/24
   ip virtual-router address 192.168.112.254/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 112 vni 1112
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
   rd auto
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
!
end
leaf1#
```
__Leaf2__
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
vrf instance vrf1
!
interface Port-Channel1
   description EVPN A-A DownLink S1-Host1-Eth8
   switchport trunk allowed vlan 112-113
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0034:0000:0000:0000:0001
      route-target import 00:03:04:00:00:05
   lacp system-id 1234.5678.0304
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
   description EVPN A-A DownLink S1-host
   switchport trunk allowed vlan 112-113
   channel-group 1 mode active
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
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 112 vni 1112
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
   rd auto
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
   vrf vrf1
      rd 65502:1
      route-target import evpn 1:666
      route-target export 1:666
      redistribute connected
!
end
leaf2#
```
__Leaf3__

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
vrf instance vrf1
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
!
interface Loopback1
   description VTEP
   ip address 10.1.0.3/32
!
interface Management1
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
   vxlan vlan 112 vni 1112
   vxlan vrf vrf1 vni 666
   vxlan learn-restrict any
!
ip virtual-router mac-address 02:00:00:00:00:00
!
ip routing
ip routing vrf vrf1
!
router bgp 65503
   router-id 10.1.0.3
   no bgp default ipv4-unicast
   timers bgp 3 9
   rd auto
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
   vrf vrf1
      rd 65503:1
      route-target import evpn 1:666
      route-target export evpn 1:666
      redistribute connected
!
end
leaf3#
```
### 6.4. Host level
```
S1-Host1#sho run
! Command: show running-config
! device: S1-Host1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname S1-Host1
!
spanning-tree mode mstp
!
vlan 2,112
!
interface Port-Channel1
   switchport trunk allowed vlan 112-113
   switchport mode trunk
!
interface Ethernet1
!
interface Ethernet2
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
   switchport access vlan 112
   switchport mode trunk
   channel-group 1 mode active
!
interface Ethernet8
   switchport mode trunk
   channel-group 1 mode active
!
interface Management1
!
interface Vlan112
   ip address 192.168.112.112/24
!
interface Vlan113
   ip address 192.168.113.113/24
!
ip routing
!
ip route 0.0.0.0/0 192.168.112.254
!
end
S1-Host1#
```





