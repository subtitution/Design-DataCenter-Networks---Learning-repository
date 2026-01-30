# Настройка BGP для Underlay сети <br>
 Задание: <br>
1. Настроите BGP в Underlay сети, для IP связанности между всеми сетевыми устройствами. iBGP или eBGP - решать вам!<br>
2. Зафиксируете в документации - план работы, адресное пространство, схему сети, конфигурацию устройств<br>
3. Убедитесь в наличии IP связанности между устройствами в BGP домене<br>
## План работ
- настроить iBGP
- посмотреть процесс установления BGP сессии
- убедиться в IP связности, попробовать отключить один линк
- настроить eBGP
- убедиться в ip связности
<br><br>
## 1. Начало <br>
<img width="527" height="326" alt="laba" src="https://github.com/user-attachments/assets/a7e00c82-9960-4a15-851a-bd3a9f01a8b0" />
<br>
Сверху, представлена схема используемой сети. <br>
<br>

Начну повествование к данной работе в стиле Квентина Тарантина, с середины... <br>
Предисловие: bgp настроен на всех Leaf и Spine. <br>
Давайте внимательно посмотрим на вывод команды: _show ip bgp_  которую выполним на leaf1<br>
<img width="1247" height="383" alt="image" src="https://github.com/user-attachments/assets/f8e22d69-58d8-4e73-8279-c060e094243f" />
<br>
Итак, мы видим IP Loopback адресов Leaf2,3, но __Next hop__ очевидно не похож на адреса spine 1 и spine 2. <br>
И правильно, мой дорогой, внимательный читатель, все Верно, на Spine коммутаторах я забыл ввести вот такю вот команду: _neighbor UNDERLAY next-hop-self_ <br>
И добавляем на spine 1,2 в настройки bgp команду: _neighbor UNDERLAY next-hop-self_ <br>
После возвращаемся на leaf1, и проверяемб что поменялось: <br>
<img width="1235" height="449" alt="image" src="https://github.com/user-attachments/assets/ef2d9cf8-c54a-489f-8227-906895829f4a" /><br>
Видно ТЕПЕРЬ видно, что адреса __Next-hop__ поменялись, на адреса Spine-ов, у Spine и leaf есть связь, через пиринговые каналы, теперь маршрутизация будет работать. <br> <br>
Так же прошу обратить внимание, что нам доступны loopback адрес 10.0.0.2 через spine 1 (10.1.1.1) и Spine 2 (10.2.2.1) - эта информация отображается в Cluster-List в колонке Path. Но loopback leaf 3 (10.0.0.3) доступен только через Spine 2. В этом нет ничего страшного, т.к. после предыдущей лабы, я забыл включить ethernet интерфейс на Leaf-3, идущий в сторону spine1. <br>
Конечно же проверим IP связность между loopback адресами, связь есть, ниже пример: <br>
<img width="983" height="1116" alt="image" src="https://github.com/user-attachments/assets/da3d6896-f47e-40aa-8a27-7a80780764dc" />
<br>
## Часть 2. Пример конфигов <br>
### 2.1. Конфигурация Spine1 <br> <br>
hostname spine1<br>
!<br>
spanning-tree mode mstp<br>
!<br>
interface Ethernet1<br>
   description Peer-to-peer link to leaf-1<br>
   no switchport<br>
   ip address 10.0.1.1/31<br>
!<br>
interface Ethernet2<br>
   description Peer-to-peer link to leaf-2<br>
   no switchport<br>
   ip address 10.0.2.1/31<br>
!<br>
interface Ethernet3<br>
   description Peer-to-peer link to leaf-3<br>
   no switchport<br>
   ip address 10.0.3.1/31<br>
!<br>
  interface Loopback1<br>
   description IP for underlay -Router-ID<br>
   ip address 10.1.1.1/32<br>
!<br>
interface Loopback2<br>
   description IP for overlay layer<br>
   ip address 10.1.1.2/32<br>
!<br>
ip routing<br>
!<br>
__router bgp 65500__<br>
   router-id 10.1.1.1<br>
   no bgp default ipv4-unicast<br>
   timers bgp 3 9<br>
   neighbor UNDERLAY peer group<br>
   neighbor UNDERLAY remote-as 65500<br>
   neighbor UNDERLAY next-hop-self<br>
   neighbor UNDERLAY route-reflector-client<br>
   __neighbor 10.0.1.0 peer group UNDERLAY__<br>
   __neighbor 10.0.2.0 peer group UNDERLAY__<br>
   __neighbor 10.0.3.0 peer group UNDERLAY__<br>
   !<br>
   address-family ipv4<br>
      neighbor UNDERLAY activate<br>
      network 10.1.1.1/32<br>
!<br>
<br>
В данном примере в настройках соседей в bgp, мы используем команду __neighbor IP адрес LoopBack leaf-ов__, в данном случае это рабочий вариант и е критично, т.к. кол-во leaf-ов всего 3. Но на практике, когда используется большее кол-во leaf-ов, лучше сразу использовать другую команду, а именно: __bgp listen range 10.0.0.0/16 peer-group UNDERLAY remote-as 65500__ , где 10.0.0.0/16 диапозон используемых адресов для loopback leaf-ов. Далее ниже по тексту, мы поменяем настройки.<br>
Пример вывода команд bgp для spine1, представлен ниже: <br>
<img width="1084" height="590" alt="image" src="https://github.com/user-attachments/assets/79dcb947-914d-4165-b8cf-28a7626b6621" />
<br>
### 2.2. Конфигурация Spine2 <br> <br>
hostname spine2<br>
!<br>
spanning-tree mode mstp<br>
!<br>
__interface Ethernet1__<br>
   description Peer-to-peer link to leaf-1<br>
   no switchport<br>
   __ip address 10.0.1.5/31__<br>
!<br>
__interface Ethernet2__<br>
   description Peer-to-peer link to leaf-2<br>
   no switchport<br>
   __ip address 10.0.2.5/31__<br>
!<br>
__interface Ethernet3__<br>
   description Peer-to-peer link to leaf-3<br>
   no switchport<br>
   __ip address 10.0.3.5/31__<br>
!<br>
__interface Loopback1__ <br>
   description IP for underlay -Router-ID <br>
   __ip address 10.2.2.1/32__ <br>
!<br>
interface Loopback2<br>
   description IP for overlay layer <br>
   ip address 10.2.2.2/32<br>
!<br>
ip routing<br>
!<br>
__router bgp 65500__<br>
   __router-id 10.2.2.1__<br>
   no bgp default ipv4-unicast <br>
   neighbor UNDERLAY peer group<br>
   neighbor UNDERLAY remote-as 65500<br>
   neighbor UNDERLAY next-hop-self<br>
   neighbor UNDERLAY route-reflector-client<br>
   neighbor grou peer group<br>
   neighbor 10.0.1.4 peer group UNDERLAY<br>
   neighbor 10.0.2.4 peer group UNDERLAY<br>
   neighbor 10.0.3.4 peer group UNDERLAY<br>
   !<br>
   address-family ipv4<br>
      neighbor UNDERLAY activate<br>
      network 10.2.2.1/32<br>
!<br>
На скриншоте снизу представлен вывод соседства со стороны spine 2 <br>
<img width="1067" height="559" alt="image" src="https://github.com/user-attachments/assets/47d7261d-e821-4adb-87f5-afb0fd65d889" />
<br>
<br>
### 2.3. Конфигурация Leaf-1 <br> <br>
hostname leaf1<br>
!<br>
spanning-tree mode mstp<br>
!<br>
vlan 1<br>
   name Host_Network<br>
!<br>
__interface Ethernet1__ <br>
   description Peer-to-peer link to Spine-1<br>
   no switchport<br>
   __ip address 10.0.1.0/31__ <br>
!<br>
__interface Ethernet2__ <br>
   description Peer-to-peer link to Spine-2<br>
   no switchport<br>
   __ip address 10.0.1.4/31__ <br>
!
interface Ethernet3<br>
   description -=Direction to host=-<br>
!<br>
__interface Loopback1__ <br>
   description IP for underlay -Router-ID<br>
   __ip address 10.0.0.1/32__ <br>
!<br>
interface Vlan1<br>
   ip address 192.168.1.1/24<br>
!<br>
ip routing<br>
!<br>
__router bgp 65500__ <br>
   __router-id 10.0.0.1__ <br>
   timers bgp 3 9<br>
   maximum-paths 2 ecmp 2<br>
   neighbor UNDERLAY peer group<br>
   neighbor UNDERLAY remote-as 65500<br>
   __neighbor 10.0.1.1 peer group UNDERLAY__ <br>
   __neighbor 10.0.1.5 peer group UNDERLAY__ <br>
   !<br>
   address-family ipv4<br>
      neighbor UNDERLAY activate<br>
      __network 10.0.0.1/32__ <br>
 Вывод команд bgp представлени ниже: <br>
<img width="1250" height="597" alt="image" src="https://github.com/user-attachments/assets/b3f2b138-830e-42c3-a42c-26074b844df0" /><br><br>
### 2.4. Конфигурация Leaf-2 <br>
hostname leaf2<br>
! <br>
spanning-tree mode mstp<br>
! <br>
interface Ethernet1<br>
   description Peer-to-peer link to Spine-1<br>
   no switchport<br>
   ip address 10.0.2.0/31<br>
!<br>
interface Ethernet2<br>
   description Peer-to-peer link to Spine-2<br>
   no switchport<br>
   ip address 10.0.2.4/31<br>
!<br>
interface Ethernet5<br>
   description -=Direction to hosts=-<br>
   no switchport<br>
   ip address 192.168.2.1/24<br>
interface Loopback0<br>
   ip address 2.2.2.2/32<br>
!<br>
interface Loopback1<br>
   description IP for underlay -Router-ID<br>
   ip address 10.0.0.2/32<br>
!<br>
ip routing<br>
!<br>
router bgp 65500<br>
   timers bgp 3 9<br>
   maximum-paths 2 ecmp 2<br>
   neighbor UNDERLAY peer group<br>
   neighbor UNDERLAY remote-as 65500<br>
   neighbor 10.0.2.1 peer group UNDERLAY<br>
   neighbor 10.0.2.5 peer group UNDERLAY<br>
   !<br>
   address-family ipv4<br>
      neighbor UNDERLAY activate<br>
      network 10.0.0.2/32<br>
<img width="1249" height="595" alt="image" src="https://github.com/user-attachments/assets/8df26f85-f78a-4b3d-9473-5ec832a38daa" /> <br><br>
## 2.5. Конфигурация Leaf-3 <br><br>
__hostname leaf3__ <br>
!<br>
vlan 3<br>
   name 3<br>
!<br>
interface Ethernet1<br>
   description Peer-to-peer link to Spine-1<br>
   no switchport<br>
   __ip address 10.0.3.0/31__ <br>
!<br>
interface Ethernet2<br>
   description Peer-to-peer link to Spine-2<br>
   no switchport<br>
   __ip address 10.0.3.4/31__ <br>
!<br>
interface Ethernet3<br>
   switchport access vlan 3<br>
!<br>
interface Loopback1<br>
   description IP for underlay -Router-ID<br>
   __ip address 10.0.0.3/32__ <br>
!<br>
interface Management1<br>
!<br>
interface Vlan3<br>
   ip address 192.168.3.1/24<br>
!<br>
ip routing<br>
!<br>
__router bgp 65500__ <br>
   __router-id 10.0.0.3__ <br>
   timers bgp 3 9<br>
   maximum-paths 2 ecmp 2<br>
   neighbor UNDERLAY peer group<br>
   neighbor UNDERLAY remote-as 65500<br>
   neighbor 10.0.3.1 peer group UNDERLAY<br>
   neighbor 10.0.3.5 peer group UNDERLAY<br>
   !<br>
   address-family ipv4<br>
      neighbor UNDERLAY activate<br>
      network 10.0.0.3/32<br>
<br>
<img width="1236" height="631" alt="image" src="https://github.com/user-attachments/assets/7cc12a5d-e0a8-434b-ad6f-4b6279ef606e" /><br>
### 3. Меняем конфигурацию Spin-ов на "Best Practice" <br>
В данном примере в настройках соседей в bgp со стороны Spine-ов, вместо команды __neighbor IP адрес LoopBack leaf-ов__, в данном примере введу одну строчку: "__bgp listen range 10.0.0.0/16 peer-group UNDERLAY remote-as 65500__ " <br>
<img width="518" height="530" alt="image" src="https://github.com/user-attachments/assets/e1e40d71-df22-49d8-9ec0-df07d287fc8d" />
<br>
Ниже картинка после изменения: <br>
<img width="1069" height="855" alt="image" src="https://github.com/user-attachments/assets/a4b524c5-f961-4edc-8b8d-0dc7af6a7390" />
<br>
## 4. Просмотр установления соседства <br>
К данной работе приложен пример трейса установления соседства. <br>
Предлагаю в кратце осветить ключевые моменты: <br>
Spine2(10.0.3.5) иницирует tcp сессию, отправив пакет на порт TCP 179, leaf3 (10.0.3.4), ПЕРВЫЙ пакет __SYN__, ожидаемый номер последовательности, Next Sequence Number =1 <br>
<img width="1014" height="824" alt="image" src="https://github.com/user-attachments/assets/ab0610a1-ed56-4bab-ac7e-c82212cff955" /><br>
<br>
2-й пакет от leaf3 к Spine 2 приходит ответ ACKnowledgment на первое SYN сообщение, во втором пакете мы видим, что установлены флаги SYN и ACK, а также появилась возможность замереить время прохождения пакетов и повился параметр RTT пример снизу: <br>
<img width="1029" height="948" alt="image" src="https://github.com/user-attachments/assets/24fc1b3c-fe3c-426e-965e-de08a0b79313" /><br><br>
На 3-ем ответном пакете установка BGP сессии завершается успехом ( в нашем случае), Spine 2 отвечает Leaf 3, что получил SYN, ACK, пакетом ACK, пример ниже: <br>
<img width="988" height="884" alt="image" src="https://github.com/user-attachments/assets/4860ff94-e3c8-4e81-9f6a-bdfeee1a2180" /> <br><br>
Далее Spine2 посылает первое BGP сообщение OPEN, в котором указывает Номер AS и свой ID: <br> <br>
<img width="869" height="884" alt="image" src="https://github.com/user-attachments/assets/8c32990a-e7a0-42d6-8610-81a68b2d7be0" /><br>
<br>
<br>
Далее, на получившее сообщенеие OPEN, leaf3 в рамках реализации работы TCP сессий, отсылает ACK  спайну, что я получил, пакет, все ОК.<br>
После чего leaf Шлет своё OPEN сообщение, в котором сообщает свой номер AS и ID: <br>
<img width="951" height="901" alt="image" src="https://github.com/user-attachments/assets/1afbb6c3-ddbc-48a2-b7cb-4c6f7807a4c2" /> <br><br>
Далее Spine в рамках все той же TCP сессиия, по правилам TCP отсылает ACK, что сообщение получил <br>
<img width="968" height="577" alt="image" src="https://github.com/user-attachments/assets/4ea4cf31-b90d-49c8-80ef-67ae5c5785b5" /> <br><br>
Далее Spine2 шлет лифу BGP KEEPALIVE Message, leaf также шлет в сторону спайна BGP KEEPALIVE, пример снизу:<br>
<img width="966" height="334" alt="image" src="https://github.com/user-attachments/assets/aee8bc43-4c2d-4b6f-953b-861a39f90341" /><br><br>
Пропустим описание работы сесссий TCP, с их постоянным ACK подтверждениями получения, следующий пакет Spine2 (10.0.3.5) шлет BGP UPDATE Message в сторону Leaf (10.0.3.4) и в следующем сообщении Leaf3 также шлет BGP UPDATE Message в сторону spine2. Пример ниже:   <br>
<img width="928" height="1936" alt="image" src="https://github.com/user-attachments/assets/07a92b79-5c7c-4280-b2f7-c6fee1e6d8d4" /> <br><br>
Ниже ответ UPDATE MEssage от Leaf3 (10.0.0.3) в сторону Spine2. <br>
<img width="872" height="876" alt="image" src="https://github.com/user-attachments/assets/861e6119-18d2-44f8-aaec-498a0521fc0c" /> <br><br>
На этом предлагаю разбор трейса закончить, подробно ознакомиться с трейсом можно самостоятельно, он находится в приложении. <br>
## 5. Просмотр информации вывода show команд <br>
Ниже представлен вывод маршрутной информации с Leaf3: <br>
<img width="1261" height="942" alt="image" src="https://github.com/user-attachments/assets/c825101a-5381-4545-a266-5fcb76f40b7c" /> <br><br>
Удобно использовать команду show ip route bgp, с расширением detail, подставляется информация из description интерфейсов, и сразу становится понятно, пример снизу:<br>
<img width="952" height="626" alt="image" src="https://github.com/user-attachments/assets/064540cc-9d54-41ec-8456-9732296b533d" /><br><br>
Далее, я хотел поразбираться с балансировкой трафика в ECMP, но тема оказалась на первый взгляд довольно сложной, данная статья поможет понять как она работаем, и почему балансировка per packet не применяется.
https://nag.ru/material/36217

Ниже пред
## 6. Попытка понять как работает ECMP
'''
 192.168.1.0/24
                        entries 2       announce 1      vtime:17
                        ip unicast BRIB last_best 451b8348
                        TSI:
                                BRIB task 192.168.1.0 ip unicast: 0x451be1b0, 2 entries
                                BGP route modification:  no metrics 0x7f894a893090, results 0x0 0x0 XXX rth 0x7f894ae0dc80 rt 0x7f89451b8348

                *BGP    7f89451b8348 Preference: 200    AttrId: 3               Source: 10.0.3.5
                        Nhe: 10.0.3.5 (7f89451b9000)    Instance: 0/0   Resolution: Resolved
                        Adjacency Id 130000010000000f   Refs 1 Hash 318, N_gw 2, Type IP
                        State: <Int Ext BGPEcmp Gateway ActiveU Unicast Resolve NheAccessor AdjAccessor>
                        Local AS: 65500 Peer AS: 65500
                        Age: 4:10:45    Metric: 0       Metric2: 100    Tag: 0   Seq: 0
                        Task: BGP_65500.10.0.3.5+179
                        Announcement bits(3): 1-static_sync 5-bgp_sync 10-BGP.0.0.0.0+179
                        AS Path: i (HashID 5) Nexthop: 10.0.3.5 Or-ID: 10.0.0.1 C-LST: 10.2.2.1

                BGP     7f89451b8438 Preference: 200    AttrId: 8
                        Nhe: 10.0.3.1 (7f89451b91c8)    Instance: 0/0   Resolution: Resolved
                        Adjacency Id 130000010000000e   Refs 6 Hash 300, N_gw 1, Type IP
                        State: <Int Ext BGPEcmp NoActiveU Gateway Unicast Resolve NheAccessor AdjAccessor>
                        Local AS: 65500 Peer AS: 65500
                        Age: 4:10:45    Metric: 0       Metric2: 100    Tag: 0   Seq: 0
                        Task: BGP_65500.10.0.3.1+179
                        AS Path: i (HashID 10) Nexthop: 10.0.3.1 Or-ID: 10.0.0.1 C-LST: 10.1.1.1
                  '''
                  <br>
<img width="712" height="199" alt="image" src="https://github.com/user-attachments/assets/e27540ed-8f59-49c2-b8b9-b96ffa32e42d" />












