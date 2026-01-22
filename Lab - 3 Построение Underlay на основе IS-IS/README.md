# Построение Underlay сети на основе протокола ISIS
## Цель
-  Настроить ISIS в Underlay сети, для IP связанности между всеми сетевыми устройствами. <br><br>

   При выполнении данной работы следует использовать следующие рекоммендации: <br>
   
  - Использовать P2P network type на интерфейсах
  - Использовать BFD в реальной жизни ( В лабе использовать не буду)
  - Для одного POD достаточно соединить всё L1 линками.
<br>
Ниже под текстом, вашему вниманию представлена используемая схема сети.
<img width="1053" height="651" alt="image" src="https://github.com/user-attachments/assets/4e18dff0-3110-4546-ab92-8426583554c1" />


# 1. Настроим соседство с leaf-1 и Spine-1, leaf-1 и Spine-2 <br><br>
## 1.1. Произведем настройки на Leaf1 <br>
leaf1# __sho run section isis__  <br>
interface Ethernet1<br>
__isis enable Underlay1__ ! __После включения данной команды на интерфейсе, начали бегать ISIS пакеты__ 
<br><br>
__router isis Underlay1__ <br>
net 49.0000.0000.0000.0001.00 <br>
is-type level-1 <br>
!<br>
 address-family ipv4 unicast<br>
 Ниже пример, как выглядит первый пакет __Hello__ <br>
<img width="647" height="840" alt="image" src="https://github.com/user-attachments/assets/d784756e-83e9-4a11-9f01-4327acfd6d36" /> <br>
Далее включаем на интерфейсе eth1 __ISIS network point-to-point__ <br> 
И сразу стоит обратить внимание, после включения меняется Destination mac adress, с бродкаста, на __mac адрес Intermidate System to Intermidate System Hello.__ <br>
А также меняется PDU type   PDU type: __L1 HELLO (15)__ <br>
На -->  PDU Type: __P2P HELLO (17)__ на Point-to-Point <br>
<img width="715" height="872" alt="image" src="https://github.com/user-attachments/assets/572a2099-fb80-43ca-b13b-d40856e025d7" /> <br>
<br>
### Итоговые настройки ISIS на Leaf1:<br>
leaf1#sho run section isis<br><br>
__interface Ethernet1__<br>
         isis enable Underlay1<br>
         isis network point-to-point<br>
__interface Loopback1__<br>
      isis enable Underlay1<br>
__router isis Underlay1__<br>
         net 49.0000.0000.0000.0001.00<br>
         is-type level-1<br>
   !<br>
   address-family ipv4 unicast<br>
## 1.2. Произведем аналогичные настройки ISIS на Spine1 <br>
Вот такие настройки на spine1:<br>
spine1#sho run section isis<br> <br>
__interface Ethernet1__<br>
   isis enable Underlay1<br>
   isis network point-to-point<br>
__interface Loopback1__<br>
   isis enable Underlay1<br>
__router isis Underlay1__<br>
   net 49.0000.0000.0000.0011.00<br>
   is-type level-1<br>
   !<br>
   address-family ipv4 unicast<br><br>
Полезная команда (__sho isis spf log__), просмотреть историю отношений: <br>
<img width="876" height="165" alt="image" src="https://github.com/user-attachments/assets/085eca66-f639-40ae-8f45-4bc50545b1c9" />


