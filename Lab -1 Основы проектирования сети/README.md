## Основы проектирования сети
# Цели занятия
  Разобрать логику проектирования адресного пространства для Underlay и Overlay сетей.
# Задание
  ### Распределите адресное пространство для Underlay сети.

Схема сети клоза с распределенными адресами, представлена снизу на рисунке.
<img width="1053" height="651" alt="image" src="https://github.com/user-attachments/assets/905d5bf4-1480-4dc0-af1a-fd38076b0f59" />
Рукводствуясь следующими общими рекомендациями:
p2p - /30, /31 или unnumbered
loopback - /32
Например:
#### IP = 10.Dn.Sn.X/31, где:
 - Dn – номер ЦОДа,
 - Sn – номер spine,
 - X – значение по порядку,
- Dn для DC1 = 0 – 7, где
- 0 – loopback1
- 1 – loopback2
- 2 – p2p links
- 3 – reserved
- 4-7 – services

Разработанный адресный план для приведенной схемы сети приведен далее по тексту:
### Конфигурация коммутаторов уровня Spine
# Конфигурация spine 1
   
#### interface Loopback1<br>
  description IP for underlay -Router-ID <br>
  ip address 10.1.1.1/32 <br>

#### interface Loopback2<br>
  description IP for overlay layer <br>
  ip address 10.1.1.2/32 <br>

#### interface Ethernet1<br>
         ip address 10.0.1.1/31
         Description Peer-to-peer link  to leaf-1 
  
#### interface Ethernet2<br>
     ip address 10.0.2.1/31
     Description Peer-to-peer link  to leaf-2 

#### interface Ethernet3<br>
     ip address 10.0.3.1/31
     Description Peer-to-peer link  to leaf-3

# Конфигурация spine 2
#### interface Loopback1<br>
    description  IP for underlay -Router-ID  <br>
    ip address 10.2.2.1/32<br>
   
#### interface Loopback2<br>
    description IP for overlay layer <br>
    ip address 10.2.2.2/32<br>
  
### interface Ethernet1<br>
     ip address 10.0.1.5/31<br>
     Description Peer-to-peer link  to leaf-1<br>

### interface Ethernet2<br>
     ip address 10.0.2.5/31<br>
     Description Peer-to-peer link  to leaf-2<br>

### interface Ethernet3<br>
     ip address 10.0.3.5/31<br>
     Description Peer-to-peer link  to leaf-3<br>

# Конфигурации коммутаторов уровня Leaf
   ## Конфигурация Leaf 1

#### interface Loopback1<br>
    description IP for underlay -Router-ID 
    ip address 10.0.0.1/32 
  
#### interface Ethernet1
     ip address 10.0.1.0/31
     Description Peer-to-peer link  to Spine-1

#### interface Ethernet2
     ip address 10.0.1.4/31
     Description Peer-to-peer link  to Spine-2

  ## Конфигурация Leaf 2
#### interface Loopback1<br>
    description IP for underlay -Router-ID 
    ip address 10.0.0.2/32

#### interface Ethernet1
     ip address 10.0.2.0/31
     Description Peer-to-peer link  to Spine-1

#### interface Ethernet2
     ip address 10.0.2.4/31
     Description Peer-to-peer link to Spine-2

  ## Конфигурация Leaf 3
#### interface Loopback1
      description IP for underlay -Router-ID 
      ip address 10.0.0.3/32

#### interface Ethernet1
     ip address 10.0.3.0/31
     Description Peer-to-peer link to Spine-1

#### interface Ethernet2
     ip address 10.0.3.4/31
     Description Peer-to-peer link to Spine-2

## Проверка IP связности между смежными узлами
Т.к. автоматическая маршрутизация, с использованием специальных протоколов не настроена,  впрочем и настройки статической маршрутизации также отсутсвуют, то мы сможем проверить  IP связность только смежных устройств, у которых имеется Directed connect.
### Проверим с leaf1 IP доступ до spine 1 и spine 2

leaf1#ping 10.0.1.1
PING 10.0.1.1 (10.0.1.1) 72(100) bytes of data.
80 bytes from 10.0.1.1: icmp_seq=1 ttl=64 time=105 ms
80 bytes from 10.0.1.1: icmp_seq=2 ttl=64 time=105 ms
80 bytes from 10.0.1.1: icmp_seq=3 ttl=64 time=98.3 ms
80 bytes from 10.0.1.1: icmp_seq=4 ttl=64 time=96.8 ms
80 bytes from 10.0.1.1: icmp_seq=5 ttl=64 time=95.2 ms


leaf1#ping 10.0.1.5
PING 10.0.1.5 (10.0.1.5) 72(100) bytes of data.
80 bytes from 10.0.1.5: icmp_seq=1 ttl=64 time=48.8 ms
80 bytes from 10.0.1.5: icmp_seq=2 ttl=64 time=42.9 ms
80 bytes from 10.0.1.5: icmp_seq=3 ttl=64 time=64.3 ms
80 bytes from 10.0.1.5: icmp_seq=4 ttl=64 time=64.9 ms
80 bytes from 10.0.1.5: icmp_seq=5 ttl=64 time=70.5 ms

### Теперь проверим доступ с leaf3
leaf1#
leaf1#ping 10.0.3.0
connect: Network is unreachable

Разумеется доступа нет, т.к. не настроена маршрутизация.


