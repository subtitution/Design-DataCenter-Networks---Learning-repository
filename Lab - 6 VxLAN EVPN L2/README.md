# VxLAN L2 VNI <br>
Цель работы: 
Настроить Overlay на основе VxLAN EVPN для L2 связанности между клиентами.

## План работы
- Вспомнить теорию и основные термины
- Настроить BGP peering между Leaf и Spine в AF (Adress Family) l2vpn evpn
- Настроить связанность между клиентами в первой зоне и убедиться в её наличии

## Список терминов и сокращений:
- __NVO__ - __Network Virtualization Overlay__ - оверлейная сеть
- __NVE__ - Network Virtualization Edge. Туннельный интерфейс инкапсуляции/декапсуляции фреймов
- __VTEP__ - __VxLAN Tunnel End Point__ - устройство, которое занимается инкапсуляцией/декапсуляцией фреймов  (обычно leaf)
- __VNI__ - Virtual Network Identifier - vtnrf МчДФТ инкапсуляции, определяющая Layer 2 домен в оверлей сети
- __EVI__ - EVPN Instance - логический свитч в EVPN домене
- __MAC-VRF__ - Virtual Routing and Forwarding table для MAC адресов

# 1. Теория
## Типы маршрутов EVPN

### Основные операци
- __Type-2 Route__: Host Advertisement Route (Для анонса информации о подключенных хостах)
- __Type-3 Route__: Inclusive Multicast Ethernet TAG Route (IMET) - Для работы с BUM (Ingress Replication)
  
### Multi-homing
- __Type-4 Route__ - Ethernet Segment Route (Для выбора Designed Forwarding (Кто будет управлять BUM трафиком))
- __Type-1 Route__ - Ethernet Auto-Discovery Route (Для объявления Ethernet Segment Identifier (__ESI__) (Для конвергенции и балансировки))

### Связь с внешним миром
- __Type-5 Route__ - IP-prefix route advertisement (Для анонса внешних маршрутов в фабрику)

### Multicast
- __Type-6,7,8__ - Для распространения PIM, IGMP Leave/Join по фабрике
# 2. Практика - Настройка
## 2.1. Настройка leaf1
```
leaf1(config)#service routing protocols model multi-agent
! Change will take effect only after switch reboot

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
interface Ethernet1
   description Peer-to-peer link to Spine-1
   no switchport
   ip address 10.0.1.0/31
!
interface Ethernet2
   description Peer-to-peer link to Spine-2
   no switchport
   ip address 10.0.1.4/31
!
interface Ethernet3
   description -=Direction to host=-
!

interface Loopback0
   description for Underlay network
   ip address 10.0.0.1/32
!
interface Loopback1
   description for Overlay VxLAN loobback
   ip address 10.1.0.1/32
!
interface Management1
!
interface Vlan1
   ip address 192.168.1.1/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 1 vni 1
!
ip routing
!
router bgp 65501
   router-id 10.0.0.1
   timers bgp 3 9
   maximum-paths 2 ecmp 2
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65500
   neighbor UNDERLAY out-delay 0
   neighbor UNDERLAY update-source Loopback0
   neighbor UNDERLAY send-community extended
   neighbor 10.0.1.1 peer group UNDERLAY
   neighbor 10.0.1.1 remote-as 65500
   neighbor 10.0.1.1 send-community extended
   neighbor 10.0.1.5 peer group UNDERLAY
   neighbor 10.0.1.5 remote-as 65500
   neighbor 10.0.1.5 send-community extended
   !
   vlan 1
      rd auto
      route-target both 1:1
      redistribute learned
   !
   address-family evpn
      neighbor 10.0.1.1 activate
      neighbor 10.0.1.5 activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.0.0.1/32
!
end
leaf1#
```
