---
{"dg-publish":true,"dg-enable-search":true,"dg-show-tags":true,"dg-permalink":"infrastructure/openwrt-router-bringup/","url":"https://notes.kazakov.xyz/infrastructure/openwrt-router-bringup/","title":"Первичная настройка OpenWrt-роутера","date":"2026-05-16","status":"published","tags":["infra/openwrt","infra/router","infra/firewall","infra/dns","infra/vpn","content/public","public"],"source":"https://openwrt.org/","permalink":"/infrastructure/openwrt-router-bringup/","dgEnableSearch":true,"dgShowTags":true,"dgPassFrontmatter":true}
---


# OpenWrt-роутер: почему, как выбрал, как настроил

> **TL;DR:** OpenWrt — Linux-платформа для роутеров, превращающая «железку» в гибкий сетевой узел. У меня — Cudy WR3000P на OpenWrt **25.12.4 stable** с podkop (VLESS VPN), b4 (DPI-обход), domain-routed DNS на sing-box, тремя сетевыми сегментами (LAN/Guest/IoT) и двухуровневым watchdog. Статья разбита на фазы: выбор и прошивка железа → фаза 1 (безопасность) → фаза 2 (VPN+DNS+storage) → диагностика и сопровождение → production-readiness чеклист.
>
> **Требования:** OpenWrt **24.10+** (статья опирается на `apk` как пакетный менеджер). На 23.05 LTS и ниже — заменять `apk add` на `opkg install` во всех командах установки.
>
> **Out of scope:** IPv6 (в моей сети есть `wan6`/`dhcpv6-client`, но конфиги фаз 1-2 написаны для IPv4 — IPv6-routing и DPI-обход для v6-трафика требуют отдельного разбора, не вошло в эту статью).
>
> **Living document.** Изначально стек писался на SNAPSHOT, сейчас всё переехало на **25.12.4 stable** — основной поток актуализирован под stable, специфика SNAPSHOT остаётся в спойлерах для тех, кому это релевантно.

## 1. Зачем OpenWrt и почему это моё решение

Отличие OpenWrt от проприетарных прошивок в одном: можно поставить любой совместимый софт и собрать нужное именно тебе решение.

В моём регионе ряд популярных сервисов недоступен или работает нестабильно. Решается это программно — комбинация инструментов:

- **[podkop](https://github.com/itdoginfo/podkop)** — динамическое туннелирование трафика к конкретным доменам через XRAY VLESS VPN
- **[b4](https://github.com/DanielLavrushin/b4)** — обход DPI для стабильной работы больших объёмов трафика

Со стоковой прошивкой такую гибкость не получишь.

## 2. Как я выбирал роутер

Начал с Netis N6, но через несколько месяцев столкнулся с проблемами стабильности — процессор и RAM были узким местом при одновременной работе podkop и b4. Под нагрузкой sing-box съедал всю доступную память, дёргал dnsmasq, DNS пропадал. 16 MB flash не давали места под пакеты. Стало понятно, что нужен апгрейд железа — и заодно перебрать критерии выбора с нуля.

<details>
<summary><strong>Детали проблем на Netis N6 (почему пришлось менять)</strong></summary>

**Процессор (MT7621, 880 MHz, quad-core):**
- Не справлялся с одновременным выполнением нескольких динамических задач
- Когда podkop обновлял nftset, а b4 обходил DPI одновременно → CPU 100% → зависание скриптов

**RAM (256 MB DDR3):**
- При росте cache.db sing-box → вся доступная память → ENOSPC → падал sing-box → падал dnsmasq → DNS пропадал
- Требовал разнесения cron по минутам, чтобы избежать OOM

**Flash (16 MB):**
- Конфиги растут, логи на самом роутере занимают место
- Обновления пакетов требуют свободного места → часто были проблемы с `opkg install`

Cudy WR3000P эти проблемы полностью закрывает: 512 MB RAM, 128 MB Flash, новый процессор не запыхивается.

</details>

Из опыта с N6 задал себе четыре требования к замене:

1. **Flash ≥ 128 MB** — чтобы не плясать вокруг каждого `opkg install` и хранить конфиги/пакеты без USB-костыля на старте
2. **Свежий SoC (MT7981 / Filogic 820)** — новое поколение MediaTek заметно лучше тянет nftables и параллельные динамические задачи
3. **USB-порт** — формально опционально, но фича очень полезная: persistent cache sing-box, geo-базы, крупные бинари, бэкапы. В моём стеке использую — без него стек заработает, но flash изнашивается заметно быстрее, а часть удобств (eженедельные снапшоты, обновляемые бинари с github) теряется
4. **Wi-Fi 6 (AX)** — не ради хайпа, а потому что 802.11ac уже узкое место для современных клиентов

И один обязательный мета-критерий перед покупкой: сверка модели **и конкретной ревизии** с официальным [Table of Hardware OpenWrt](https://openwrt.org/toh/start). У многих вендоров (Cudy в этом смысле особенно показателен — см. [список по Cudy](https://openwrt.org/toh/hwdata/cudy/start)) под одним маркетинговым названием существуют v1/v2/v3 с разными SoC и разным статусом поддержки. Купить "ту же модель" не глядя на ревизию — самый быстрый способ получить кирпич или несовместимое железо.

Дальше — bang-for-buck подбор. Не искал premium-железо за оверпрайс и не брал самый дешёвый MT7621 ради экономии. **Cudy WR3000P** попал в нужное окно: MT7981B + 512 MB RAM + 128 MB Flash + USB 2.0 + AX3000, ~5200₽ на маркетплейсе "дикие ягоды" на момент покупки. По соотношению характеристики/цена ни Xiaomi, ни TP-Link, ни GL.iNet не дали лучшего варианта в этом сегменте.


![Cudy WR3000P vs Cudy WR3000H vs Cudy WR3000E Cudy WR3000](../../assets/openwrt-router-bringup/cudy-wr3000p-comparing.png "Cudy WR3000P v1 — модель, на которой собрана конфигурация в этой статье")
> [YouTube comparing](https://www.youtube.com/watch?v=ugDx4x1M1Kg)


<details>
<summary><strong>Почему именно эти четыре критерия (а не «больше антенн» или «гигабит на WAN»)</strong></summary>

Все четыре — про то, что реально упирается на нагрузке. 

1. **Flash ≥ 128 MB**: 16 MB у Netis регулярно давал `No space left on device` на `opkg install` — танцы с overlay и USB. 
2. **Свежий SoC (MT7981B vs MT7621)**: не «больше герц», а другая архитектура — заметно лучше тянет параллельные nftables-операции и flow offloading. 
3. **USB-порт**: опционально, но очень удобно — sing-box cache на tmpfs исчезает при reboot и каждый старт тянет rule-sets с github; крупные бинари (b4, обновлённый sing-box) растут и съедают flash; еженедельные бэкапы куда-то надо складывать. Без USB всё работает, но flash изнашивается быстрее и стек тяжелее переживает sysupgrade. USB 3.0 vs 2.0 — не критично, пишем мелкими блоками. 
4. **Wi-Fi 6 (AX)**: не про peak speed, а про OFDMA/MU-MIMO — заметно лучше держит 10+ одновременных клиентов.

</details>

<details>
<summary><strong>Подводный камень: ревизии Cudy (и почему ToH — не формальность)</strong></summary>

У Cudy под одним именем модели легко встречаются v1, v2, v3 — с разным SoC, разным объёмом памяти и разным статусом в OpenWrt. Реальные сценарии, которые я видел:

- **v1 supported, v2 — нет.** Производитель сменил SoC ради удешевления, OpenWrt-сборка под v1 не подходит, под v2 портов ещё нет.
- **Одинаковое имя на коробке, разная плата внутри.** Определить ревизию можно только по этикетке снизу (обычно мелким шрифтом «HW Ver: v2.0») или по MAC-префиксу. На листинге маркетплейса этого может вообще не быть.
- **«Supported» в ToH ≠ «работает из коробки».** У некоторых моделей статус — community build с ручной сборкой sysupgrade. Это не плохо, но это сразу +1 шаг в bring-up.

Поэтому правильный порядок действий перед покупкой:
1. Открыть [ToH по бренду](https://openwrt.org/toh/start), найти модель.
2. Проверить колонки `Supported Current Rel.` и `Device Type`.
3. Если есть несколько ревизий — выяснить, **какая именно** будет в коробке у конкретного продавца (написать в чат, посмотреть фото этикетки).
4. Прочитать device page — там часто есть нюансы (нужна ли U-Boot модификация, есть ли recovery-режим, какой метод первичной прошивки).

Пропускать этот шаг — экономия 15 минут ценой риска получить железку, которую придётся возвращать.

</details>

<details>
<summary><strong>Что отверг и почему</strong></summary>

**Xiaomi / Redmi AX-серия.**
Цена привлекательная, но закрытый bootloader и регулярные сложности с прошивкой OpenWrt — recovery через TFTP с патчингом, риск кирпича при ошибке. Для роутера, который должен работать «поставил и забыл», лишний риск.

**TP-Link Archer AX-серия.**
В community-поддержке OpenWrt есть, но конкретно нужные модели либо стоили дороже Cudy при тех же характеристиках, либо имели худший SoC (Broadcom без открытых драйверов).

**GL.iNet Flint / Beryl.**
OpenWrt из коробки + хорошая поддержка, но цена 2-3× выше Cudy за сопоставимое железо. Платишь за бренд и preinstalled GL UI, который всё равно снесёшь под чистый OpenWrt.

**И общий минус ко всем альтернативам — доступность.**
Не все модели стабильно представлены в конкретном городе/регионе. Что-то приходится ждать неделями по доставке из другого города, что-то не привозят вообще или возят серым импортом без гарантии. Cudy WR3000P на момент покупки был в наличии локально с быстрой доставкой — и для железки, которая должна заехать, прошиться и работать, это тоже считается.

</details>

## 3. Как прошить роутер на примере Cudy WR3000P

Прошивка Cudy на OpenWrt — двухшаговая процедура: сток-прошивка подписана и произвольный sysupgrade-файл не примет, нужна промежуточная подписанная прошивка, которая снимает проверку. После неё можно лить чистый OpenWrt:

1. **Сток Cudy** → залить **cudy-signed transitional** через стандартный web UI
2. **Transitional** → залить **OpenWrt sysupgrade** через UI уже новой прошивки
3. **OpenWrt** → войти по SSH, доустановить LuCI (на SNAPSHOT — в stable уже в образе), поднять временный wifi, перенести роутер на постоянное место и настраивать дальше «по воздуху»

**Сейчас я на stable — `OpenWrt 25.12.4` (r32933-4ccb782af7).** Релевантный image — `openwrt-mediatek-filogic-cudy_wr3000p-v1-squashfs-sysupgrade.bin` со страницы релиза. Device page как точка входа в документацию: [openwrt.org/toh/hwdata/cudy/cudy_wr3000p_v1](https://openwrt.org/toh/hwdata/cudy/cudy_wr3000p_v1).

Изначально статья писалась на SNAPSHOT (`r34310-8de418547f`, билд 2026-05-05) ради свежих wifi-драйверов mt76, но после стабилизации MediaTek-стека в 25.12 я переехал на stable — sysupgrade-апдейты предсказуемее, LuCI в образе из коробки. Контекст для тех, кто всё ещё на SNAPSHOT, оставлен в спойлере ниже.

**В stable LuCI уже в образе** — можно пропустить SSH-этап с `apk add luci` после прошивки. В SNAPSHOT LuCI собирается отдельно (минимальный образ), там план bring-up другой (см. спойлер с чеклистом).

<details>
<summary><strong>Почему SNAPSHOT, а не stable (и чем за это платишь)</strong></summary>

Если в двух словах — SNAPSHOT даёт свежее ядро и драйверы ценой потери удобств. Подробнее по пунктам:

| Параметр | Stable Release | SNAPSHOT |
|----------|----------------|----------|
| Стабильность | Протестирован, production-ready | Bleeding edge, экспериментальный |
| Обновления | Security + bugfix внутри ветки | Ежедневные сборки из master |
| LuCI Web UI | В образе по умолчанию | Обычно отсутствует, ставится руками |
| Kernel и драйверы | Старее, но проверенные | Свежие, с актуальными wifi-фиксами |
| Совместимость пакетов | Стабильная между минорными апдейтами | Может сломаться между сборками |
| Поддержка нового железа | Догоняет позже | Первым делом появляется здесь |
| Sysupgrade сохраняет пакеты | Да (внутри ветки) | Нет — каждый upgrade ставит весь стек заново |
| Размер образа | Полный (core + LuCI + утилиты) | Минимальный (только core) |
| Откат на предыдущую версию | Через архив релизов | Старый билд может уже исчезнуть с зеркал |
| Рекомендуется для | Домашний роутер «поставил и забыл» | Тестинг, dev, power-users со своим bootstrap |

**Конкретно по Cudy WR3000P v1**

В WR3000P v1 стоит свежий MediaTek MT7981B / Filogic 820, и исторически MediaTek wifi-стек заметно дорабатывается в master раньше, чем эти патчи доезжают до stable. На практике это:

- свежие `mt76` wifi-драйверы (поправленные таймеры, лучшее scheduling)
- выше пропускная способность по воздуху на тех же клиентах
- более вменяемый roaming между 2.4 и 5 ГГц
- меньше случайных wifi-крашей под нагрузкой
- более полная поддержка hardware offload (NAT, flow accelerator)

Цена та же: stable надёжнее как «поставил и забыл», SNAPSHOT — для тех, кто готов держать свой `bootstrap.sh` и пересобирать конфиг при каждом upgrade. У меня этот скрипт есть, поэтому выбор очевиден. Если bootstrap-скрипта нет — бери stable, по wifi-перфомансу разница не настолько драматична, чтобы платить за неё временем на каждый апдейт.

</details>

<details>
<summary><strong>Пошаговый чеклист прошивки (на основе сообщества + личный опыт)</strong></summary>

Базовая канва шагов взята из подробной [инструкции на 4pda по WR3000S v1](https://4pda.to/forum/index.php?showtopic=1087457&view=findpost&p=141588931) и адаптирована под WR3000P. По шагам процедура идентична — отличаются только имена бинарей.

**Шаг 0 — подготовка**

Скачать **два файла** и положить в одну папку:

1. `cudy_wr3000p-v1-sysupgrade_*.bin` — cudy-signed transitional (ищется по ToH device page WR3000P v1 / OpenWrt forum / релизы maintainer'а в gh)
2. `openwrt-mediatek-filogic-cudy_wr3000p-v1-squashfs-sysupgrade.bin` — целевой OpenWrt SNAPSHOT с downloads.openwrt.org

Подключить роутер кабелем LAN-LAN к ноуту. Не по wifi: на одном из шагов он отвалится. Пройти стартовый wizard стока, если ещё не проходил.

**Шаг 1 — заливка transitional**

1. Открыть стоковый web UI: `http://192.168.10.1/cgi-bin/luci/admin/panel`
2. Перейти в раздел «Локальное обновление» / Firmware upgrade
3. Выбрать `cudy_wr3000p-v1-sysupgrade_*.bin`, нажать «Продолжить»
4. Дождаться прошивки и автоперезагрузки. IP роутера изменится → `192.168.1.1`

![Стоковый web UI Cudy — раздел Firmware Upgrade](../../assets/openwrt-router-bringup/cudy-stock-firmware-upgrade.png "Раздел Firmware Upgrade в стоковом UI Cudy WR3000P — сюда загружается cudy-signed transitional bin")

**Шаг 2 — заливка OpenWrt**

1. Открыть новый адрес: `http://192.168.1.1/cgi-bin/luci/admin/system/flash`
2. Пароль может не спрашиваться — `Enter` через пустое поле
3. В разделе «Установить новый образ» выбрать `openwrt-mediatek-filogic-cudy_wr3000p-v1-squashfs-sysupgrade.bin`
4. **Снять галочку «Сохранить настройки»** — пытаться унаследовать конфиг от transitional бессмысленно, будут конфликты
5. «Продолжить» → дождаться прошивки и автоперезагрузки

![LuCI после transitional — страница flash для заливки OpenWrt sysupgrade](../../assets/openwrt-router-bringup/luci-flash-openwrt-sysupgrade.png "Раздел System → Flash в transitional LuCI — сюда загружается openwrt sysupgrade.bin, важно снять галочку Keep settings")

**Шаг 3 — bring-up: вынести роутер на место и перейти на wifi**

Тут SNAPSHOT диктует немного другой порядок, чем привычный «открыл LuCI и настроил». Идея простая: сделать минимум по SSH прямо на столе, потом физически перенести роутер туда, где он будет жить, и доделать всё уже по воздуху.

1. Подключаюсь по SSH с ноута (всё ещё LAN-LAN):

   ```sh
   ssh root@192.168.1.1
   # пароль пустой, сразу:
   passwd
   ```

   ![SSH-баннер OpenWrt — Wireless Freedom ASCII-арт](../../assets/openwrt-router-bringup/openwrt-ssh-banner.png "Первый коннект по SSH после прошивки OpenWrt — приветственный баннер с известным W I R E L E S S   F R E E D O M ASCII")

2. Ставлю LuCI, чтобы дальше не плясать через UCI вручную:

   ```sh
   apk update
   apk add luci luci-ssl
   service uhttpd restart
   ```

3. Поднимаю **дефолтный wifi без пароля** — это временный канал управления, не для постоянной работы. В LuCI: `Network → Wireless → Enable` для обоих радио (2.4 + 5 ГГц). SSID можно оставить дефолтные `OpenWrt` / `OpenWrt-5G`, security — `No Encryption`. Это нормально на 15 минут в собственной квартире, но **сразу после переноса** этот wifi выключаю и пересоздаю с WPA2/WPA3.

4. Выключаю питание, **переношу роутер в прихожую** — туда, где из стены выходит провод провайдера.

5. Подключаю провод провайдера в WAN-порт роутера. В большинстве случаев у провайдера DHCP — поднимается само. Если PPPoE/static — настраивается чуть позже из LuCI, проблема не блокирующая.

6. Включаю роутер. С ноута в комнате цепляюсь к открытому SSID `OpenWrt-5G`, открываю `http://192.168.1.1` — и дальше всё делаю «удалённо», без таскания кабеля LAN-LAN через всю квартиру.

7. **Сразу после того, как удалённый доступ заработал**, первое действие — выключить открытый wifi и пересоздать защищённый. Открытая сеть на роутере, торчащем в прихожей, — это окно на 15 минут, не больше.

**Шаг 4 — мелочи после прошивки**

WAN LED после прошивки часто не горит при живом интернете. Лечится в `System → LED Configuration → Add LED action`:

```text
Name:        WAN_Status
LED name:    white:wan-online   # имя зависит от модели, может быть другое
Trigger:     Network Device Activity (kernel: netdev)
Device:      wan
Trigger Mode: Link On
```

Save & Apply.

</details>

<details>
<summary><strong>TFTP-recovery: когда понадобился (и как делать)</strong></summary>

Stock-прошивка через web UI справляется в 95% случаев. TFTP-recovery понадобился мне один раз — когда **откатывался на сток** для теста гарантии. Сток-прошивку OpenWrt-овский LuCI отказывается заливать как valid sysupgrade (это не sysupgrade-формат), поэтому путь только один — через U-Boot recovery по TFTP.

**⚠️ Параметры recovery отличаются по моделям и ревизиям.** Процедура ниже — типовая для Cudy на MT7981. Перед началом обязательно сверь с **device page твоей конкретной модели на OpenWrt ToH**:

- recovery-IP сервера (у Cudy MT7981 обычно `192.168.1.88`, но у других моделей может быть `192.168.10.10` или другое)
- имя файла, ожидаемое bootloader'ом (часто `recovery.bin`, но не всегда)
- метод входа в recovery (зажим reset перед питанием, hold-during-boot — варианты разные)

Если в device page нет recovery-процедуры — лучше **не лезь в TFTP вслепую**, можно получить кирпич. Спроси на forum.openwrt.org или в OpenWrt-Telegram-чате для своей модели.

Кратко типовая процедура для Cudy MT7981 (с поправкой на твою модель!):

1. Скачать сток-прошивку с сайта Cudy под точную модель и ревизию
2. Поднять на ноуте TFTP-сервер (`tftpd-hpa` / `Tftpd64` на windows), положить файл в его рут с коротким именем (`recovery.bin`)
3. Назначить ноуту статический IP `192.168.1.88/24`
4. Подключиться к роутеру LAN-LAN
5. Зажать reset, подать питание, держать 8-10 секунд, пока LED не замигает в recovery-режиме
6. Роутер сам качает `recovery.bin` с `192.168.1.88` и шьётся

Официальный гайд от Cudy — [тут](https://www.cudy.com/ru-ru/blogs/faq/как-восстановить-cudy-маршрутизатор-из-openwrt-прошивки-до-официальной-прошивки-cudy).

Это же recovery — спасательный круг, если transitional или OpenWrt sysupgrade встанут криво и роутер откажется грузиться. Поэтому я бы сразу при покупке проверил, что TFTP-recovery работает на твоей ревизии: один раз потратить 10 минут на dry-run спокойнее, чем разбираться в проде, когда сеть лежит.

</details>

## 4. Базовая настройка (фаза 1: безопасность)

Цель фазы — закрыть периметр и обеспечить безопасный канал управления, прежде чем поднимать сервисы. На свежем OpenWrt из коробки доступ — root без пароля по SSH на стандартном порту, LuCI на всех интерфейсах, никаких сегментов сети. Это надо менять **в первые 15 минут** после первой загрузки, ещё до того, как роутер уехал в прихожую.

**⚠️ Смена LAN-IP по ходу фазы.** Дефолтный OpenWrt LAN-IP — `192.168.1.1`, с него ты работаешь после прошивки. Все примеры в этой и следующих главах используют **`192.168.10.1`** — он назначается в подразделе 4.4 (Сетевая сегментация). После применения 4.4 ты потеряешь текущее соединение, придётся переподключиться к роутеру по новому IP. Это нормально и ожидаемо. Если ты ещё в bring-up (раздел 3) и видишь команды с `192.168.10.1` — мысленно подставляй `192.168.1.1` до завершения 4.4.

**⚠️ Порядок применения ≠ порядок чтения.** Подразделы пронумерованы логически (SSH → LuCI → Wi-Fi → сегментация → firewall → system), но 4.2 и 4.3 ссылаются на IP и networks, которых до 4.4 ещё нет. При линейном apply uhttpd упадёт с `EADDRNOTAVAIL` на bind к несуществующему `192.168.10.1`, а wifi-iface'ы `guest`/`iot` молча проигнорируются `wifi reload` — networks ещё не созданы. Применяй в таком порядке:

1. **4.1** — SSH (на текущем `192.168.1.1`)
2. **4.4** — сегментация (меняет LAN-IP → переподключение по `192.168.10.1`)
3. **4.5** — firewall (zones зависят от networks из 4.4)
4. **4.3** — Wi-Fi (iface'ы цепляются к networks из 4.4)
5. **4.2** — LuCI HTTPS (`listen` на новый IP из 4.4)
6. **4.6** — system (timezone / NTP / hostname; порядок не важен)

Читать удобно сверху вниз, применять — по списку выше.

**Пакеты в этой главе:**

```sh
apk add px5g-mbedtls libustream-mbedtls
```

Всё остальное — конфигурация уже стоящих сервисов (dropbear, uhttpd, hostapd, firewall, dnsmasq). Generic-значения в примерах (IP, SSID, порт SSH) — придумал для статьи, меняй под себя.

### 4.1 SSH (dropbear)

**dropbear** — лёгкая реализация SSH-сервера, по умолчанию стоит на OpenWrt. Из коробки слушает на :22 на всех интерфейсах, root без пароля.

**Какую проблему решает.** Свежий OpenWrt пускает кого угодно по SSH без пароля. Это нужно закрыть **первым** — до WAN и Wi-Fi. Иначе через 5 минут после подключения WAN роутер начнут долбить bot'ы со всего интернета.

**Минимальная настройка.** `/etc/config/dropbear`:

```
config dropbear 'main'
    option PasswordAuth 'off'         # только key auth
    option RootPasswordAuth 'off'     # отдельный флаг для root, обязательно
    option Port '22022'                # любой нестандартный, не 22
    option Interface 'br-lan'          # слушать только на LAN-bridge
    option IdleTimeout '600'           # 10 мин — kill зависших сессий
    option MaxAuthTries '3'            # лимит попыток в одном handshake
    option GatewayPorts 'off'          # запретить reverse-tunnels наружу
```

Публичный ключ — в `/etc/dropbear/authorized_keys` (по строке на ключ). Перезапуск — `service dropbear restart`. На клиенте удобно прописать alias в `~/.ssh/config`:

```text
Host home-router
    HostName 192.168.10.1
    Port 22022
    User root
    IdentityFile ~/.ssh/id_ed25519_router
    IdentitiesOnly yes
```

Ключевые опции и почему они важны:

- **`Interface 'br-lan'`** — закрывает порт SSH снаружи на уровне сокета, а не firewall-правила. Это второй рубеж: даже если firewall случайно перепишется, sshd физически не примет коннект с wan.
- **`IdleTimeout '600'`** — зависшие сессии без таймаута живут вечно, занимая память и pty. На роутере с 512 MB RAM это копится.
- **`GatewayPorts 'off'`** — если злоумышленник всё-таки получил root, эта опция запрещает делать reverse-tunnel наружу, превращая роутер в плацдарм.

Если есть физический доступ к UART — выключить `ttylogin` (разобрано в 4.6).

<details>
<summary><strong>Что часто забывают: host-keys, brute-force, выбор клиентского ключа</strong></summary>

1. **Регенерация host keys после прошивки.** Dropbear генерит их при первом старте, но при `sysupgrade` с сохранением конфига они могут уехать. После каждой прошивки полезно проверить fingerprint вручную (`dropbearkey -y -f /etc/dropbear/dropbear_ed25519_host_key`) и сравнить с тем, что в `~/.ssh/known_hosts` клиента.
2. **Brute-force защита на firewall-уровне.** Привязка к LAN снимает риск с wan. Но если когда-нибудь захочется временно открыть SSH с интернета (форвардинг порта на VPN-туннеле) — нужен `banip` или nftables rate-limit (`limit rate 5/minute` + drop above).
3. **Использовать ed25519, не RSA.** Меньше ключ, быстрее handshake, лучше криптография. RSA 4096 — приемлемо, но не нужно.

</details>

### 4.2 LuCI HTTPS на LAN-зоне

**Какую проблему решает.** Из коробки LuCI на uhttpd слушает на `0.0.0.0` — торчит наружу на WAN. Плюс HTTPS не работает без двух недоустановленных пакетов (uhttpd биндит только `:80`, `redirect_https='1'` показывает ошибку). Цель — поднять HTTPS и привязать его только к LAN-зоне.

**Быстрая установка.** Два пакета (уже в `apk add` главы):

```sh
apk add px5g-mbedtls libustream-mbedtls
service uhttpd restart
```

`px5g-mbedtls` генерирует self-signed cert при первом старте, `libustream-mbedtls` — TLS-обёртка для uhttpd. После рестарта uhttpd начнёт слушать `:443`.

**Минимальная настройка.** Привязать LuCI к LAN-IP в `/etc/config/uhttpd`:

```
list listen_http  '192.168.10.1:80'
list listen_https '192.168.10.1:443'
option redirect_https '1'
```

`redirect_https='1'` решает только редирект `http→https`. Сама привязка к `0.0.0.0` означает «слушаем на всех интерфейсах, включая wan». Два пути закрыть:

1. **Привязка listen к LAN-IP** (как в примере выше) — hard guarantee на уровне сокета, не зависит от firewall.
2. **`0.0.0.0` + firewall-правило** `Allow-HTTPS-LAN` для `src='lan'`, плюс `wan→input REJECT` (стандартный default).

Первый вариант надёжнее: даже если firewall-правило случайно пропадёт при rebuild конфига, LuCI физически не примет коннект из wan.

Альтернатива пакетам mbedtls — `px5g-wolfssl` + `libustream-wolfssl`. На SNAPSHOT обычно дефолт mbedTLS, его и использую.

### 4.3 Wi-Fi: три SSID под три класса клиентов

Конфигурация в `/etc/config/wireless` с тремя отдельными `wifi-iface` под разные классы устройств: главная LAN-сеть (5 ГГц), гостевая (2.4 ГГц), IoT (2.4 ГГц).

**Какую проблему решает.** Один SSID «для всего» — удобно, но небезопасно: гости и IoT попадают в одну сеть с домашним NAS и ноутбуком. Разделение SSID + разное шифрование под каждый класс закрывают эту брешь, плюс учитывают что IoT часто не умеет ни 5 ГГц, ни WPA3. **Дополнительно:** `isolate='1'` блокирует L2-видимость между клиентами в пределах одного SSID — защита от ARP-spoofing и lateral movement внутри гостевой/IoT сетей.

**Минимальная настройка.** В `/etc/config/wireless` — два `wifi-device` (radio0/2.4 ГГц + radio1/5 ГГц) и три `wifi-iface` (lan_5g на radio1, guest_2g + iot_2g на radio0). Шифрование разное: `sae-mixed` для LAN/Guest (WPA2+WPA3), `psk-mixed` для IoT (WPA+WPA2, для совместимости со старыми гаджетами). Перезапуск — `wifi reload`.

<details>
<summary><strong>Полный конфиг <code>/etc/config/wireless</code></strong></summary>

```
config wifi-device 'radio0'             # 2.4 ГГц (Guest + IoT)
    option band '2g'
    option channel '1'
    option htmode 'HE20'
    option country 'RU'                 # регулятор для твоего региона
    option txpower '23'                 # мощность, дБм

config wifi-device 'radio1'             # 5 ГГц (LAN)
    option band '5g'
    option channel '149'                # фиксированный канал (не auto на 5G)
    option htmode 'HE80'
    option country 'US'                 # расширенные каналы
    option txpower '23'

config wifi-iface 'lan_5g'
    option device 'radio1'
    option network 'lan'
    option mode 'ap'
    option ssid 'YourLAN-5G'
    option encryption 'sae-mixed'       # WPA2/WPA3
    option key '<длинный пароль>'
    option disabled '0'

config wifi-iface 'guest_2g'
    option device 'radio0'
    option network 'guest'
    option mode 'ap'
    option ssid 'YourGuest'
    option encryption 'sae-mixed'       # единообразное с LAN
    option key '<длинный пароль>'
    option isolate '1'                  # L2-изоляция клиентов
    option disabled '0'

config wifi-iface 'iot_2g'
    option device 'radio0'
    option network 'iot'
    option mode 'ap'
    option ssid 'YourIoT'
    option encryption 'psk-mixed'       # WPA + WPA2 — совместимость со старыми IoT
    option key '<длинный пароль>'
    option isolate '1'
    option disabled '0'
```

</details>

**Шифрование по классам:**
- **Главная сеть (LAN, 5 ГГц):** `sae-mixed` (WPA2-PSK + WPA3-SAE с PMF). Свежие клиенты идут WPA3, старые остаются на WPA2. Никаких legacy WPA или TKIP.
- **Гостевая (2.4 ГГц):** `sae-mixed` + `isolate='1'` — гости тоже заслуживают нормального шифрования, плюс L2-изоляция. **Caveat:** если у тебя часто меняются гости с очень старыми устройствами (Android <5, iPhone <6s) — SAE/PMF может давать проблемы. Тогда ставь `psk2` (как IoT), это надёжнее.
- **IoT (2.4 ГГц):** `psk-mixed` (WPA + WPA2). Старые IoT-гаджеты часто поддерживают только WPA (оригинальный стандарт), не WPA2. `psk-mixed` даёт совместимость со старым железом, которое не может в WPA2. `isolate='1'` обязательно: компрометированный гаджет не должен видеть соседей по той же сети.

**Channel и txpower:**
- **2.4 ГГц:** `channel '1'` (фиксированный, не `auto`) — избегаешь скачков на соседних каналах. `cell_density '0'` для типичной квартиры; значения 1/2/3 повышают порог входа клиентов в плотной обстановке.
- **5 ГГц:** **фиксированный канал** (149/153/157, не `auto`) — auto-режим на 5 ГГц часто скачет на DFS-событиях, выбивая клиентов. Фиксированный канал стабильнее.
- **txpower:** одинаковая на обоих радио (23 дБм в примере) — симметричный coverage.

**Country code:** `RU` на 2.4 ГГц, `US` на 5 ГГц — последний открывает U-NII-1/2A (36-64) и U-NII-3 (149-165), снимает DFS-проблемы в плотной городской обстановке. Это **grey area**: юридически корректно — реальный код страны; де-факто RU и US пересекаются по большинству диапазонов.

### 4.4 Сетевая сегментация: LAN, Guest, IoT

Три L3-сегмента с отдельными bridge'ами, IP-диапазонами и DHCP-пулами в `/etc/config/network` + `/etc/config/dhcp`.

**Какую проблему решает.** Гость и IoT — разные классы угроз, но и тот, и другой *не доверенные*. Скомпрометированный IoT-гаджет (а они регулярно ломаются массово через CVE в популярных прошивках) в плоской сети видит NAS, ноутбук, телевизор — всё. Сегментация даёт техническую границу, через которую инцидент не растекается.

**Минимальная настройка.** В `/etc/config/network` — три interface'а (lan/guest/iot) + два bridge'а для guest/iot без физических портов (`bridge_empty='1'`, только wifi-iface'ы цепляются через `network=…`). LAN на `192.168.10.1/24`, guest на `10.20.30.1/24`, iot на `10.40.50.1/24`.

DHCP-пулы для guest и iot — в `/etc/config/dhcp` (lan-пул уже в дефолтном конфиге). `leasetime` короче для guest (мобильные устройства приходят/уходят), длиннее для IoT (камеры/розетки статичные). Без этих блоков клиенты привяжутся к Wi-Fi, но IP не получат — типичная ловушка после `service network restart`.

<details>
<summary><strong>Полный конфиг <code>/etc/config/network</code> + <code>/etc/config/dhcp</code></strong></summary>

`/etc/config/network`:

```
config interface 'lan'
    option device 'br-lan'
    option proto 'static'
    list ipaddr '192.168.10.1/24'
    option ip6assign '60'

config device
    option type 'bridge'
    option name 'br-guest'
    option bridge_empty '1'

config interface 'guest'
    option device 'br-guest'
    option proto 'static'
    option ipaddr '10.20.30.1'
    option netmask '255.255.255.0'

config device
    option type 'bridge'
    option name 'br-iot'
    option bridge_empty '1'

config interface 'iot'
    option device 'br-iot'
    option proto 'static'
    option ipaddr '10.40.50.1'
    option netmask '255.255.255.0'
```

`/etc/config/dhcp` (добавить к существующему lan-пулу):

```
config dhcp 'guest'
    option interface 'guest'
    option start '100'
    option limit '150'
    option leasetime '2h'

config dhcp 'iot'
    option interface 'iot'
    option start '100'
    option limit '100'
    option leasetime '12h'
```

</details>

**⚠️ Применение этого конфига меняет LAN-IP.** После `service network restart` ты потеряешь текущее SSH/LuCI-соединение. Переподключиться нужно по новому LAN-IP:
- SSH: `ssh root@192.168.10.1 -p 22022` (если 4.1 уже применён, порт может быть другой — твой выбор)
- LuCI: `https://192.168.10.1`

DHCP-клиенты в LAN получат новые IP из пула `192.168.10.0/24` после release/renew (на Mac — `Renew DHCP lease` в Network Settings; на Linux — `dhclient -r && dhclient eth0`).

![Схема трёх сетевых сегментов: LAN, Guest, IoT](../../assets/openwrt-router-bringup/network-segments-diagram.png "Топология трёх сегментов с указанием разрешённых направлений forwarding — LAN доверенный, IoT и Guest изолированы")

Логика угроз и правил forwarding'а:

- **LAN (доверенный).** Личные устройства, NAS, ноутбук. Доступ ко всему, включая IoT (нужно с ноута тыкать в умную розетку через её API).
- **Guest (гости).** Чужие люди и устройства. Цель — дать интернет, ничего больше. Никакого доступа в LAN, никакого в IoT.
- **IoT (умные устройства).** Своя сеть, но **не доверенная**: китайские камеры с прошивками 2019 года, hub'ы с включённым telnet-портом. Цель — IoT может в интернет (cloud-API вендора), но **не может в LAN**. LAN при этом может в IoT (нужно управление).

Сама forwarding-матрица закрепляется в firewall (см. следующий подраздел).

### 4.5 Firewall: зонирование + critical defaults

**Какую проблему решает.** OpenWrt firewall (`fw4`/nftables) конфигурируется через зоны и forwarding в `/etc/config/firewall`. Без зон сегментация — это только L3-разделение: гость через шлюз увидит LAN на уровне маршрутизации. Зоны определяют, кто куда может ходить, и закрывают дефолтные дырки, которые в свежем OpenWrt не выставлены (`drop_invalid`, `flow_offloading`).

> **⚠️ Дополни, не заменяй.** Дефолтный `/etc/config/firewall` 25.12.4 уже содержит `Allow-DHCP-Renew`, `Allow-Ping`, `Allow-DHCPv6`, `Allow-MLD`, `Allow-ICMPv6-Input`, `Allow-ICMPv6-Forward`, `Allow-IPSec-ESP`, `Allow-ISAKMP` — оставь как есть, иначе отвалятся IPv6 RA, ping внутри LAN, IPsec passthrough. Из конфига ниже: `config defaults` подправь in-place (добавь `drop_invalid` и выключи `flow_offloading*`), zones lan/wan тоже уже есть — проверь поля и допиши недостающее. Полностью новые блоки — только zones `guest`/`iot` и все `config forwarding`.

**Минимальная настройка.** В `/etc/config/firewall`: в `config defaults` добавить `drop_invalid='1'` и выключить `flow_offloading*` (для b4/sing-box обязательно); добавить zones `guest`/`iot` (input/forward `REJECT`, output `ACCEPT`); добавить пять `config forwarding` (`lan→wan`, `lan→guest`, `lan→iot`, `guest→wan`, `iot→wan`); добавить input-`config rule` для DHCP+DNS в зонах guest и iot, иначе клиенты не получат IP и не зарезолвят имена. После правок — `service firewall restart`.

<details>
<summary><strong>Полный конфиг <code>/etc/config/firewall</code> (дополнение к дефолту)</strong></summary>

```
config defaults
    option syn_flood '1'
    option input 'REJECT'
    option output 'ACCEPT'
    option forward 'REJECT'
    option drop_invalid '1'              # дропать out-of-state пакеты
    option flow_offloading '0'           # выкл — иначе ломает NFQUEUE (для b4/sing-box)
    option flow_offloading_hw '0'

config zone
    option name 'lan'
    list network 'lan'
    option input 'ACCEPT'
    option output 'ACCEPT'
    option forward 'ACCEPT'

config zone
    option name 'wan'
    list network 'wan'
    list network 'wan6'
    option input 'REJECT'
    option output 'ACCEPT'
    option forward 'DROP'
    option masq '1'
    option mtu_fix '1'

config zone
    option name 'guest'
    list network 'guest'
    option input 'REJECT'
    option output 'ACCEPT'
    option forward 'REJECT'

config zone
    option name 'iot'
    list network 'iot'
    option input 'REJECT'
    option output 'ACCEPT'
    option forward 'REJECT'

config forwarding
    option src 'lan'
    option dest 'wan'

config forwarding
    option src 'lan'
    option dest 'guest'                  # LAN может достучаться до Guest (мониторинг)

config forwarding
    option src 'lan'
    option dest 'iot'                    # LAN может управлять IoT (API, конфигурация)

config forwarding
    option src 'guest'
    option dest 'wan'

config forwarding
    option src 'iot'
    option dest 'wan'                    # IoT может в интернет (cloud API)
```

Input-rules для DHCP+DNS в guest/iot:

```
config rule
    option name 'Guest-DHCP'
    option src 'guest'
    option proto 'udp'
    option dest_port '67-68'
    option target 'ACCEPT'

config rule
    option name 'Guest-DNS'
    option src 'guest'
    option proto 'tcp udp'
    option dest_port '53'
    option target 'ACCEPT'

config rule
    option name 'IoT-DHCP'
    option src 'iot'
    option proto 'udp'
    option dest_port '67-68'
    option target 'ACCEPT'

config rule
    option name 'IoT-DNS'
    option src 'iot'
    option proto 'tcp udp'
    option dest_port '53'
    option target 'ACCEPT'
```

</details>

Forwarding-матрица в человеческом виде:

| От \ К | wan | lan | guest | iot |
|--------|-----|-----|-------|-----|
| **lan**   | ACCEPT | — | ACCEPT | ACCEPT |
| **guest** | ACCEPT | REJECT | — | REJECT |
| **iot**   | ACCEPT | REJECT | REJECT | — |
| **wan**   | — | REJECT | REJECT | REJECT |

LAN имеет полный доступ ко всем сегментам для управления и мониторинга. Guest и IoT изолированы друг от друга и от LAN (входящий трафик из LAN на Guest/IoT дропится).

Два дефолта, которых **нет** в свежем `/etc/config/firewall`, и которые нужно включить руками:

- **`drop_invalid='1'`** — дропать пакеты вне established/related state. Закрывает класс connection-tracking обходов.
- **`flow_offloading='0'` и `flow_offloading_hw='0'`** — если использовать NFQUEUE-based инструменты (sing-box, b4, zapret). Если их нет — можно оставить включённым, прирост +200-300 Mbps реальный.

<details>
<summary><strong>Почему flow offloading и NFQUEUE несовместимы — механика</strong></summary>

Без offloading каждый пакет проходит через **netfilter slow path**: conntrack lookup → nft prerouting → forwarding → postrouting → SNAT/DNAT. На каждом этапе пакет можно матчить, перенаправить в `NFQUEUE`, прочитать содержимое, модифицировать, вернуть в стек. На этом и работают DPI-обходчики (`zapret`, `b4`, `xtables-addons`) и любые custom nftables-правила с `queue` target.

Когда включён flow offloading, kernel ведёт себя иначе. После установления соединения (TCP handshake или несколько успешных UDP пакетов) conntrack помечает поток как «offloaded», и дальше пакеты этого conn ловятся в **fastpath** — либо на уровне kernel-хуков (`sw` flow offloading), либо в NPU чипа MediaTek (`hw` flow offloading). Fastpath **обходит netfilter pipeline целиком**: пакет не доходит ни до nft prerouting, ни до NFQUEUE, ни до custom matchers.

Дополнительный коварный момент: offloading включается **не сразу**, а после нескольких пакетов в потоке. DPI-обход успевает сработать на первых пакетах (TLS ClientHello, который и есть мишень для DPI), а дальше поток уходит в fastpath, и продолжение сессии идёт мимо. Со стороны выглядит как «соединение установилось, но через 5-10 секунд отвалилось».

Вердикт:
- Чистый routing без DPI-инструментов → `flow_offloading='1'` (и `_hw='1'` если железо умеет), прирост реальный
- Хоть один NFQUEUE-based инструмент → оба флага в `'0'`, без исключений

</details>

### 4.6 Системные мелочи: hostname, timezone, NTP

**Какую проблему решает.** В `/etc/config/system` живут hostname, timezone, NTP и уровни логирования. Hostname `OpenWrt` палит роутер в DHCP-логе провайдера. Неправильная timezone ломает cron и logrotate (ротация уйдёт по UTC, а не по местному времени). Без NTP время плывёт — критично для TLS-handshake и разбора инцидентов по логам.

**💡 Hostname:** Используй что-то осмысленное вместо `OpenWrt` — `home-router`, `corvus`, `filogic-gate` или свой вариант. Это не только уменьшает риск быть замеченным в логах провайдера, но и облегчает отладку в мультиинтерфейсной среде.

**Минимальная настройка.** `/etc/config/system`:

```
config system
    option hostname 'home-router'        # или свой вариант (не 'OpenWrt')
    option timezone 'MSK-3'              # POSIX TZ для busybox (date, cron)
    option zonename 'Europe/Moscow'      # IANA tzname для glibc-утилит
    option ttylogin '0'                  # без логина на UART
    option log_size '128'                # KB

config timeserver 'ntp'
    option enabled '1'
    option enable_server '0'
    list server '0.openwrt.pool.ntp.org'
    list server '1.openwrt.pool.ntp.org'
    list server '2.openwrt.pool.ntp.org'
    list server '3.openwrt.pool.ntp.org'
```

После правок — `service system restart`.

**LED-индикаторы портов (опционально).** Если у роутера на панели физические LED'ы под порты — их можно привязать к активности через `config led` в том же `/etc/config/system`. У каждой модели свои имена в `/sys/class/leds/` (проверь `ls /sys/class/leds/`), пример конфига для 4×LAN + WAN — в спойлере.

<details>
<summary><strong>Пример блока <code>config led</code></strong></summary>

```
config led 'led_lan1'
    option name 'lan1'
    option sysfs 'white:lan-1'
    option trigger 'netdev'
    option mode 'link tx rx'
    option dev 'lan1'

# аналогично для lan2, lan3, lan4 и wan
```

</details>

Hostname важен не только для self-identification: у некоторых провайдеров он виден в их dhcp-логе, и `OpenWrt` сразу палит, что роутер не из их парка. Минорно, но факт. Timezone — реальная боль при разборе инцидентов через сутки: лог в UTC выглядит совсем иначе, чем твоя голова, привыкшая к MSK.

Про пару `timezone` и `zonename` — это **не дубликаты**, а два разных параметра с разной семантикой:
- `timezone='MSK-3'` — это POSIX TZ-строка, которую читает busybox-стек (`date`, `cron`, скрипты с `tzset`). Без неё системное время — в UTC.
- `zonename='Europe/Moscow'` — это IANA-имя для процессов, которые ищут зону через `/usr/share/zoneinfo/...` (часть утилит, демоны, которые завязаны на полный tzdata). Без неё может отвалиться dnsmasq при некоторых пограничных конфигах, или ntpd-клиент.

Оба параметра задаются вместе. Часто LuCI заполняет их одновременно при выборе зоны через UI.

`option ttylogin '0'` — выключает root-login на UART. Если у роутера есть физический serial-порт (а на Cudy WR3000P и большинстве openwrt-устройств он есть на плате под крышкой), кто-то с отвёрткой и USB-UART получит root-shell без пароля за минуту. Параметр это закрывает: на UART остаётся только consoleline без auto-login. Не важно при домашнем стенде на столе, **критично** для роутера в общем коридоре или на стене подъезда.

## 5. Расширенная настройка (фаза 2: VPN + DNS + storage)

Когда периметр закрыт, можно поднимать функционал. Эта фаза превращает роутер из коробки-маршрутизатора в активный узел: persistent storage на USB, кастомный DNS-резолвер, domain-based VPN-туннелирование, DPI-обход и собственный watchdog. Куски связаны: VPN хочет cache на диске, DNS должен пережить ребут VPN, watchdog должен знать про оба.

**Пакеты в этой главе:**

```sh
apk add kmod-usb-storage kmod-fs-ext4 block-mount e2fsprogs \
        logrotate watchcat
```

Дополнительно: `podkop` и `b4` ставятся собственными install-скриптами с github (см. подразделы 5.3 и 5.4) — это не apk-пакеты.

### 5.1 USB как persistent storage (опционально, рекомендуется)

> **Опционально, но сильно упрощает жизнь.** Стек работает и без USB — sing-box можно поставить из apk-feeds, cache держать на tmpfs, бэкапы выгружать на внешний сервер. У меня USB подключён, и дальше статья предполагает что он есть: это разгружает flash от write-cycles, упрощает sysupgrade (бинари и cache не теряются), даёт куда сложить еженедельные снапшоты. Если USB-порта нет или не хочется его занимать — пути на `/mnt/usb/*` в инструкциях ниже мысленно подменяй на tmpfs / apk-feeds / внешний backup.

USB-накопитель, смонтированный в `/mnt/usb` по UUID — для хранения кеша VPN, geo-баз, бинарей и снапшотов конфигов.

**Какую проблему решает.** OpenWrt живёт в overlay-fs поверх squashfs — flash-память ограничена (100+ MB на современных моделях) и имеет write-cycles. Крупное и часто-пишущееся (cache sing-box, geo-базы, обновляемые с github бинари, регулярные бэкапы) хочется вынести с flash, чтобы не тратить его ресурс. USB-флешка — простой способ это сделать.

**Быстрая установка.**

```sh
apk add kmod-usb-storage kmod-fs-ext4 block-mount e2fsprogs
```

UCI fstab по UUID (не по `/dev/sda1` — он меняется при перетыкании):

```
config mount 'usb'
    option target '/mnt/usb'
    option uuid '<uuid флешки>'        # block info | grep UUID
    option enabled '1'
```

После — `service fstab start` (или `block mount`) и проверка `df -h /mnt/usb`. Файловая система — ext4 (или f2fs для медленных флешек).

**⚠️ USB-флешка — это тоже NAND, и она быстрее умирает, чем кажется.** Бюджетная флешка ($5-10, noname) — это TLC/QLC чипы на 1000-3000 циклов записи, без wear-leveling и без SMART. Write-heavy workload (постоянный debug-log, рабочая БД, online-метрики) **убьёт её за несколько месяцев**: переход в read-only mode, исчезающие файлы, странные I/O ошибки в dmesg. И ты узнаёшь о проблеме, только когда уже сломалось.

Поэтому критично понимать, что класть на USB:

| Тип нагрузки | Подходит для бюджетного USB | Нужно что-то надёжнее |
|---|---|---|
| Бинари (запись при установке) | ✅ | — |
| Geo-базы (обновление раз в день) | ✅ | — |
| Persistent cache sing-box | ✅ | — |
| Snapshot-backup конфигов (раз в неделю) | ✅ | — |
| Online-логи на debug-level | ❌ | ✅ industrial SLC USB / SD high-endurance |
| БД активного сервиса | ❌ | ✅ SSD через USB 3.0 adapter |
| Application metrics с высокой частотой | ❌ | ✅ только через cron-агрегацию |

**Что у меня:** обычная Kingston DataTraveler 32 GB под бинари, geo-базы, sing-box cache (запись 1-2 раза в час), weekly snapshot конфигов. Логи остаются на tmpfs, на флешку едут только в виде архива при ротации. Это даёт нагрузку «несколько MB записи в час» — бюджетная флешка живёт годами.

<details>
<summary><strong>Альтернативы для serious workload</strong></summary>

Если планируется write-heavy задача и бюджетной флешки точно мало:

- **SanDisk High Endurance / Samsung Pro Endurance microSD** через USB-адаптер — рассчитаны на видеорегистраторы, 100000+ часов записи. Хороший компромисс цена/надёжность.
- **Industrial USB-флешки** (Innodisk, Apacer industrial) — SLC NAND, hardware wear-leveling, дороже в разы, надёжнее на порядки.
- **2.5" SSD через USB 3.0 adapter** — overkill для роутера, но если есть лишний SSD на полке — самый надёжный вариант + кратно больше места.

</details>

<details>
<summary><strong>Бонус: USB как сетевое хранилище (Samba / NFS / DLNA)</strong></summary>

Раз флешка воткнута и смонтирована, на ней можно поднять простую сетевую шару — роутер становится маленьким NAS:

- **Samba (SMB/CIFS)** — `samba4-server` + `luci-app-samba4`. Видна как сетевая папка в Finder, Проводнике, `smb://` в Linux. Универсальный выбор для смешанной сети.
- **NFS** — `nfs-kernel-server` + `luci-app-nfs`. Только для Linux/Unix-клиентов, но быстрее и проще по правам.
- **MiniDLNA** — `luci-app-minidlna`. Медиа-сервер для телевизоров/приставок по UPnP/DLNA.

Настройка через LuCI — 5-10 минут. Все три пакета вместе укладываются в 30-50 MB RAM.

**Но предупреждение про write-cycles остаётся.** Хранить семейные фотки и редкие архивы — нормально. Использовать как scratch-диск под торренты или видеомонтаж — нет, бюджетную флешку это убьёт за месяц-два.

</details>

<details>
<summary><strong>Расширение PATH под USB-бинари</strong></summary>

Когда крупные бинари переезжают на USB (`/mnt/usb/usr/bin/`), удобно подцепить их в PATH через `/etc/profile.d/`:

```sh
# /etc/profile.d/10-usb-bin.sh
[ -d /mnt/usb/usr/bin ] && export PATH="$PATH:/mnt/usb/usr/bin"
```

Условие `[ -d ]` важно: если USB вырвали, PATH остаётся системный — никакого «command not found» в неожиданных местах.

Подводный момент: **процессы, запущенные через procd (init.d), `/etc/profile.d/` не читают** — это шелловская инициализация. Для сервисов, ищущих бинарь по имени, прописывай явные пути в init-скрипте или клади symlinks в `/usr/bin/`.

</details>

### 5.1a Swap буфер на USB (опционально, для тонкого RAM)

> **На моём Cudy WR3000P swap не настроен.** 512 MB RAM при стеке `podkop + sing-box + dnsmasq + b4` оставляют ~319 MB available — давления нет, OOM ни разу не случался. Этот раздел релевантен в первую очередь для роутеров с **256 MB RAM** (например, Netis N6 на MT7621), где баланс действительно тонкий. Если у тебя ≥512 MB RAM и стандартный стек — пропускай, либо ставь tiny-вариант ниже как страховку.

**Когда swap нужен.** Сигналы для принятия решения:

| Сигнал | Что делать |
|---|---|
| `free -m` стабильно показывает `available` < 30 % от total | swap полезен |
| В `dmesg \| grep -i oom` уже есть `Out of memory: Killed process` | swap нужен срочно |
| `available` стабильно > 50 % от total, OOM-ов нет | swap не нужен |

Первая линия защиты на этом стеке — не swap, а **`memory_limit: "150MiB"` на b4** (см. 5.4). Это закрывает основной OOM-сценарий, остальные сервисы сами по себе укладываются в RAM.

**⚠️ zRAM сознательно не использую** — компрессированный swap-in-RAM выглядит привлекательно (нет wear на флешку, выше throughput), но в community много репортов крашей на OpenWrt-роутерах с low-RAM, особенно с `lzo-rle` (из-за этого отключён дефолтом в 2020). Swap-on-USB предсказуемее.

**Минусы swap на USB.** USB flash wear (запись на NAND убавляет ресурс); latency spike (UDP-ответ DNS из swap = пауза в десятки ms, клиент может оборвать запрос); USB-зависимость (если флешку выдернули или I/O error — активный swap → kernel panic).

#### Два варианта установки

**Вариант 1 — Tiny emergency swap** (для роутеров с достаточным RAM, как страховка):

```sh
dd if=/dev/zero of=/mnt/usb/swapfile bs=1M count=64
mkswap /mnt/usb/swapfile && swapon /mnt/usb/swapfile
echo "vm.swappiness=1"  >> /etc/sysctl.conf && sysctl -p
echo "swapon /mnt/usb/swapfile" >> /etc/rc.local
```

64 MB + `swappiness=1` — ядро почти никогда не использует swap, но он есть на случай абсолютного OOM. Wear-нагрузка близка к нулю.

**Вариант 2 — Активный swap** (для тонкого RAM 256 MB или нагруженного стека):

```sh
dd if=/dev/zero of=/mnt/usb/swapfile bs=1M count=256
mkswap /mnt/usb/swapfile && swapon /mnt/usb/swapfile
echo "vm.swappiness=10" >> /etc/sysctl.conf && sysctl -p
echo "swapon /mnt/usb/swapfile" >> /etc/rc.local
```

`swappiness=10` — swap трогается только при заполнении RAM на 80–90 %, не превентивно. 256 MB хватает на типовые пики: discovery в b4 (+100–150 MB), `podkop list_update` (+80 MB), одновременный старт сервисов при reboot. Wear при таких настройках — порядка 100–500 MB/мес, бюджетная Kingston живёт 2–3 года.

Проверка после установки: `free`, `swapon -s`, `cat /proc/sys/vm/swappiness`.

**Когда даже swap не спасёт.** Если резидентный набор стабильно > RAM + swap (∼751 MB на конфигурации 495 + 256). Лечение — `memory_limit` на самого толстого потребителя (см. 5.4 про b4), не наращивать swap.

Более продвинутые подходы (`oom_score_adj` через `service_started()`, earlyoom, `vm.min_free_kbytes`, `GOMEMLIMIT` для Go-сервисов, PSI вместо порога `MemFree`) — в отдельной заметке: [[Knowledge/Infrastructure/openwrt-memory-management\|Knowledge/Infrastructure/openwrt-memory-management]].

### 5.2 Бинари с GitHub: musl vs glibc

OpenWrt использует **musl libc**, а не glibc. Соответственно, бинари с github releases должны быть либо статически линкованы, либо собраны под musl.

**Зачем нужно знать.** Если просто скачать «linux-arm64» tarball с release-страницы какого-нибудь sing-box или go-tool, он, скорее всего, слинкован против glibc и на OpenWrt просто не запустится: `not found`, хотя файл прямо перед глазами. Это популярный источник часа потерянного времени при первой попытке поставить что-то «руками».

**Минимальная проверка.** Перед запуском любой бинарь — через `file`:

```sh
file /mnt/usb/bin/sing-box
# хочешь увидеть:
#  - "statically linked"                                    (идеально)
#  - "dynamically linked, interpreter /lib/ld-musl-aarch64.so.1"  (musl-вариант)
# что НЕ подходит:
#  - "interpreter /lib/ld-linux-aarch64.so.1"               (glibc, не запустится)
```

В release-страницах серьёзных проектов обычно есть выбор:

```text
sing-box-1.13.11-linux-arm64.tar.gz        ← glibc dynamic, НЕ подходит
sing-box-1.13.11-linux-arm64-musl.tar.gz   ← статически линкованный, подходит
```

<details>
<summary><strong>Если на странице release нет musl-варианта</strong></summary>

Либо собирать самому с `CGO_ENABLED=0` (для Go — даёт статический бинарь без glibc-зависимости), либо использовать пакет из apk/opkg feeds (если он там есть). Свой билд предпочтительнее для свежих фич, feeds — для стабильных версий.

</details>

### 5.3 VPN-стек: podkop

[Podkop](https://github.com/itdoginfo/podkop) — community-реализация **VLESS VPN** для OpenWrt: domain-routed стек на базе sing-box с динамической подгрузкой списков доменов от сообщества + возможность дополнять собственными.

**Какую проблему решает.** Стандартный wireguard/openvpn — это full-tunnel: «весь трафик в одну точку». Для домашнего случая обычно нужно другое — заворачивать через VPN **только конкретные домены** (AI-сервисы, конкретные SaaS, заблокированные ресурсы), оставив всё остальное идти напрямую. Это domain-based routing, и podkop его реализует «из коробки».

**Быстрая установка.**

```sh
sh <(wget -O - https://raw.githubusercontent.com/itdoginfo/podkop/refs/heads/main/install.sh)
```

После установки появляется LuCI-страница и UCI-конфиг `/etc/config/podkop`. Минимум, что нужно ввести руками — VLESS-URL твоего сервера.

![LuCI-интерфейс podkop — основная страница настроек](../../assets/openwrt-router-bringup/podkop-luci-dashboard.png "Главная страница podkop в LuCI с настройками connection_type, VLESS-URL, community lists и user_domains_text")

**Под капотом** — sing-box (VLESS XTLS, domain matching, fake-IP) + dnsmasq (DNS-кеш) + nftables (перехват трафика), собранные в готовый UCI-интерфейс.

**Режим работы: proxy vs tunnel.** Два режима через `connection_type`:

- **`proxy`** — *domain-routed*. Через туннель идут только домены/IP из списков, всё остальное — напрямую. Это режим домашнего «избирательного VPN», который я и использую.
- **`tunnel`** — *full-tunnel*. Весь трафик через туннель, кроме явных исключений (LAN, NTP). Аналог обычного VPN, но через podkop-стек.

Для домашнего use-case почти всегда `proxy`: netflix/torrent не тормозит, банкинг/госуслуги не ломаются о VPN-исходящий IP, провайдер видит только VPN-handshake.

**Динамические списки: community + свой.** Podkop подгружает два уровня. Сами списки живут в отдельном репозитории — [itdoginfo/allow-domains](https://github.com/itdoginfo/allow-domains), пересобираются CI **ежедневно** (релизы тегаются по дате, например `2026-03-09_09-08`).

**Региональный список (один из двух взаимоисключающих):**

- **`russia_inside`** — для тех, кто **внутри России**. Содержит зарубежные ресурсы, которые либо заблокированы в РФ, либо сами блокируют российские подсети: Anime, Block, GeoBlock, News, Porn, HDRezka, Meta, TikTok, Twitter, YouTube, Discord. Базовый выбор для большинства пользователей в стране.
- **`russia_outside`** — для тех, кто **за пределами России**, но кому нужен доступ к российским сервисам (банкинг, госуслуги, локальные SaaS), которые блокируют зарубежные IP. Полная противоположность `russia_inside`.

Для Украины аналогично — `ukraine_inside`.

**Сервисные/тематические списки** — добавляются поверх регионального для тонкой настройки. В моей конфигурации (типовой набор для активного AI-юзера и SaaS-разработчика):

```
list community_lists 'meta'         # facebook/instagram/whatsapp домены и CDN
list community_lists 'telegram'     # tg-серверы и медиа-CDN (стабильность при шейпинге)
list community_lists 'cloudflare'   # подсети CF — много западных SaaS прячется за ними
list community_lists 'google_ai'    # Gemini/AI Studio — отдельный список, потому что Google
                                    # часто помечает зарубежные серверы как RU
list community_lists 'google_play'  # синхронизация и обновления Play Store
list community_lists 'cloudfront'   # AWS CloudFront — крупная часть западных сайтов
```

Доступны также: `discord`, `twitter`, `tiktok`, `youtube`, `hdrezka`, `anime`, `news`, `porn`, `hodca` (Hetzner/OVH/DO/Cloudflare/AWS — облачные подсети для anti-georestriction в одной пачке).

Логика выбора простая: бери то, чем реально пользуешься. Каждый список — это +N тысяч доменов в nftset и +RAM/CPU на матчинг. Если YouTube смотришь нативно, hdrezka не используешь, twitter не открываешь — соответствующие списки не подключай, экономь ресурсы роутера.

**Свой список** — мержится поверх community, не ломая авто-обновление:

```
option user_domain_list_type 'text'
option user_domains_text 'claude.ai
chatgpt.com
linkedin.com
your-personal-domain.example'
```

Свой список — для AI-сервисов, профессиональных ресурсов и личных доменов, которые не попадают в community-категории.

Обновление списков прописывается в crontab **автоматически самим podkop** при первой настройке — не нужно править руками. Расписание задаётся через UCI-опцию `update_interval`, она маппится на cron-выражение:

| `update_interval` | Cron-расписание |
|---|---|
| `1h` | `13 * * * *` (каждый час в :13) |
| `3h` | `13 */3 * * *` (каждые 3 часа в :13) |
| `12h` | `13 */12 * * *` (каждые 12 часов в :13) |
| `1d` *(default)* | `13 9 * * *` (ежедневно в 09:13) |
| `3d` | `13 9 */3 * *` (раз в 3 дня в 09:13) |

У меня `1d` — для домашней сети ежедневного обновления более чем достаточно. Если ставишь чаще — следи за нагрузкой на CPU при `podkop list_update` (на MT7981 один цикл занимает 10-20 сек CPU).

**Cache на USB, не tmpfs.** По умолчанию sing-box хранит cache в `/tmp/sing-box/cache.db` — после каждого reboot он пустой, и sing-box при старте качает 6+ rule-sets с github. Лечится переносом на USB:

```
podkop.settings.cache_path='/mnt/usb/etc/sing-box/cache.db'
```

После reboot rule-sets читаются с диска моментально.

**Flow offloading должен быть выключен** — sing-box использует nftables, а flow offloading уводит пакеты мимо netfilter pipeline. Подробный разбор механики — в фазе 1 (4.5).

**`disable_quic='1'` — обязательная настройка.** Без этой опции domain-routing работает «только наполовину»: многие современные сервисы (YouTube, Google Search, всё что через HTTP/3) предпочитают QUIC поверх UDP вместо HTTPS поверх TCP. QUIC на UDP в NFQUEUE обрабатывается иначе и сессии уходят «мимо» правил, из-за чего домены из списков частично не попадают в туннель. `option disable_quic '1'` в podkop ставит nftables-правило на блокировку исходящего UDP/443, клиенты автоматически падают обратно на TCP/443, который ловится правилами однозначно. У меня в проде включено по этой причине.

<details>
<summary><strong>Admin API (Clash) — bind на 127.0.0.1</strong></summary>

Sing-box умеет Clash API на порту 9090 (управление маршрутами, мониторинг трафика). По умолчанию podkop выставляет его на LAN-IP без авторизации — любой клиент в LAN может управлять прокси.

Правильно — bind на localhost:

```
podkop.settings.service_listen_address='127.0.0.1'
```

Это undocumented опция, но работает. Доступ через ssh-tunnel:

```sh
ssh home-router -L 9090:127.0.0.1:9090
```

Этот же паттерн применяю ко всем admin-API на роутере.

</details>

### 5.4 DPI-обход: b4

[b4](https://github.com/DanielLavrushin/b4) (*Bye Bye Big Bro*) — пакетный процессор на NFQUEUE: ловит TLS ClientHello, модифицирует его так, чтобы DPI не смог склеить fingerprint, и пропускает дальше. Туннелирования нет, трафик остаётся на том же канале — только handshake становится «невидимым» для DPI.

**Какую проблему решает.** Podkop заворачивает целевые домены через VPN, но трафик, идущий **мимо** туннеля, проходит через провайдера — а провайдер делает DPI и может портить TLS-handshake даже на легитимные ресурсы. Симптом: «`https://example.com` грузится с шестого раза, или сессия отваливается через 5-10 секунд». b4 чинит handshake для трафика «вне VPN», чтобы повседневное browsing было стабильным.

**Быстрая установка.**

```sh
wget -qO- https://raw.githubusercontent.com/DanielLavrushin/b4/main/install.sh | sh
```

Скрипт сам определяет платформу, качает musl-бинарь, ставит зависимости (`kmod-nft-queue`, `kmod-nf-conntrack-netlink`, `iptables-mod-nfqueue`, `jq`, `wget-ssl`), кладёт init.d, разворачивает web UI и **интерактивно спрашивает** про HTTPS-сертификат и логин/пароль — соглашайся, это базовая защита из коробки.

**Альтернатива — [zapret2](https://github.com/bol-van/zapret2) от bol-van.** Делает то же самое другими стратегиями. Что сработает — зависит от DPI-оборудования провайдера: у меня zapret2 не справился с тем, с чем справился b4; у тебя может быть наоборот — тестируй оба.

**Связка с podkop.** Не конфликтуют — podkop заворачивает целевые домены в VLESS, b4 чинит handshake для остального. Оба NFQUEUE-based, поэтому `flow_offloading='0'` обязателен (см. 4.5).

**Безопасность web UI на :7000 — два рубежа поверх install-скрипта:**

1. **HTTPS + basic-auth** — что предлагает install-скрипт. Для домашней сети обычно достаточно.
2. **+ ограничение на firewall-уровне** — firewall разрешает input на :7000 только из LAN-зоны (зоны Guest/IoT уже имеют `input REJECT` — то есть из коробки они не достучатся, даже зная credentials).

В b4 v1.62.x **bind на конкретный IP через конфиг не поддерживается** (поля `web.listen` / `web_server.listen` нет — UI всегда слушает `:::7000` на всех интерфейсах). Единственный реальный рубеж — firewall zones + basic-auth. Этого для домашней LAN с доверенными клиентами достаточно.

<details>
<summary><strong>Активация TLS для web UI на v1.62.x (через PEM-сертификат)</strong></summary>

В v1.62.x **нет** отдельного флага «включить HTTPS». TLS активируется автоматически, если поля `system.web_server.tls_cert` и `system.web_server.tls_key` указывают на валидные **PEM**-файлы. После старта HTTP-запросы получают 400, только HTTPS — нормальный ответ.

⚠️ **LuCI-сертификат `/etc/uhttpd.crt` не подойдёт** — он в **DER**-формате (px5g с флагом `-der`), b4 пишет в лог `Invalid TLS certificate/key pair: tls: failed to find any PEM data in certificate input — falling back to HTTP`.

Сгенерить отдельную PEM-пару через тот же `px5g`, но без `-der`:

```sh
px5g selfsigned \
    -days 397 -newkey ec -pkeyopt ec_paramgen_curve:P-256 \
    -keyout /etc/b4-tls.key -out /etc/b4-tls.crt \
    -subj /C=ZZ/ST=Home/L=Local/O=OpenWrt/CN=b4-router \
    -addext "extendedKeyUsage=serverAuth" \
    -addext "subjectAltName=IP:<router-LAN-IP>,DNS:b4-router"
chmod 600 /etc/b4-tls.key
```

В `b4.json`:

```jsonc
"system": {
    "web_server": {
        "tls_cert": "/etc/b4-tls.crt",
        "tls_key":  "/etc/b4-tls.key"
        // password/username из install.sh — не трогать
    }
}
```

Браузер всё равно покажет warning о self-signed cert (это норма для домашнего стека), один раз принимаешь exception. Если хочется без warning — подкладывать LE-серт через DNS-01 challenge, но для LAN-UI это overkill.

</details>

**Крутая фишка: watchdog + discovery.** b4 умеет периодически пинговать целевые домены и проверять handshake. Если домен перестал работать — автоматически запускается [discovery](https://daniellavrushin.github.io/b4/ru/docs/discovery): перебор десятков стратегий обхода в реальном времени, поиск рабочей, автоматическое её применение. Из [документации watchdog](https://daniellavrushin.github.io/b4/ru/docs/watchdog):

> «Мониторинг периодически проверяет доступность указанных доменов и при обнаружении блокировки автоматически запускает дискавери для поиска рабочей конфигурации.»

У меня так и настроено — список «обязательных к работе» доменов, который b4 мониторит. Когда провайдер меняет фильтры, b4 сам подбирает новую стратегию, без моего вмешательства.

![Web UI b4 — главная страница со статусом и watchdog-списком](../../assets/openwrt-router-bringup/b4-web-ui-dashboard.png "Dashboard b4 на :7000 — видны активная стратегия обхода, watchdog-список доменов, статус discovery")

**⚠️ Discovery съедает CPU.** Один полный цикл на MT7981 нагружает CPU в потолок на 30-60 секунд. Если в watchdog-списке десятки доменов и проверки частые, роутер постоянно занят discovery, страдает throughput.

Практическое правило: в watchdog-списке держать **только критичные домены** — обычно 3-7 штук (рабочая почта, основной AI-чат, банкинг). Не весь `user_domains_text`. Для остальных — обнаружение проблемы вручную и точечный discovery через UI.

**⚠️ Memory limit обязателен — иначе watchdog может выесть всю RAM.**
По дефолту b4 не имеет верхней границы по памяти. Когда watchdog перебирает стратегии в discovery + ведёт статистику по домен-листу, потребление памяти растёт. На роутере с 512 MB RAM один «убежавший» b4 может вытолкнуть в OOM dnsmasq, sing-box или dropbear — и роутер уходит в нерабочее состояние без явного error log'а.

В `b4.json` добавляй явный `memory_limit` (в блоке `system`, как и всё остальное):

```jsonc
"system": {
    "memory_limit": "150MiB",
    "web_server": {
        // ... credentials, tls_cert/tls_key
    }
}
```

У меня стоит `"150MiB"` — этого достаточно для нормальной работы b4 с watchdog'ом на ~5-7 доменов. При превышении лимита b4 сам себя ограничивает (либо терминирует и перезапускается через procd respawn) вместо того чтобы съесть всё. Без этой опции на роутере с тонким RAM-балансом — мина замедленного действия.

<details>
<summary><strong>Оптимизации v1.62.x: --threads, geo paths</strong></summary>

**`--threads N` — сократить worker pool под shared CPU.** Дефолт b4 — `--threads 4`, под 4-ядерный ARM64 он считает все ядра своими. На роутере, где параллельно работают sing-box / dnsmasq / podkop, лишние воркеры дают только context-switch overhead. Через `/etc/init.d/b4` поправить запуск:

```diff
-procd_set_param command $PROG --config $CONFIG
+procd_set_param command $PROG --config $CONFIG --threads 2
```

После рестарта `--threads 2` запишется и в config как `queue.threads: 2` (b4 персистит CLI-флаги в `b4.json`).

**Geo paths (`system.geo` в b4.json).** Блок с путями на `geoip.dat` / `geosite.dat` нужен **только если** в каком-то set'е используется `targets.geosite_categories` или `targets.geoip_categories`. Если все правила построены на `sni_domains` — geo не нужен.

Откуда берётся блок:

- **install.sh** — может прописать дефолтные пути при первой установке.
- **Web UI** — добавляешь после установки через Settings, b4 сам ходит и качает `.dat` файлы по `_url`.

Файлы по 30-60 MB каждый — кладёшь на USB (`/mnt/usb/etc/b4/`) или не подключаешь geo-фичи вовсе.

</details>

Документация: [quickstart](https://daniellavrushin.github.io/b4/docs/quickstart), [основной сайт](https://daniellavrushin.github.io/b4/), [обсуждение на ntc.party](https://ntc.party/t/b4-%E2%80%94-dpi-bypass-%D0%B4%D0%BB%D1%8F-linux-%D1%83%D1%81%D1%82%D1%80%D0%BE%D0%B9%D1%81%D1%82%D0%B2-%D1%81-%D0%B2%D0%B5%D0%B1-%D0%B8%D0%BD%D1%82%D0%B5%D1%80%D1%84%D0%B5%D0%B9%D1%81%D0%BE%D0%BC-%D0%B8-%D0%B0%D0%B2%D1%82%D0%BE%D0%BF%D0%BE%D0%B4%D0%B1%D0%BE%D1%80%D0%BE%D0%BC-%D0%BD%D0%B0%D1%81%D1%82%D1%80%D0%BE%D0%B5%D0%BA/21443).

<details>
<summary><strong>Параноидальный вариант: bind на 127.0.0.1 + ssh-tunnel</strong></summary>

Тот же паттерн, что описан в 5.3 для Clash API на :9090: бинд на `127.0.0.1`, доступ через `ssh -L`. Для b4 — `listen: "127.0.0.1:7000"` в `b4.json` и `ssh home-router -L 7000:127.0.0.1:7000` с ноута.

Я этого не использую, потому что UI нужно открывать с разных устройств и поднимать tunnel каждый раз ломает UX. Но если в LAN потенциально могут быть недоверенные клиенты (общественная квартира, шеринг с соседом) — это правильный путь.

</details>

<details>
<summary><strong>Если b4 и podkop конфликтуют (troubleshooting)</strong></summary>

Оба ставят свои nftables-правила в input chain'ы. Иногда они мешают друг другу — особенно если оба пытаются захватить TLS handshake. Симптом — конкретный домен либо не обходится (трафик уже ушёл в podkop), либо висит (оба запросили пакет в очередь).

Лечится явным разделением: в b4 в конфиге перечисляешь домены, которые он трогает, и убеждаешься, что они **не пересекаются** со списком podkop. Podkop заворачивает целиком в туннель → ему DPI-обход не нужен; b4 чинит то, что идёт напрямую.

</details>

### 5.5 DNS: где живёт резолвер

В моей конфигурации DNS — это **гибридная схема dnsmasq + sing-box**: dnsmasq слушает клиентов и кеширует, форвардит запросы на `127.0.0.42` (sing-box DNS), sing-box ходит на upstream через DoH и ведёт fake-IP таблицу для маршрутизации.

**Зачем именно так.** Когда роутер делает domain-based routing, DNS становится частью routing-стека: sing-box смотрит на A-запись резолвера и принимает решение, что делать с трафиком (отправить через VPN или напрямую). Один резолвер не годится — нужен умный матчер (sing-box) + надёжный кеш (dnsmasq). Каждый делает то, в чём он силён.

**Минимальная настройка.** Если podkop уже стоит — **ничего делать не надо**, гибрид настраивается им из коробки. Проверка работает ли всё:

```sh
nslookup claude.ai
# IP в 198.18.0.0/15 = sing-box взял домен на себя (fake-IP)
# реальный IP = идёт напрямую через dnsmasq
```

**Bootstrap-DNS для DoH.** Чтобы sing-box разрешил имя DoH-апстрима (`dns.google`), ему нужен **plain UDP**-резолвер на старте — `8.8.8.8`, `1.1.1.1` или ISP. Сделать bootstrap тоже через DoH = chicken-and-egg, sing-box застрянет в startup. Plain UDP используется один раз, дальше всё идёт через DoH.

**DNS rebind protection — не выключать.**
OpenWrt по умолчанию имеет `option rebind_protection '1'` в `/etc/config/dhcp` — dnsmasq дропает ответы, в которых публичные домены резолвятся в RFC1918 IP (классическая DNS-rebinding атака на LAN). Это включено из коробки, и **выключать его не надо**, даже если кажется, что «что-то не резолвится».

Если у тебя есть локальный домен (например, `home.lan` или wildcard для домашних сервисов) — добавь его в whitelist через `option rebind_domain`, не отключай защиту целиком:

```
config dnsmasq
    option rebind_protection '1'
    list rebind_domain '/home.lan/'
    list rebind_domain '/my.local.dev/'
```

<details>
<summary><strong>Когда имеет смысл https-dns-proxy (если sing-box у тебя нет)</strong></summary>

Если ты **не используешь podkop / sing-box** для domain-routing'а, а просто хочешь надёжный DoH-резолвер на роутере, имеет смысл поставить отдельный демон — например `https-dns-proxy` (есть в feeds OpenWrt) или `cloudflared`. Это даёт зашифрованный DNS, изолированный от любого VPN-стека.

Со sing-box эта схема избыточна — sing-box и так умеет DoH, и его DNS уже работает (плюс ведёт fake-IP). Ставить ещё `https-dns-proxy` параллельно — лишнее звено и точка отказа.

Правило простое: **есть sing-box (с podkop)** → используй его DNS. **Нет sing-box** → ставь `https-dns-proxy` отдельно.

</details>

<details>
<summary><strong>Опционально: апгрейд sing-box до 1.13.x вместо дефолтных feeds</strong></summary>

**По умолчанию** podkop ставит sing-box из apk feeds — это **1.12.x** (на момент 2026-05: `1.12.17-r1`, 38.6 MB). **Это рекомендованный путь** — пакет проверенный, обновляется через `apk upgrade`, не требует ручного обслуживания.

Если хочется свежих фич 1.13-ветки — её можно подкатить вручную бинарём с github releases. Релевантные изменения 1.13 vs 1.12 для VLESS+podkop:

- **Производительность fake-IP** — заметнее всего на domain-routing'е, центральной фиче.
- **Улучшен local DNS server** — релевантно гибриду dnsmasq+sing-box (см. 5.5).
- **kTLS support** — потенциальный ускоритель TLS handshake на ядрах 6.x.
- **Свежие rule-set форматы (SRS)** — community-списки на github публикуются с учётом возможностей последней версии.
- **Багфиксы в DoH-bootstrap** — в 1.12 был edge-case с retry на bootstrap-резолве при недоступности upstream, чинят в 1.13.

**Цена апгрейда:**

- **Размер**: musl-бинарь 1.13.12 — **61.4 MB** против **38.6 MB** у 1.12.17 (+59%). На роутерах с тонким flash (44 MB overlay на Cudy WR3000P) это критично — занимает почти всё свободное место.
- **Совместимость**: на musl-системах **обязательно** качать `sing-box-X.Y.Z-linux-arm64-musl.tar.gz`, а не дефолтный `-linux-arm64.tar.gz` (тот собран под glibc, см. 5.2). Проверка: `sing-box version` должна содержать `with_musl` в `Tags:`.
- **Обслуживание**: ручное обновление, бинарь не контролируется apk → не обновляется автоматически при `apk upgrade`.

**Когда оправдан апгрейд:**

- У роутера много свободного flash (≥80 MB overlay) — размер некритичен.
- Бинарь живёт на USB через `/etc/profile.d/10-usb-bin.sh` (см. 5.2) — flash не страдает.
- Действительно нужны фичи 1.13 (например, kTLS для CPU-нагруженных сценариев).

В остальных случаях — оставайся на **apk-версии 1.12.x**. Стабильно работает, легко обновляется, не съедает flash. Я лично сижу на feeds-версии.

</details>

### 5.6 Логи: tmpfs и осознанная ротация

Logrotate, настроенный под особенность OpenWrt — `/var` это symlink на `/tmp`, поэтому `/var/log/*.log` физически живут на **tmpfs (RAM)**.

**Какую проблему решает.** Логи на tmpfs не давят на overlay (никакой «лог-бомбы», убивающей flash) и исчезают при reboot (естественная ротация). Но они **давят на tmpfs**, и при безудержном росте могут забить RAM-диск и уронить всё, что пишет в `/tmp`. Если `b4` или `sing-box` логирует на debug-level, легко выесть 100 MB tmpfs за пару часов.

**Быстрая установка.**

```sh
apk add logrotate
```

Конфиг в `/etc/logrotate.d/openwrt`:

```
/var/log/*.log
{
    size 1M
    rotate 3
    missingok
    notifempty
    copytruncate
    compress
    delaycompress
}
```

Cron — ежечасно (`/etc/crontabs/root`):

```
0 * * * * /usr/sbin/logrotate /etc/logrotate.conf
```

Пороги жёстче, чем на десктопе — потому что под tmpfs (256 MB), а не overlay. Cron чаще ежечасно — overhead на CPU; реже — недостаточно реактивно при росте логов.

<details>
<summary><strong>Объяснения по опциям logrotate (если интересно почему именно так)</strong></summary>

- **size 1M, rotate 3** — мелкие пороги под tmpfs, не overlay. На десктопе ты бы поставил 100M; на роутере с 256 MB tmpfs нужно жёстче.
- **copytruncate** — сервисы держат fd на текущий лог. Без `copytruncate` logrotate переименует файл, а процесс продолжит писать в старый дескриптор, и новый лог-файл останется пустым.
- **delaycompress** — первая ротированная копия (`.log.1`) остаётся несжатой. У некоторых процессов есть race condition при reopening fd, и доступ к свежему `.log.1` без gzip-обёртки помогает дебажить инциденты.
- **compress** — для `.log.2+` сжатие имеет смысл (gzip overhead окупается).

</details>

### 5.7 Reliability: двухуровневый watchdog (опционально)

Связка `watchcat` (мониторинг ping'а в режиме `run_script`) и собственного recovery-скрипта `/root/bin/watchdog-recovery.sh`, который реализует мягкую эскалацию вместо немедленного reboot.

**Какую проблему решает.** Стандартный `watchcat` имеет один режим эскалации — `reboot` при пропавшем ping. Для домашнего роутера это слишком жёстко: один пропущенный пакет до 1.1.1.1 не должен ребутить устройство и роняющий wifi всем в квартире. Двухуровневая схема даёт промежуточные шаги (ifdown/ifup, network restart) перед тем как доходить до reboot.

**Быстрая установка.**

```sh
apk add watchcat
service watchcat enable
```

UCI в `/etc/config/watchcat`:

```
watchcat.wan_monitor.mode='run_script'
watchcat.wan_monitor.script='/root/bin/watchdog-recovery.sh wan'
watchcat.wan_monitor.pingperiod='60'
watchcat.wan_monitor.forcedelay='180'
watchcat.wan_monitor.pinghosts='1.1.1.1 8.8.8.8'
```

После — `service watchcat restart`.

Схема:

1. **Watchcat** мониторит ping (1.1.1.1, 8.8.8.8) каждые 60 сек, делает delay 180 сек на fail, вызывает custom recovery-script.
2. **Recovery-script** делает свою эскалацию: cooldown 15 мин, 3 retry ping, `ifdown/ifup wan`, `service network restart`, и только если ничего не помогло — `reboot`.

Сам скрипт — отдельный shell с `set -eu`, lockdir, cooldown-файлом, и поддержкой `WDR_TEST_MODE=1` для dry-run проверки эскалации без реального reboot. Краткое описание и ключевые решения — в подразделе 6.1, примеры реализации — в репозитории `openwrt-safe-helper-scripts` (см. раздел 9 «Связанные заметки»).

<details>
<summary><strong>Почему TEST_MODE критически важен</strong></summary>

Без TEST_MODE дебажить recovery-script страшно — один баг, и ты получаешь reboot-loop с роутером в прихожей, до которого нужно идти ножками.

С TEST_MODE можно прогнать всю цепочку эскалации на boot, увидеть весь output в логе, и убедиться, что `reboot` позовётся только когда реально надо. Тест проходит, потом ставится в продакшен без TEST_MODE — и можно спокойно спать.

</details>

## 6. Диагностика и сопровождение

Когда стек поставлен, важно знать что он работает, и быстро понимать *где* именно сломалось, когда не работает. Эта глава — про operations: периодические скрипты, cron-задачи и однострочники для диагностики.

**Пакеты в этой главе:**

```sh
apk add bind-dig curl
```

(`nftables`, `ss`, `logread`, `ubus`, `df` — встроенные в систему, отдельной установки не требуют)

### 6.1 Скрипты в /root/bin/

Три shell-скрипта повседневной эксплуатации: мониторинг диска, recovery от сетевых сбоев, автообновление b4 binary.

**Какую проблему решают.** Без них ты обнаруживаешь проблемы post-mortem — когда tmpfs уже забит и роутер падает, когда WAN отвалился ночью и ты узнаёшь утром, когда новый release b4 ломает совместимость и стек уходит в reboot-loop без бэкапа. Скрипты ниже это закрывают.

**Быстрая установка.**

```sh
mkdir -p /root/bin
# положить .sh-файлы туда — примеры реализации в репозитории
# openwrt-safe-helper-scripts (ссылка в разделе 9 «Связанные заметки»)
chmod +x /root/bin/*.sh
```

**Остальные детали:**

**`monitor-tmp.sh`** — каждые 15 минут проверяет свободное место на `/tmp` (tmpfs), `/overlay` (ubifs) и `/mnt/usb`, и при пересечении порогов пишет `CRIT`/`WARN`/`INFO` в syslog. Особо проверяет факт монтирования `/mnt/usb` — если флешку вытащили, отдельный `CRIT` (потому что b4/sing-box cache становятся недоступны без предупреждения). Все пороги настраиваются env-переменными.

**`watchdog-recovery.sh`** — уже рассмотрен в подразделе «Reliability: двухуровневый watchdog» (раздел 5). Двухуровневая эскалация с cooldown, lockdir и `WDR_TEST_MODE=1` для dry-run.

**`b4-binary-update.sh`** — проверяет github releases b4, скачивает musl-бинарь под платформу, верифицирует sha256, делает atomic swap старого бинаря на новый, перезапускает сервис через clean procd-cycle, прогоняет smoke-test (pidof + HTTP-probe + nft-tables), и **при провале откатывает на предыдущий бэкап** с восстановлением живого сервиса. По умолчанию работает в `--check` режиме (только сравнить версии) — apply вручную, чтобы не получить auto-update пятницей вечером.

<details>
<summary><strong>Тонкие моменты в b4-binary-update.sh, которые стоит знать</strong></summary>

- **Multi-version retention бэкапов** — `/mnt/usb/etc/b4/bin/b4-<ver>-<ts>.bak`. Хранится последние N бэкапов **+ как минимум один бэкап каждой уникальной версии**. То есть много update на одну версию не вытесняет историю.
- **Кэширование архивов** — скачанный tar.gz и его sha256 сохраняются в `/mnt/usb/etc/b4/cache/`. Если новый билд оказался плохой, и старый бэкап повреждён — можно восстановиться из upstream-архива оффлайн.
- **Direct-run probe при failure** — если smoke-test не прошёл, скрипт запускает b4 напрямую (`/mnt/usb/usr/bin/b4 --config ...`) на 8 секунд и пишет stderr в `probe-*.log`. Часто причина — config-incompatibility между версиями, и direct-run сразу пишет ошибку конфига.
- **Aggressive procd reset** — обычный `service b4 restart` не сбрасывает respawn-budget после превышения порога. Скрипт делает `ubus call service delete` + явный `killall` + `rm pidfile` перед стартом, чтобы запустить b4 «с чистого листа».

</details>

### 6.2 Cron-задачи

Сводный crontab со всеми периодическими задачами сопровождения.

**Зачем нужно.** Cron-расписание разбросано по тексту разных разделов — здесь оно собрано в одном месте, чтобы было видно «когда что запускается» без перекапывания статьи.

**Минимальная настройка.** Содержимое `/etc/crontabs/root`:

```cron
*/15 * * * * OVL_INFO_KB=5120 /root/bin/monitor-tmp.sh
23 4 * * * /root/bin/b4-binary-update.sh --check
0 * * * * /usr/sbin/logrotate /etc/logrotate.conf
13 9 * * * /usr/bin/podkop list_update
```

После правок crontab — `service cron restart` обязательно.

**Остальные детали:**

- `monitor-tmp` — каждые 15 минут, с overridden `OVL_INFO_KB=5120` чтобы info-порог для `/overlay` был 5 MB (вместо дефолтных 8 MB — мне ок более тесные пороги, я не сильно расширяю overlay)
- `b4-binary-update.sh --check` — раз в сутки в 04:23, **только проверка**, не apply. Если есть новая версия — пишется в syslog, обновление ручное командой `--apply`. Так я не получаю auto-update пятницей вечером
- `logrotate` — ежечасно (см. раздел 5, «Логи: tmpfs и осознанная ротация»)
- `podkop list_update` — раз в сутки в 09:13. Это **default подкоп'а** при `update_interval='1d'` (см. 5.3); подкоп сам прописывает строку в crontab при первой настройке

### 6.3 Диагностические команды

Набор однострочников для быстрой проверки, что стек живой.

**Какую проблему решают.** Когда что-то «не работает», цена ошибки — время на разбор. Эти команды сужают поиск: DNS, маршрутизация, DPI-обход, туннель или провайдер.

**Минимальная настройка** (никакой установки сверх `bind-dig` + `curl`):

```sh
# DNS работает?
nslookup claude.ai
dig claude.ai @127.0.0.1

# что в nftables (видны таблицы podkop и b4)?
nft list tables

# что слушает на роутере?
ss -tlnp

# домен идёт через туннель или напрямую?
nslookup claude.ai
# если IP в 198.18.0.0/15 — пошёл через sing-box (fake-IP)
# если реальный IP — напрямую

# что в логах сервисов?
logread -e podkop | tail -50
logread -e b4 | tail -50
logread -e dnsmasq | tail -20

# статус сервисов
ubus call service list '{"name":"podkop"}'
ubus call service list '{"name":"b4"}'
ubus call service list '{"name":"sing-box"}'
```

<details>
<summary><strong>Типичные сценарии: где смотреть когда «не работает»</strong></summary>

**«Конкретный домен не работает».**
Сначала проверь резолв (`dig`), потом fake-IP (через `nslookup` — IP в `198.18.0.0/15` = sing-box взял на себя). Если резолв пустой → DNS проблема (`logread -e dnsmasq`). Если fake-IP отсутствует → домен не в списках podkop, проверь конфиг. Если есть, но не открывается → проблема в туннеле (`logread -e podkop`, статус Clash API).

**«Сайт грузится частично / сессия отваливается через 5-10 секунд».**
Это обычно DPI-проблема для трафика мимо туннеля. Запусти discovery в b4 web UI для этого домена, посмотри найдётся ли рабочая стратегия.

**«Wifi есть, интернета нет».**
`ping -I wan 1.1.1.1` с роутера. Если не идёт — провайдерская сторона (или WAN-настройки). Если идёт, но клиенты не работают → проверь DNS (`logread -e dnsmasq`) и forwarding между зонами (`fw4 print` для проверки активных правил).

**«Tmpfs забит, странные ENOSPC».**
`df -h /tmp /overlay /mnt/usb` + `logread | grep monitor-tmp`. Свежий output из cron-мониторинга подскажет, какой mount-point под давлением и когда это началось.

</details>

### 6.4 Backup конфигурации

**Какую проблему решает.** SNAPSHOT не сохраняет пакеты между sysupgrade — после апдейта роутер возвращается к «чистый OpenWrt + дефолтные конфиги». Без актуального backup'а восстановление превращается в час кликания в LuCI и копипасты UCI. С backup'ом — несколько минут.

**Простой путь — штатный `sysupgrade -b`.** Это та же команда, которой LuCI собирает архив (System → Backup/Flash Firmware → Generate Archive). Берёт всё, что перечислено в `/etc/sysupgrade.conf` (`/etc/config/*`, `/etc/dropbear/authorized_keys`, ещё несколько системных файлов). Расширь под свой стек — допишешь то, что нужно сохранить:

```sh
cat >> /etc/sysupgrade.conf <<'EOF'
/root/bin/
/etc/b4/
/etc/b4-tls.crt
/etc/b4-tls.key
EOF
```

Что и зачем:
- `/root/bin/` — кастомные helper-скрипты (`monitor-tmp.sh`, `watchdog-recovery.sh`, `b4-binary-update.sh` — см. 6.1)
- `/etc/b4/` — рабочий конфиг b4 (`b4.json` и сопутствующее)
- `/etc/b4-tls.crt` + `/etc/b4-tls.key` — PEM-пара для HTTPS web UI b4 (см. 5.4, спойлер про TLS)

Дальше — еженедельный cron:

```cron
30 3 * * 0 sysupgrade -b /mnt/usb/backups/openwrt-$(date +\%Y\%m\%d).tar.gz
```

Ротация (держать последние N) — отдельной cron-командой через `find -mtime` или `ls -1t … | tail -n +N | xargs rm`.

**Через LuCI вручную.** System → Backup/Flash Firmware → Generate Archive. Используй для разовых snapshots перед рискованными изменениями.

**Backup на USB, не во flash:** USB можно вытащить и прочитать с ноута, если роутер не грузится. При rollback после кирпича архив доступен на любой машине, не требует интернета на старте. Опционально — `git push` в приватное репо после успешного backup'а для off-site копии.

**⚠️ Backup содержит секреты.** В архив попадают wifi-пароли в plaintext (`/etc/config/wireless`), VLESS-URL (`/etc/config/podkop`), basic-auth для b4 web UI (`/etc/b4/b4.json`), TLS-ключ b4 (`/etc/b4-tls.key`), SSH host keys (`/etc/dropbear/dropbear_*_host_key`). Если USB-флешку вытащили или нашли — все эти credentials уехали с ней.

Варианты защиты:
- USB шифровать (ext4 поверх LUKS или f2fs + dm-crypt).
- Хранить backup на роутере (не на USB) — но тогда теряется главное преимущество: возможность достать backup с ноута если роутер не грузится.
- Бэкапить отфильтрованный набор (без `/etc/config/wireless`, `/etc/config/podkop`, `/etc/b4-tls.key`, host keys) и держать секреты отдельно в password manager. Минус — bootstrap после кирпича потребует ручного дозалития секретов.

Токены и chat_id для алертов (см. 6.5) **не клади в `/root/bin/` скрипты** — храни в `/root/.notify.env` и подключай через `. /root/.notify.env`. Этот файл осознанно НЕ добавляй в `/etc/sysupgrade.conf`.

**Bootstrap после sysupgrade — стратегия, а не готовый скрипт.** Идея простая: `sysupgrade` с `Keep settings: OFF` → восстановить конфиги из последнего архива (`tar -xzf` на `/`) → переустановить пакеты той же командой из шапок 4 и 5 (`apk add` + install-скрипты podkop/b4) → `service network restart && wifi reload && service firewall restart` → старт сервисов. На SNAPSHOT весь цикл — 5–10 минут вручную. Готовый скрипт у меня в работе, но **e2e на живом sysupgrade ещё не оттестирован** — выложу в [`openwrt-safe-helper-scripts`](раздел 9) после прохождения live-теста.

### 6.5 Alerting: куда уходят CRIT (опционально)

**Какую проблему решает.** `logread | grep CRIT` хорош для пост-мортем разбора, но не для проактивного реагирования. Если tmpfs забит ночью, а узнаёшь утром по факту, что wifi отвалился — алерт сработал, но его никто не увидел.

**У меня сейчас.** Алерты пишутся в syslog (`logger -t` из `monitor-tmp.sh` и других скриптов). Внешней доставки в проде пока нет — проблемы ловлю при чтении `logread` или по симптому. Для домашней сети, где роутер физически рядом, этого хватает. Если хочется проактивности — варианты ниже.

**Минимум — Telegram bot через `curl`.** Бот через @BotFather, `<TOKEN>` и `<CHAT_ID>` — в `/root/.notify.env` (НЕ в `/root/bin/`, чтобы не попало в backup). Подключаешь и зовёшь из `emit()`:

```sh
notify_telegram() {
    curl -s --max-time 10 \
        -d chat_id="$CHAT_ID" \
        -d text="[$(hostname)] $1" \
        "https://api.telegram.org/bot${TOKEN}/sendMessage" >/dev/null
}

emit() {
    level="$1"; mount="$2"; free="$3"; threshold="$4"
    logger -t "$TAG" "${level} ${mount}=${free}KB below ${threshold}KB"
    [ "$level" = "CRIT" ] && notify_telegram "${level} ${mount}=${free}KB below ${threshold}KB"
}
```

**Альтернативы.** Syslog forwarding (`option log_ip` + Loki/Graylog) — если уже есть домашний сервер 24/7; MQTT в Home Assistant (`mosquitto-client` + `mosquitto_pub`) — если HA уже работает и хочется один dashboard; Healthchecks.io / Cronitor — алертит не на CRIT, а на факт «cron не отработал вовсе», полезно как страховка к самим скриптам.

Для one-роутера без домашней инфры Telegram-bot — минимум сложности.

### 6.6 Upgrade path на следующие версии (на будущее)

**Какую проблему решает.** На SNAPSHOT каждый `sysupgrade` работает по принципу «прошил → bootstrap.sh → готово». Но между мажорными релизами или между SNAPSHOT и stable могут ломаться формат UCI, пути к бинарям, схема feeds. Без плана это превращается в часы дебага.

**Минимальная процедура** перед мажорным апгрейдом:

1. Свежий backup конфигов (см. 6.4), проверить что архив на USB читается.
2. Прочитать [release notes](https://openwrt.org/releases/start) на предмет breaking changes.
3. Скачать sysupgrade-образ под точную модель и ревизию (см. ToH).
4. Прошить с `Keep settings: ON` в первый раз — посмотреть что осталось живым.
5. Если что-то поломалось — прошить ещё раз с `Keep settings: OFF` и применить bootstrap.sh для пересоздания стека с нуля.

**Текущая ветка — 25.12.** Главное breaking change свежей истории — `opkg → apk` (Alpine Package Keeper), произошло между 24.10 и 25.12. Все команды установки/удаления — `apk add`/`apk del`, репозитории — `/etc/apk/repositories.d/*` вместо `/etc/opkg/*`. UCI-конфиги в большинстве почти совместимы, fw4/DSA — стабилизированы.

Точечные релизы внутри 25.12 (25.12.1/2/3/4) — service releases с security и bugfix, breaking нет, `sysupgrade` проходит штатно.

**Старые миграции** (21.02→22.03 — fw3→fw4; 22.03→23.05 — DSA для MediaTek; 23.05→24.10 — первая волна apk) — релевантны только при апгрейде со старых веток. Если ты уже на 24.10+, эти барьеры пройдены.

**Мой план дальше:** сижу на 25.12.x stable до выхода 26.x. Миграцию делаю двухшаговой: `Keep settings: ON` → проверка стека → при необходимости `Keep settings: OFF` + `bootstrap.sh`. После релиза 26.x обновлю эту статью.

## 7. Production-readiness чеклист

Чтобы не «забыть и удивиться через два месяца», вот короткий чек-лист по группам — пробежался перед тем как роутер уехал в прихожую, и каждый раз перед `sysupgrade`. Стиль и подход взяты из моей же [статьи на Habr про production-readiness checklists](https://habr.com/ru/companies/otus/articles/572822/) — там подробно про то, **почему** такой чек-лист вообще нужен и как держать его под контролем.

Здесь только то, что критично для нашего OpenWrt-кейса (бытовой роутер с domain-routed VPN + DPI-обход). Каждая группа закрывает один класс ошибок.

### Безопасность периметра

- [ ] **SSH:** `PasswordAuth='off'`, `RootPasswordAuth='off'`, нестандартный порт, привязка к `br-lan`
- [ ] **SSH:** публичный ключ в `/etc/dropbear/authorized_keys`, root-password установлен (на всякий случай)
- [ ] **LuCI:** установлены `px5g-mbedtls` + `libustream-mbedtls`, HTTPS поднялся
- [ ] **LuCI:** `listen_http`/`listen_https` привязаны к LAN-IP, не к `0.0.0.0`
- [ ] **Wi-Fi:** дефолтные открытые SSID (`OpenWrt`, `OpenWrt-5G`) удалены
- [ ] **Wi-Fi:** главная сеть на `sae-mixed` (WPA2+WPA3), IoT на `psk2`
- [ ] **System:** `ttylogin='0'` (если есть физический доступ к UART)

### Сеть и сегментация

- [ ] Три отдельных сегмента: LAN, Guest, IoT (свой bridge, свой подсеть)
- [ ] Wi-Fi: SSID для гостей и IoT с `isolate='1'`
- [ ] Firewall-зоны: lan/wan/guest/iot с правильными `input`/`output`/`forward`
- [ ] Forwarding-матрица: LAN→IoT разрешён, IoT→LAN запрещён, Guest изолирован от LAN+IoT
- [ ] `firewall.@defaults[0].drop_invalid='1'`
- [ ] `firewall.@defaults[0].flow_offloading='0'` (если стоит podkop / b4 / zapret)
- [ ] DHCP- и DNS-правила для guest и iot (`Allow-DHCP`, `Allow-DNS`)

### Сервисы (VPN + DPI + DNS)

- [ ] `podkop` установлен, VLESS-URL задан, нужный режим (`proxy`/`tunnel`) выбран
- [ ] `podkop`: региональный список (`russia_inside`/`russia_outside`/`ukraine_inside`) — ровно один
- [ ] `podkop`: свой `user_domains_text` с критичными доменами добавлен
- [ ] `podkop`: `cache_path='/mnt/usb/etc/sing-box/cache.db'` — кеш на USB
- [ ] `b4` установлен install-скриптом, HTTPS+auth для web UI согласован
- [ ] `b4`: web UI привязан к LAN-IP (не `0.0.0.0`), firewall закрывает :7000 от guest/iot
- [ ] `b4`: watchdog-список содержит **только критичные** домены (3-7 штук, не больше)
- [ ] `b4`: `"memory_limit": "150MiB"` в `b4.json` — иначе watchdog может выесть RAM и уронить dnsmasq/sing-box через OOM
- [ ] DNS работает в гибридной схеме (dnsmasq + sing-box), bootstrap-DNS на plain UDP

### Storage и устойчивость

- [ ] USB смонтирован по UUID (не `/dev/sda1`), ext4 или f2fs
- [ ] На USB лежат: бинари из github, sing-box cache, snapshot-бэкапы
- [ ] На USB **не лежат** активные debug-логи и БД с частой записью (write-cycles!)
- [ ] `logrotate` установлен, конфиг под tmpfs (size 1M, rotate 3), cron ежечасно
- [ ] `watchcat` в режиме `run_script` с собственным recovery-скриптом
- [ ] Recovery-script протестирован с `WDR_TEST_MODE=1` перед prod-режимом

### Диагностика и сопровождение

- [ ] `monitor-tmp.sh` в cron каждые 15 минут — порог на `/tmp`, `/overlay`, `/mnt/usb`
- [ ] `b4-binary-update.sh --check` в cron раз в сутки (apply вручную)
- [ ] `podkop list_update` в cron раз в сутки
- [ ] Известны основные diagnostic-команды (`nslookup`, `nft list tables`, `logread -e`)
- [ ] Снимок текущей рабочей конфигурации (`/etc/config/*`) лежит в git или на USB

### Recovery и план Б

- [ ] TFTP-recovery от стока проверен (или хотя бы прочитан гайд) до того, как понадобится
- [ ] Снапшот трансита (`cudy-signed transitional`) и нужный OpenWrt sysupgrade лежат на ноуте
- [ ] Знаешь, как откатиться с OpenWrt на сток (на случай гарантийного обслуживания)
- [ ] Wi-Fi-пароли и SSH-ключи продублированы в password manager — на случай если роутер всё-таки кирпич
- [ ] Backup `/etc/config/*` на USB (см. 6.4), последний архив — не старше недели

### Внешний security audit

- [ ] **Запустить `nmap` снаружи** (с мобильного интернета или VPS) против WAN-IP роутера и убедиться, что **ничего не открыто кроме того, что ты сам пробросил**:

  ```sh
  nmap -sT -p- --min-rate 1000 <wan-ip>     # все TCP-порты
  nmap -sU --top-ports 50 <wan-ip>           # топ-50 UDP
  ```

  Ожидаемый результат: всё `filtered`/`closed`. Если открыто что-то неожиданное (например, :7000 от b4, :9090 от Clash, :22022 от SSH) — что-то не привязано к LAN-интерфейсу или forwarding-rule не сработал. Это **должно** быть проверено после каждого мажорного изменения firewall или установки нового сервиса.

---

Если какой-то пункт пропускается — лучше явно отметить **почему**, чем «забыть и удивиться через два месяца». Это нормальная практика чек-листов: пропуски документируются, а не замалчиваются.

## 8. AI Prompt для подбора своего стека (опционально)

Эта статья описывает мой конкретный сценарий: SNAPSHOT + Cudy WR3000P + podkop + b4. У тебя может быть **другой провайдер, другой регион, другой роутер, другие потребности**. Промпт ниже даёт AI-ассистенту контекст моей конфигурации, и просит адаптировать её под твою — чтобы не начинать с нуля.

Скопируй блок ниже в Claude, ChatGPT или Gemini, замени `[свой контекст]` на свои данные, и попроси сгенерировать персонализированный план.

```text
Я хочу настроить OpenWrt-роутер для domain-routed VPN + DPI-обход в домашней сети.
Опираюсь на статью https://notes.kazakov.xyz/infrastructure/openwrt-router-bringup/.

Базовая модель из статьи:
- Cudy WR3000P (MT7981B, 512 MB RAM, 128 MB Flash, USB 2.0)
- OpenWrt SNAPSHOT r34310 + LuCI Master + sing-box 1.13.11 на USB
- podkop (VLESS) + b4 (DPI-обход через NFQUEUE)
- Три сегмента: LAN, Guest, IoT
- Двухуровневый watchdog + monitor-tmp + b4 auto-update

Мой контекст:
- Роутер: [модель и ревизия]
- Регион: [страна, провайдер]
- Что нужно заворачивать в VPN: [AI-сервисы / соцсети / личные домены / всё]
- Нужен ли DPI-обход для трафика мимо VPN: [да/нет/не знаю]
- Сколько у меня устройств в IoT-сегменте: [число]
- Опыт с OpenWrt: [новичок / средний / уверенный]

Что я хочу от тебя:
1. Скажи, подходит ли моя модель роутера под этот стек (проверь OpenWrt ToH).
2. Адаптируй список podkop community-lists под мой регион и use-case.
3. Подскажи, какие пункты из чек-листа в разделе 7 для меня критичны,
   а какие можно пропустить (с обоснованием почему).
4. Если у меня роутер слабее WR3000P — предложи, что урезать
   (что-то одно: b4 ИЛИ podkop, какой режим, какой объём списков).
5. Дай мне последовательность действий на ближайшие 2 часа.
```

## 9. Связанные заметки

- [[Knowledge/Infrastructure/openwrt-safe-helper-scripts\|Knowledge/Infrastructure/openwrt-safe-helper-scripts]]
- [[Knowledge/Infrastructure/Netis N6 Router/PUBLIC-GUIDE\|Knowledge/Infrastructure/Netis N6 Router/PUBLIC-GUIDE]]
- [[Knowledge/Infrastructure/vpn-instructions\|Knowledge/Infrastructure/vpn-instructions]]
- [[Knowledge/Infrastructure/yandex-reliability-principles\|Knowledge/Infrastructure/yandex-reliability-principles]]

---

*Дисклеймер / Disclaimer: material is published for informational and research purposes. [Полный отказ от ответственности / Full disclaimer](https://notes.kazakov.xyz/legal/disclaimer/).*

*Дисклеймер / Disclaimer: material is published for informational and research purposes. [Полный отказ от ответственности / Full disclaimer](https://notes.kazakov.xyz/legal/disclaimer/).*
