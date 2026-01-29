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
Видно ТЕПЕРЬ видно, что адреса __Next-hop__ поменялись, на адреса Spine-ов, у Spine и leaf есть связь, через пиринговые каналы, теперь маршрутизация будет работать. <br>
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
В данном примере в настройках соседей в bgp, мы используем команду __neighbor IP адрес LoopBack leaf-ов__, в данном случае это рабочий вариант и е критично, т.к. кол-во leaf-ов всего 3. Но в большинстве случаев, когда используется большее кол-во leaf-ов, лучше сразу использовать другую команду: __bgp listen range 10.0.0.0/16 peer-group UNDERLAY remote-as 65500__ , где 10.0.0.0/16 диапозон используемых адресов для loopback leaf-ов. Далее ниже по тексту, мы поменяем настройки.<br>


leaf2#sho run section bgp  <br>
router bgp 65500  <br>
   maximum-paths 2 ecmp 2  <br>
   neighbor UNDERLAY peer group  <br>
   neighbor UNDERLAY remote-as 65500  <br>
   !  <br>
   address-family ipv4  <br>
      neighbor UNDERLAY activate  <br>
      network 10.0.0.2/32  <br>
