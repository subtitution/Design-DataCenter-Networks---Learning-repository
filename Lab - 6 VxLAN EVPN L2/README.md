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

# Теория
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

leaf1(config)#service routing protocols model multi-agent
! Change will take effect only after switch reboot

leaf1(config-if-Vx1)#vxlan vlan 1 vni ?
  $                  list end
  <0.1-65535.65534>  VXLAN Network Identifier (VNI) or range(s) of VNIs

leaf1(config-if-Vx1)#vxlan vlan 1 vni 1
