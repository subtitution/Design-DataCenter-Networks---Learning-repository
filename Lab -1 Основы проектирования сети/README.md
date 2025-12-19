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
#### ----[ конфигаруция для Spine Underlay Level]----
## Конфигурация spine 1
   
#### interface Loopback1<br>
  description IP for underlay -Router-ID
  ip address 10.0.1.1/32

#### interface Loopback2<br>
  description IP for overlay layer
  ip address 10.1.1.1/32

#### interface Ethernet1<br>
         ip address 10.2.1.1/31 
         Description Peer-to-peer link  to leaf-1 
  
#### interface Ethernet2<br>
     ip address 10.2.1.2/31
     Description Peer-to-peer link  to leaf-2 

#### interface Ethernet3<br>
     ip address 10.2.1.3/31
     Description Peer-to-peer link  to leaf-3

## Конфигурация spine 2
interface Loopback1<br>
  description  IP for underlay -Router-ID  <br>
    ip address 10.0.2.1/32<br>
   
interface Loopback2<br>
  description IP for overlay layer <br>
  ip address 10.1.2.1/32<br>
  
### interface Ethernet1<br>
   ip address 10.2.2.1/31<br>
   Description Peer-to-peer link  to leaf-1<br>

### interface Ethernet2<br>
   ip address 10.2.2.2/31<br>
   Description Peer-to-peer link  to leaf-2<br>

### interface Ethernet3<br>
   ip address 10.2.2.3/31<br>
   Description Peer-to-peer link  to leaf-3<br>

# ----------[ конфигаруция для Spine Overlay Level]----------
В данном блоке опишем какие IP адреса будут использоваться для Overlay <br>
interface Loopback2<br>
  <p>description IP for overlay routing  <br>
  ip address 10.1.0.0/32<br>

interface Ethernet1<br>
         ip address 10.0.1.1/31 <br>
         Description Peer-to-peer link  to leaf-1  <br>
  
interface Ethernet2<br>
     ip address 10.0.1.3/31 <br>
     Description Peer-to-peer link  to leaf-2  <br>

interface Ethernet3<br>
     ip address 10.0.0.5/31 <br>
     Description Peer-to-peer link  to leaf-3<br>

### Конфигурация spine 2
interface Loopback1<br>
  description IP for underlay routing  <br>
    ip address 10.0.1.0/32<br>
   

interface Ethernet1<br>
   ip address 10.2.2.1/31<br>
   Description Peer-to-peer link  to leaf-1<br>

interface Ethernet2<br>
   ip address 10.2.2.2/31<br>
   Description Peer-to-peer link  to leaf-2<br>

interface Ethernet3<br>
   ip address 10.2.2.3/31<br>
   Description Peer-to-peer link  to leaf-3<br>

#  -----[ Leaf Level]------
### Конфигурации коммутаторов уровня Leaf
   ## Конфигурация Leaf 1
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


