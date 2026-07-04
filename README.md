Привет! Я веду эту страницу, как публичный дневник своего роста до сетевого архитектора ЦОД.

##### Я делюсь тем, что изучаю и применяю на практике:
- 📘 Конспекты и шпаргалки по сложным темам
- 🔧 Лабораторные работы в EVE-NG/GNS3
- 📐 Примеры архитектурных решений и их обоснование
- 💡 Разбор реальных кейсов и ошибок проектирования
- 📈 Обзор инструментов (DC Fabric, ACI, NDFC) и их сравнение

### Здесь я систематизирую свой опыт и обучение, разбираю ключевые технологии и принципы проектирования дата-центров:

- • EVPN/VXLAN — архитектура overlay сетей
- • BGP & Underlay (IS-IS, OSPF) — отказоустойчивая маршрутизация
- • Design Principles — фабрики сетей (Clos), масштабируемость, согласованность
- • Automation & DevOps — Ansible, Python, Nornir для сетей
- • Multi-vendor — концепции для Cisco, NVIDIA, Arista, Juniper
- • Best Practices — документация, стандартизация, переходные периоды

## Моя цель — создать структурированную базу знаний и помочь инженерам вырасти до архитекторов.

Литература:
- ⚪️  A Modern, Open, and Scalable Fabric: VXLAN EVPN [Электронный ресурс]. – Режим доступа: https://www.cisco.com/c/dam/en/us/td/docs/switches/datacenter/nexus9000/sw/vxlan_evpn/VXLAN_EVPN.pdf.
- ⚪️ Cisco VXLAN BGP EVPN Design and Implementation Guide [Электронный ресурс]. – Режим доступа: https://www.cisco.com/c/en/us/td/docs/dcn/whitepapers/cisco-vxlan-bgp-evpn-design-and-implementation-guide.html.
- ⚪️Data Center Fabric (Juniper) [Электронный ресурс]. – Режим доступа: https://www.juniper.net/documentation/us/en/software/nce/sg-005-data-center-fabric/sg-005-data-center-fabric.pdf.
- ⚪️ EVPN (Juniper) [Электронный ресурс]. – Режим доступа: https://www.juniper.net/documentation/us/en/software/junos/evpn/evpn.pdf.
- ⚪️ VxLAN фабрика. Часть 1 [Электронный ресурс]. – Режим доступа: https://habr.com/ru/company/otus/blog/505442/.
- ⚪️ VxLAN фабрика. Часть 2 [Электронный ресурс]. – Режим доступа: https://habr.com/ru/company/otus/blog/506800/.
- ⚪️ VxLAN фабрика. Часть 2.5 [Электронный ресурс]. – Режим доступа: https://habr.com/ru/company/otus/blog/518128/.
- ⚪️ VxLAN фабрика. Часть 3 [Электронный ресурс]. – Режим доступа: https://habr.com/ru/company/otus/blog/519256/.
- ⚪️VxLAN фабрика. Часть 4. Multipod [Электронный ресурс]. – Режим доступа: https://habr.com/ru/company/otus/blog/526628/.
- ⚪️ VxLAN фабрика. Часть 5 [Электронный ресурс]. – Режим доступа: https://habr.com/ru/company/otus/blog/551854/.
- ⚪️Внедрение Multicast VPN на Cisco IOS (часть 1 — знакомство с Default MDT) [Электронный ресурс]. – Режим доступа: https://habr.com/ru/post/528120/.
- ⚪️ Сети для самых маленьких. Часть девятая. Мультикаст [Электронный ресурс]. – Режим доступа: https://habr.com/ru/post/217585/.
- Руководство по дизайну IP-фабрики на коммутаторах Eltex
https://eltexcm.ru/assets/files/site/3686/ip_fabric_design_guide.pdf

### На чем практиковать?
- EVE-NG Community version, она ставится легко и особо проблем с ней нет. Но есть определённые ограничения (нельзя "на горячую" подключать кабеля, например это из того что не нравится мне : ). сравнение фич с платной версией тут - https://www.eve-ng.net/index.php/features-compare/

- Второй вариант - pnetlab ( https://www.pnetlab.com/pages/main ) это форктуная версия профессиональной EVE NG, но с установкой могут возникнуть "нюансики". Я бы рекомендовал pnetlab попробовать - если приготовите её хорошо это будет хорошим подспорьем в будущем, за пределами данного курсаhttp://ciscomaster.ru/content/chto-takoe-vxlan
## Образы для скачивания
 - Например ариста https://labhub.eu.org/addons/qemu/Arista%20vEOS/

    ## Instraction for beginners work with GITHUB
   
### 1. Подготовьте файлы проекта <br> <br>
Добавьте все текущие файлы папки ai_project в индекс Git:bashgit add .

### 2. Сделайте первый коммит <br> <br>
Создайте базовую точку отсчета для вашего проекта:bashgit commit -m "Initial commit"
(Если Git напишет, что нечего коммитить, значит, ваша папка ai_project пуста. Просто создайте в ней любой файл, например, через команду touch README.md, а затем повторите git add . и git commit).
### 3. Создайте тег заново <br> <br>
Теперь, когда коммит существует, команда сработает без ошибок:bashgit tag -a v1.0.0 -m "The first stable local version 1.0"
### 4. Отправьте всё на GitHub <br> <br>
Загрузите ваш код вместе с созданным тегом на сайт:bashgit push -u origin main --tags

