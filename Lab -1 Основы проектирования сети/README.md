## Основы проектирования сети
# Цели занятия
  Разобрать логику проектирования адресного пространства для Underlay и Overlay сетей.
# Задание
  Распределите адресное пространство для Underlay сети.
<img width="914" height="503" alt="image" src="https://github.com/user-attachments/assets/81fea0f3-7bb0-4fd1-a47d-48b8780de459" />
Рукводствуясь следующими общими рекомендациями:
p2p - /30, /31 или unnumbered
loopback - /32
Например:
- IP = 10.Dn.Sn.X/31, где:
 - Dn – номер ЦОДа,
 - Sn – номер spine,
 - X – значение по порядку,
- Dn для DC1 = 0 – 7, где
- 0 – loopback1
- 1 – loopback2
- 2 – p2p links
- 3 – reserved
- 4-7 – services

# Разработанный адресный план для приведенной схемы сети приведен далее по тексту:
## Конфигурация коммутаторов уровня Spine
# ----------[ Spine Level]----------
### Конфигурация spine 1
interface Loopback1
   ip address 10.0.0.0/32

interface Ethernet1
       ip address 10.0.0.1/31 <br>
       Description Peer-to-peer link  to leaf-1  

interface Ethernet2
   ip address 10.0.0.3/31 <br>
   Description Peer-to-peer link  to leaf-2  

interface Ethernet3
   ip address 10.0.0.5/31 <br>
   Description Peer-to-peer link  to leaf-3  

### Конфигурация spine 2
interface Loopback1
   ip address 10.0.1.0/32

interface Ethernet1
   ip address 10.0.1.1/31
   Description Peer-to-peer link  to leaf-1

interface Ethernet2
   ip address 10.0.1.3/31
   Description Peer-to-peer link  to leaf-2

interface Ethernet3
   ip address 10.0.1.5/31
   Description Peer-to-peer link  to leaf-3


#  ----------[ Leaf Level]-------------
## Конфигурации коммутаторов уровня Leaf
   ### Конфигурация Leaf 1
interface Loopback1
   ip address 10.0.0.0/32 ---?

interface Ethernet1
   ip address 10.0.0.2/31
   Description Peer-to-peer link  to Spine-1

interface Ethernet2
   ip address 10.0.1.2/31
   Description Peer-to-peer link  to Spine-2

  ### Конфигурация Leaf 2
interface Loopback1
   ip address 10.0.0.0/32 ---?

interface Ethernet1
   ip address 10.0.0.4/31
   Description Peer-to-peer link  to Spine-1

interface Ethernet2
   ip address 10.0.1.4/31
   Description Peer-to-peer link to Spine-2

  ### Конфигурация Leaf 3
interface Loopback1
   ip address 10.0.0.0/32 ---?

interface Ethernet1
   ip address 10.0.0.6/31
   Description Peer-to-peer link to Spine-1

interface Ethernet2
   ip address 10.0.1.6/31
   Description Peer-to-peer link to Spine-2


