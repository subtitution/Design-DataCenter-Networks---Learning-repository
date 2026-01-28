# Настройка BGP для Underlay сети <br>
 Задание: <br>
1. Настроите BGP в Underlay сети, для IP связанности между всеми сетевыми устройствами. iBGP или eBGP - решать вам!<br>
2. Зафиксируете в документации - план работы, адресное пространство, схему сети, конфигурацию устройств<br>
3. Убедитесь в наличии IP связанности между устройствами в BGP домене<br>
## План работ
- настроить iBGP
- посмотреть процесс установления BGP сессии
- убедиться в IP связности, попробовать отключить один линк
- настроить eBGP
- убедиться в ip связности
<br><br><img width="527" height="326" alt="laba" src="https://github.com/user-attachments/assets/a7e00c82-9960-4a15-851a-bd3a9f01a8b0" />
<br>
Сверху, представлена схема используемой сети. <br>

leaf2#sho run section bgp  <br>
router bgp 65500  <br>
   maximum-paths 2 ecmp 2  <br>
   neighbor UNDERLAY peer group  <br>
   neighbor UNDERLAY remote-as 65500  <br>
   !  <br>
   address-family ipv4  <br>
      neighbor UNDERLAY activate  <br>
      network 10.0.0.2/32  <br>
