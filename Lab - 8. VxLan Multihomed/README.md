# VXLAN. Multihoming

## Цель:
Настроить отказоустойчивое подключение клиентов с использованием EVPN Multihoming
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
#### 1.1.2. Настроим интерфейс vxlan1, указав loopback1 в качестве источника.
vxlan1 - это логический интерфейс, который будет обеспечивать функции инкапсуляции и декапсуляции заголовков VXLAN <br>
Команда (__vxlan source-interface Loopback1__):
```
interface Vxlan1
   vxlan source-interface Loopback1
```
### 1.2. Настройка службы EVPN Layer 2 на leaf3 коммутаторе
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
