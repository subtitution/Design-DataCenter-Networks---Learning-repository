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
<img width="596" height="103" alt="image" src="https://github.com/user-attachments/assets/ca39f740-72b3-4fad-8268-bb9f11e15078" />
По выводу команды, мы понимаем, что Leaf2/3 доступны через spine 1/2.

