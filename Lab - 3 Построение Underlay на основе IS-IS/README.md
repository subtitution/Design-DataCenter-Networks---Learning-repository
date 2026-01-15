# Построение Underlay сети на основе протокола ISIS
## Цель
-  Настроить ISIS в Underlay сети, для IP связанности между всеми сетевыми устройствами. <br><br>

   При выполнении данной работы следует использовать следующие рекоммендации: <br>
   
  - Использовать P2P network type на интерфейсах
  - Использовать BFD в реальной жизни ( В лабе использовать не буду)
  - Для одного POD достаточно соединить всё L1 линками.
<br>
    ### План работ
1. Настроим соседство с leaf-1 и Spine-1, leaf-1 и Spine-2 <br><br>
leaf1# __sho run section isis__  <br>
interface Ethernet1<br>
         __isis enable Underlay1__ ! __После включения данной команды на интерфейсе, начали бегать ISIS пакеты__ 
<br><br>
router isis Underlay1 <br>
         net 49.0000.0000.0000.0001.00 <br>
         is-type level-1 <br>
   !<br>
      address-family ipv4 unicast<br>
      Ниже пример, как выглядит первый пакет Hello <br>
      <img width="647" height="840" alt="image" src="https://github.com/user-attachments/assets/d784756e-83e9-4a11-9f01-4327acfd6d36" />

