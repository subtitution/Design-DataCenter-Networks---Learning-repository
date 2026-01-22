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
## Итоговые настройки ISIS на Leaf1: <br>
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
<br>
<img width="876" height="166" alt="image" src="https://github.com/user-attachments/assets/8b5eec30-8d9e-4f05-adfe-59c2b5a36d55" /><br>
Первый пакет от Spine1, после включения ISIS <br>
<img width="1637" height="706" alt="image" src="https://github.com/user-attachments/assets/6b91d7fa-45b0-4525-abf3-dad2fae72bbf" /> <br>
Второе сообщение: <br>
<img width="1849" height="742" alt="image" src="https://github.com/user-attachments/assets/f738f19e-99be-4715-8096-f9fbf8c9a5e6" />

## 1.3. Добавим аналогичные настройки на Spine2, leaf2,leaf3
После добавления настроек на оставшееся оборудование, мне удалось запечатлить момент установки взаимотношений, между leaf3 и соседями.<br>
Ниже представлен скриншот первого сообщения во внешние интерфейсы, посланное leaf3 <br>
<img width="851" height="676" alt="image" src="https://github.com/user-attachments/assets/ed120846-02a1-4a04-bf77-754860cb0688" /><br>
В данном сообщении, мы видим следующее:<br>
- PDU Type: P2P Hello
- SystemID: 0000.0000.0013
- Point-to-Point Adjacency State: __Down__
- Все сообщения посылаются в "общую песочницу", некий такой мультикаст-бродкаст адрес: 0900:2b00:0005
<br>
  В данной песочнице, Spine1 увидел сообщение Hello и говорит, "давай дружить", посылая в ответ также Hello сообщение, Но Adjacency State: Initializing <br>
  Скрин ниже: <br>
 <img width="765" height="491" alt="image" src="https://github.com/user-attachments/assets/5092d7db-33ee-4491-8646-365371933221" /><br>
  И вот на __3-м__ сообщении, они договорились дружить, и быть соседями <br>
  <img width="913" height="491" alt="image" src="https://github.com/user-attachments/assets/51860762-09b4-4a36-b83b-19780b449e1e" /> <br>
  После установления сосдства, маршрутизаторы начинают обмениваться сообщениями CSNP (Complete Sequence Number PDU (CSNP) в протоколе IS-IS — это пакет, содержащий полный список всех LSP (Link State PDU) в базе данных (LSDB) маршрутизатора)  (это аналог DB (database Description в OSPF)) <br>
  Ниже пример CSNP сообщения (Spine1, шлёт leaf1 информацию, и говорит, смотри, что у меня есть:))<br>
  <img width="731" height="734" alt="image" src="https://github.com/user-attachments/assets/e6b7a72f-23ae-455d-a006-fb51f5334d95" />
<br>
  ## 3. Проверка маршрутов и связности <br>
  Просмотрим информацию о соседях например на leaf3: <br>
   <img width="857" height="95" alt="image" src="https://github.com/user-attachments/assets/0017f043-9364-4d94-82db-ebf11dbe2e5d" /> <br>
   Теперь давайте взглянем на анонсированные сети и через какие интерфейсы они доступны?:<br>
   <img width="539" height="414" alt="image" src="https://github.com/user-attachments/assets/784af28c-19db-4cac-a528-995f7fa8fe99" /> <br>
   Очень информативная команда, которая мне понравилась, это просмотр топологии isis, снизу пример вывода, думаю комментарии излишни и так все понятно: <br>
  <img width="703" height="195" alt="image" src="https://github.com/user-attachments/assets/d18805a5-d9f4-448d-91b5-893438e73d46" /><br>
Ну и на последок, я добавил в анонсы сеть 192.168.1.0/24, на leaf1, проверим и пинганем хост, который подключен к leaf1. <br>
<img width="473" height="579" alt="image" src="https://github.com/user-attachments/assets/985917da-3d56-483d-9ed8-7bcc25d1a95e" /> <br>
Имитируем отказ одного линка, выключаем порт в сторону Spine 1, и проверяем трейсом, до выключения трафик шел через spine1, после отключения, трафик пошел через Spine2. <br>
<br>
<img width="580" height="675" alt="image" src="https://github.com/user-attachments/assets/d086a662-8837-451d-b728-3343a7e239b8" />


   










