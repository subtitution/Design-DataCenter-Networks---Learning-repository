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
### 1. Давайте добавим SVI адреса и VLAN для подключения хостов<br>
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
Хост не доступен, смотрим настройки порта eth3, который смотрит в сторону хоста:<br>
interface Ethernet3<br>
   description -=Direction to host=-<br>
   no switchport<br>
<br><br>
   Меняем конфигурацию порта на switchport, проверяем.<br>
   leaf1# ping 192.168.1.2<br>
PING 192.168.1.2 (192.168.1.2) 72(100) bytes of data.<br>
80 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=10.1 ms<br>


  
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































