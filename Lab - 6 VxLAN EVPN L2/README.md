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
