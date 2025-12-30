# Построение Underlay сети с использованием OSPF
## Цель:
Настроить OSPF для Underlay сети.
### План работы
- Для IP адресации возьмем схему из лабораторного задания 1
- Для конечных устройств, определить используемую IP подсеть
- Настроить OSPF в Underlay сети, для IP связанности между всеми сетевыми устройствами
- Убедится в наличии IP связанности между устройствами в OSFP домене

Возьмем для основы схему из 1-й лабораторной работы, к ней добавим IP подсети для конечных устройств, пример схемы L3 приведен ниже: <br>
<img width="527" height="326" alt="OSPF" src="https://github.com/user-attachments/assets/14d81854-fad3-455f-9904-657817d00b77" />




Для Хостов которые будут жить за лифами будем использовать сеть 192.168.x.0/24 <br>
где х - это номер leaf<br>
Поехали:<br>
### 1. Давайте добавим SVI адреса и VLAN для подключения хостов - Настраиваем Leaf 1<br>
- Создаем vlan 1, для сети хостов <br>
  vlan <br>
   name Host_Network<br>
  - Согдаем интерфейс vlan 1<br>
    <br><br>
- interface Vlan1<br>
   ip address 192.168.1.1/24<br>
<br><br>
  Теперь проверим доступность хоста с ip адресом 192.168.1.2, который находится за 3-м ethernet портом:<br>
  leaf1#<br>
leaf1#ping 192.168.1.2<br>
PING 192.168.1.2 (192.168.1.2) 72(100) bytes of data.<br>
<br>
--- 192.168.1.2 ping statistics ---<br>
5 packets transmitted, 0 received, 100% packet loss, time 43ms<br>
<br><br>
Хост не доступен, проверяем настройки порта eth3, который смотрит в сторону хоста:<br>
interface Ethernet3<br>
   description -=Direction to host=-<br>
   no switchport<br>
<br><br>
   Меняем конфигурацию порта на switchport, проверяем.<br>
   leaf1# ping 192.168.1.2<br>
PING 192.168.1.2 (192.168.1.2) 72(100) bytes of data.<br>
80 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=10.1 ms<br>

## Настроим и включим ospf на интерфейсах eth1 and eth2 на leaf1, интерфейсы идут в сторону spein1 / spine 2
leaf1(config)#int eth1<br>
leaf1(config-if-Et1)#ip ospf area ?<br>
  A.B.C.D         OSPF area-id in IP address format<br>
  <0-4294967295>  OSPF area-id in decimal format<br>
<br><br>
leaf1(config-if-Et1)#ip ospf area 0.0.0.0<br>
leaf1(config-if-Et1)#ip ospf network point-to-point<br>
<br>
Аналогичные настройки прописываем на интерфейсе eth2<br>
<br>
interface Ethernet2<br>
   description Peer-to-peer link to Spine-2<br>
   no switchport<br>
  ip address 10.0.1.4/31  <br>
   ip ospf network point-to-point<br>
   ip ospf area 0.0.0.0<br>
!
### try to enable OSPF proccess on leaf1
leaf1(config)#router ospf 1<br>
leaf1(config-router-ospf)#router-id 10.0.0.1<br>
<br>
After this command entered, we can see the first message from leaf 1, you can see it on screen below:

<img width="1852" height="645" alt="image" src="https://github.com/user-attachments/assets/85545c79-6585-420d-bb8e-732b5feab880" />

As you can see, после включения оспиэф, первое сообщение от роутера, это IGMP которое сообщает, что роутер присоединился к группе мультикасата с адресом  224.0.0.5 <br>
### Теперь, давайте рассмотрим первое Hello сообщение от Leaf1
На рисунке ниже представлен скриншот сообщения, давайте посмотрим, на что стоит обратить внимание?
<img width="909" height="255" alt="image" src="https://github.com/user-attachments/assets/0e347185-0416-4b54-b15b-42702ba0f0f4" />


<img width="999" height="988" alt="image" src="https://github.com/user-attachments/assets/0b8c12a1-f606-4f24-ab9a-29c96d876c59" />


ip ospf summary <br>
OSPF instance 1 with ID 10.0.0.1, VRF default <br>
Time since last SPF: 12517 s <br>
Max LSAs: 12000, Total LSAs: 1 <br>
Type-5 Ext LSAs: 0 <br>
ID               Type   Intf   Nbrs (full) RTR LSA NW LSA  SUM LSA ASBR LSA TYPE-7 LSA <br>
0.0.0.0          normal 3      0    (0   ) 1       0       0       0       0 <br>

![Uploading image.png…]()


   <br>
#### Leaf 1, конфигурация порта в сторону хоста<br>
<br> 
interface Ethernet3<br>
         description -=Direction to host=-<br>
         no switchport<br>
         ip address 192.168.1.1/24<br>
         <br> <br>
### Leaf 2, конфигурация порта в сторону хоста<br>
interface Ethernet5 <br>
   description -=Direction to hosts=-<br>
   no switchport<br>
   ip address 192.168.2.1/24<br>

<br>
#### Leaf 3, конфигурация порта в сторону хоста<br>
interface Vlan3<br>
   ip address 192.168.3.1/24<br>
<br><br>
   interface Ethernet3<br>
   switchport access vlan 3<br>









































