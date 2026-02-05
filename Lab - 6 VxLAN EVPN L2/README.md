# VxLAN L2 VNI <br>
Цель работы: 
Настроить Overlay на основе VxLAN EVPN для L2 связанности между клиентами.

## План работы
- Настроить BGP peering между Leaf и Spine в AF l2vpn evpn
- Настроить связанность между клиентами в первой зоне и убедиться в её наличии

## Список терминов и сокращений:
- __NVO__ - __Network Virtualization Overlay__ - оверлейная сеть
- __NVE__ - Network Virtualization Edge. Туннельный интерфейс инкапсуляции/декапсуляции фреймов
- __VTEP__ - __VxLAN Tunnel End Point__ - устройство, которое занимается инкапсуляцией/декапсуляцией фреймов  (обычно leaf)
- __VNI__ - Virtual Network Identifier - vtnrf МчДФТ инкапсуляции, определяющая Layer 2 домен в оверлей сети
- __EVI__ - EVPN Instance - логический свитч в EVPN домене
- __MAC-VRF__ - Virtual Routing and Forwarding table для MAC адресов
  
 Задание: <br>
1. Настроите BGP в Underlay сети, для IP связанности между всеми сетевыми устройствами. iBGP или eBGP - решать вам!<br>
2. Зафиксируете в документации - план работы, адресное пространство, схему сети, конфигурацию устройств<br>
3. Убедитесь в наличии IP связанности между устройствами в BGP домене<br>
<br><br>
leaf2#sho run section bgp  <br>
router bgp 65500  <br>
   maximum-paths 2 ecmp 2  <br>
   neighbor UNDERLAY peer group  <br>
   neighbor UNDERLAY remote-as 65500  <br>
   !  <br>
   address-family ipv4  <br>
      neighbor UNDERLAY activate  <br>
      network 10.0.0.2/32  <br>
