## Основы проектирования сети
# Цели занятия
разобрать логику проектирования адресного пространства.

# Краткое содержание
рассмотрим логику проектирования адресных пространств для underlay и overlay сети;
суммаризация сетей;
настройка интерфейсе через unnumbered;

## Основы проектирования сети
# Цели занятия
разобрать логику проектирования адресного пространства.

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
## Был разработан адресный план для приведенной на примере схеме сети:
### конфигурация spine 1
interface Loopback1
   ip address 10.1.1.1/31

interface Ethernet1
   ip address unnumbered Loopback1

### конфигурация spine 2
interface Loopback1
   ip address 10.1.2.1/31

interface Ethernet1
   ip address 10.10.2.1/31
