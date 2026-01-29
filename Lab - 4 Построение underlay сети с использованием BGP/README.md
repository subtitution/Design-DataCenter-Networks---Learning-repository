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
<img width="1250" height="597" alt="image" src="https://github.com/user-attachments/assets/b3f2b138-830e-42c3-a42c-26074b844df0" />
<br><br>
### 2.4. Конфигурация Leaf-2 <br><br>
hostname leaf2<br>
!<br>
spanning-tree mode mstp<br>
!<br>
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
### 2.5. Конфигурация Leaf-3 <br><br>

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
<img width="1236" height="631" alt="image" src="https://github.com/user-attachments/assets/7cc12a5d-e0a8-434b-ad6f-4b6279ef606e" />
<br>
### 3. Меняем конфигурацию Spin-ов на "Best Practice" <br>
В данном примере в настройках соседей в bgp со стороны Spine-ов, вместо команды __neighbor IP адрес LoopBack leaf-ов__, в данном примере введу одну строчку: "__bgp listen range 10.0.0.0/16 peer-group UNDERLAY remote-as 65500__ " <br>
<img width="518" height="530" alt="image" src="https://github.com/user-attachments/assets/e1e40d71-df22-49d8-9ec0-df07d287fc8d" />
<br>
Ниже картинка после изменения: <br>
<img width="1069" height="855" alt="image" src="https://github.com/user-attachments/assets/a4b524c5-f961-4edc-8b8d-0dc7af6a7390" />
<br>

     



      
      


      


leaf2#sho run section bgp  <br>
router bgp 65500  <br>
   maximum-paths 2 ecmp 2  <br>
   neighbor UNDERLAY peer group  <br>
   neighbor UNDERLAY remote-as 65500  <br>
   !  <br>
   address-family ipv4  <br>
      neighbor UNDERLAY activate  <br>
      network 10.0.0.2/32  <br>
