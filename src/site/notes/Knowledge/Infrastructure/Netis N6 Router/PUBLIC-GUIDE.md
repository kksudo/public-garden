---
{"dg-publish":true,"dg-permalink":"infrastructure/netis-n6-openwrt/","url":"https://notes.kazakov.xyz/infrastructure/netis-n6-openwrt/","dg-enable-search":true,"dg-show-tags":true,"title":"OpenWrt на Netis N6: DNS, VPN по доменам, обход DPI и стабильность","date":"2026-03-16","tags":["infrastructure","openwrt","router","vpn","dpi","public"],"permalink":"/infrastructure/netis-n6-openwrt/","dgEnableSearch":true,"dgShowTags":true,"dgPassFrontmatter":true}
---


# OpenWrt на Netis N6: DNS, VPN по доменам, обход DPI и стабильность в одной связке

![Netis N6 — роутер с OpenWrt](https://megaobzor.com/uploads/stories/192083/P1054493.webp)

*Фото: [обзор Netis N6 на MegaObzor](https://megaobzor.com/review-netis-N6.html)*

## Зачем и что за железо

Задача — один домашний роутер: часть трафика в VPN (AI, сервисы), зарубежные сайты без DPI-блокировок, российские — без капчи и банов. Всё на своём железе, без облачных подписок на каждый сценарий.

Гайд для тех, у кого уже стоит OpenWrt на Netis N6 (или аналог на MT7621) и есть базовое понимание UCI и SSH. Пошаговой установки OpenWrt с нуля здесь нет.

Роутер — **Netis N6**, взят на Озоне примерно за **2500 ₽**. USB 3.0, Wi-Fi 6, платформа MT7621 — под [OpenWrt](https://openwrt.org/) тянет и DoH, и прокси, и обход DPI.

## Что должно быть уже сделано

Перед повторением конфига нужно иметь: прошивку OpenWrt 24.10.x (или совместимую), установленные пакеты — https-dns-proxy, podkop, sing-box, b4, logrotate, watchcat, tune2fs (e2fsprogs), USB-накопитель смонтирован в `/mnt/usb`. Полная карта файлов, пути и команды отката — в исходной документации (README и разделы 01–07 в том же репозитории или архиве, где лежит этот гайд).


## Как устроена связка

Роутер на MT7621/mt7915 под [OpenWrt](https://openwrt.org/) 24.10.x.

- **DNS** не завязан на VPN — интернет живёт, даже если прокси падает.
- Часть трафика уходит в **VPN** по списку доменов (AI, копайлот, сервисы).
- Зарубежный HTTPS — через **DPI-bypass ([b4](https://github.com/DanielLavrushin/b4))**.
- Российские сайты в обход не трогаем — без капчи и блокировок.

## DNS

Резолвинг идёт через [https-dns-proxy](https://docs.openwrt.melmac.net/https-dns-proxy/) на порты `5053` / `5054`, dnsmasq отдаёт клиентам только их. [Podkop](https://github.com/itdoginfo/podkop) не трогает DHCP (`dont_touch_dhcp=1`).

**Итог:** при падении [sing-box](https://sing-box.sagernet.org/) остаёшься с рабочим интернетом.

**Важно:** не рестартовать dnsmasq без нужды и не трогать его, когда в `/tmp` мало места. Почему: `/tmp` — это tmpfs в RAM; когда sing-box или кто-то забивал его (например, кеш), dnsmasq при рестарте не мог создать pidfile → DNS отваливался каскадом. Скрипты синхронизации проверяют свободное место и не дергают dnsmasq, если nftset не менялся.

## VPN по доменам

dnsmasq резолвит домены → IP попадают в nft set → [podkop](https://github.com/itdoginfo/podkop) гонит трафик через [sing-box](https://sing-box.sagernet.org/).

- Список доменов хранится в UCI: `podkop.main.user_domains_text`.
- Скрипт `/root/bin/sync-podkop-nftset.sh` пишет в `/etc/dnsmasq.conf` блок (между маркерами `podkop-nftset-sync start` / `end`): для каждого домена из UCI — строка с `nftset=` в нужную nft table#set, чтобы резолвенные IP попадали в set и podkop вёл их через sing-box. Рестарт dnsmasq только при реальном изменении блока и при достаточном свободном месте в `/tmp`.
- Cron в **10** и **40** минут плюс `rc.local` после бута восстанавливают правила.

**Добавить домен:** дописать в `user_domains_text`, выполнить sync-скрипт. Без ручной правки dnsmasq.

## ECH

[ECH](https://datatracker.ietf.org/doc/draft-ietf-tls-esni/) (Encrypted Client Hello) скрывает SNI — [sing-box](https://sing-box.sagernet.org/) не матчит правила и дропает трафик.

**Решение:** tproxy catch-all в конце маршрутов: весь непойманный tproxy-трафик уходит в прокси. В конец `route.rules` в `/etc/sing-box/config.json` скрипт добавляет правило (идемпотентно, без дубликатов):

```json
{"inbound":"tproxy-in","outbound":"proxy"}
```

`outbound` — имя прокси-аутбаунда в твоём конфиге (скрипт подставляет первый не-direct outbound из конфига). Скрипт: `/root/bin/patch-singbox-tproxy-catchall.sh`. Хук в [podkop](https://github.com/itdoginfo/podkop) при сохранении конфига + cron раз в **15** минут + запуск через **60 с** после бута в `rc.local` — правило не теряется.

<details>
<summary>Скрипт <code>patch-singbox-tproxy-catchall.sh</code> (полный текст)</summary>

```sh
#!/bin/sh
# Adds catch-all route rule: all tproxy-in traffic -> proxy outbound
# Usage: patch-singbox-tproxy-catchall.sh [config-file]
# If no argument, patches /etc/sing-box/config.json

CONFIG="${1:-/etc/sing-box/config.json}"
[ -f "$CONFIG" ] || exit 0

# Skip if already patched (last rule is catch-all for tproxy-in without rule_set)
jq -e '.route.rules[-1] | select(.inbound == "tproxy-in" and (.rule_set | not))' "$CONFIG" >/dev/null 2>&1 && exit 0

OUTBOUND=$(jq -r '[.outbounds[] | select(.type != "direct")][0].tag' "$CONFIG" 2>/dev/null)
[ -z "$OUTBOUND" ] && exit 1

jq --arg out "$OUTBOUND" '.route.rules += [{"action":"route","inbound":"tproxy-in","outbound":$out}]' "$CONFIG" > "${CONFIG}.tmp" && mv "${CONFIG}.tmp" "$CONFIG"

logger -t singbox-patch "Added tproxy-in catch-all -> $OUTBOUND in $CONFIG"
```
</details>

## b4 и российские сайты

[b4 (ByeDPI)](https://github.com/DanielLavrushin/b4) направляет весь трафик на порт 443 через NFQUEUE; в результате для Яндекса и других российских сервисов появлялись капчи и возникали блокировки.

**Патч:** nft set **`b4_domestic`** в таблице `inet b4_mangle`. Для IP из этого сета — return в цепях b4 (prerouting/postrouting), трафик не идёт в NFQUEUE. Скрипт: `/root/bin/b4-domestic-bypass.sh`, вызывается из `/etc/init.d/b4` в `start()` (ожидание появления `b4_mangle` до 15 с).

**Статичные подсети РУ** (Yandex, VK, Mail.ru, Рамблер), добавляемые в `b4_domestic`:

```
5.45.192.0/18  5.255.192.0/18  77.88.0.0/18
87.240.128.0/17  87.250.224.0/19  93.158.128.0/18
93.186.224.0/20  94.100.176.0/20  95.108.128.0/17
95.142.192.0/20  95.163.32.0/19  141.8.128.0/18
178.154.128.0/17  213.180.192.0/19  217.69.128.0/20
```

**Динамический bypass:** [allow-domains](https://github.com/itdoginfo/allow-domains) — список `Russia/outside-dnsmasq-nfset.lst`. Скрипт `/root/bin/update-allow-domains.sh` качает его на USB, заменой `fw4#vpn_domains` → `b4_mangle#b4_domestic` превращает в конфиг dnsmasq и кладёт в `conf.d` (например `/tmp/dnsmasq.cfg*.d/b4-domestic-domains.conf`). По резолву IP домена попадают в `b4_domestic`. Cron 4:17, после бута — восстановление из USB в tmpfs.

- РУ-трафик идёт **мимо** b4.
- Зарубежный — через DPI-обход.
- Логи b4 на USB, уровень логирования понижен (SILENT): меньше записей — меньше нагрузка на CPU и флеш, ошибки всё равно пишутся в отдельный файл.

## Стабильность

**Ядро и память:** sysctl — буферы netlink 2 МБ, `swappiness` 10. Зачем: тяжёлые nft-операции (podkop, b4) давали `netlink: buffer overflow`; увеличение буферов убирает ошибки. Swappiness 10 — swap на USB, при значении по умолчанию ядро слишком охотно сбрасывает туда память, флешка изнашивается и тормозит.

**Логи:** в `/mnt/usb/var/log/`, ротация через [logrotate](https://github.com/logrotate/logrotate) при **512 КБ**, раз в час по крону. Зачем: логи на USB переживают ребут и не жрут RAM; лимит 512 КБ не даёт одному файлу раздуваться и забивать флешку.

**Cron** разнесён по минутам. Зачем: на 256 МБ RAM несколько тяжёлых задач в одну минуту могут дать OOM или переполнение tmpfs.

- sync доменов — 10, 40 мин;
- патч ECH — каждые 15 мин;
- allow-domains — 4:17;
- usb-health — раз в 6 ч.

После бута ждём **60 с**, потом sync и патч — за это время podkop и sing-box успевают подняться, иначе скрипты отработают впустую или сломают конфиг.

**Мониторинг sing-box:** размер `cache.db` (пороги 1 / 5 / 10 МБ) и свободное место в `/tmp` (16 / 8 / 4 МБ), раз в 5 мин, в лог только при пересечении порогов. Зачем: кеш sing-box (`cache.db`) лежит в `/tmp` (RAM) и при росте выедал всю доступную память → ENOSPC → падал и sing-box, и dnsmasq, пропадал DNS. Мониторинг даёт увидеть рост до срыва и при необходимости отреагировать.

**USB:** mount с `noatime` и `commit=120` (сброс журнала ext4 раз в 2 мин). Зачем: `noatime` — при каждом чтении не обновлять время доступа, меньше записей и износ флешки. `commit=120` вместо дефолтных 5 сек — журнал сбрасывается реже, записей в разы меньше (при сбое питания можно потерять до 2 мин данных — для логов приемлемо). Проверка здоровья раз в 6 ч ([tune2fs](https://e2fsprogs.sourceforge.net/), запись-чтение тестового файла, порог по записи **50 ГБ**) — чтобы заметить умирающую флешку или переполнение до того, как всё упадёт.

## Wi‑Fi и система

- **2.4 ГГц** — HT20 вместо HE20: на старых устройствах реже обрывы и дисконнекты.
- **5 ГГц** — страна RU (корректные каналы и мощность для РФ), txpower 22 вместо 23: меньше интерференции, чище сигнал в плотной застройке.
- Логи системы на USB, буфер **1024 КБ**: больше истории для разбора крашей и ребутов, логи не теряются при перезагрузке.
- **[watchcat](https://openwrt.org/docs/guide-user/advanced/watchcat)** пингует 1.1.1.1 и 8.8.8.8 каждые 15 с; при потере связи запускает скрипт восстановления (dnsmasq, podkop, b4) — интернет возвращается без ручного входа.
- IPv6 отключён — провайдер не раздаёт, лишние процессы и объявления в LAN не нужны.

## Ограничения и риски

Настройка затрагивает весь домашний трафик: ошибка может оставить дом без интернета. Имеет смысл иметь запасной канал (мобильный интернет) для отладки и бэкап конфигов перед изменениями. После `opkg upgrade` пакетов (podkop, b4 и др.) хуки и вызовы в init-скриптах могут перезаписаться — проверяй и при необходимости восстанавливай скрипты и вызовы из документации.

## Доступ

SSH на свой порт. При необходимости — смонтировать документы с роутера через SSHFS ([macFUSE](https://macfuse.io/) + [sshfs](https://github.com/libfuse/sshfs)).

**Проверка, что всё работает:** (1) Выключи sing-box или podkop — DNS и обычный сёрф должны остаться (резолв через https-dns-proxy). (2) Зайди на Яндекс — без капчи и блокировок (трафик в bypass). (3) Открой сайт с ECH (например, за фронтом Cloudflare) — страница открывается через VPN, а не дропается.

## Ссылки

| Решение | Назначение |
|--------|------------|
| [OpenWrt](https://openwrt.org/) | Фирмварь роутера |
| [https-dns-proxy](https://docs.openwrt.melmac.net/https-dns-proxy/) | DNS-over-HTTPS для dnsmasq |
| [podkop](https://github.com/itdoginfo/podkop) | Роутинг доменов в VPN через nftset + sing-box |
| [sing-box](https://sing-box.sagernet.org/) | Универсальный прокси |
| [b4 (ByeDPI)](https://github.com/DanielLavrushin/b4) | Обход DPI (NFQUEUE, 443) |
| [allow-domains](https://github.com/itdoginfo/allow-domains) | Список РУ доменов для bypass b4 |
| [ECH (draft)](https://datatracker.ietf.org/doc/draft-ietf-tls-esni/) | Encrypted Client Hello |
| [logrotate](https://github.com/logrotate/logrotate) | Ротация логов |
| [tune2fs / e2fsprogs](https://e2fsprogs.sourceforge.net/) | Статистика ext4, проверка USB |
| [watchcat](https://openwrt.org/docs/guide-user/advanced/watchcat) | Ping-мониторинг, автовосстановление (OpenWrt) |
| [macFUSE](https://macfuse.io/) | Монтирование SSHFS на macOS |
| [sshfs](https://github.com/libfuse/sshfs) | Файловая система поверх SSH |

---

<details>
<summary>Промпт для AI-агента: внести оптимизации</summary>

Скопируй один из вариантов ниже и передай агенту вместе с этим документом или ссылкой на него.

**Русский:**

```
Контекст: этот документ описывает конфигурацию OpenWrt 24.10.x на роутере Netis N6 (MT7621/mt7915, 256 МБ RAM). Стек: https-dns-proxy + dnsmasq, podkop + sing-box (VPN по доменам), b4 (DPI-bypass) с bypass российских IP, ECH catch-all, логи и кеш на USB, logrotate, watchcat, cron и rc.local.

Задача: предложи и при необходимости внедри дополнительные оптимизации под это железо и стек. Учитывай: ограничение RAM и tmpfs (/tmp в памяти), износ USB-флешки, стабильность DNS при падении прокси, каскадные рестарты, netlink/nft нагрузку. Для каждой оптимизации кратко поясни, зачем она нужна и какой эффект даёт. Используй разделы «Стабильность», «DNS», «b4 и российские сайты» этого гайда как образец формата: описание меры + блок «Зачем». Ориентируйся на практики из гайда (sysctl, разнесённый cron, проверка свободного места перед рестартом dnsmasq, мониторинг cache.db, noatime/commit на USB, пороги по размеру логов). Не меняй логику работы связки (DNS отдельно от VPN, домены в UCI, b4_domestic и т.д.) — только тюнинг, мониторинг и защитные меры.
```

**English (for broader model compatibility):**

```
Context: This document describes an OpenWrt 24.10.x setup on a Netis N6 router (MT7621/mt7915, 256 MB RAM). Stack: https-dns-proxy + dnsmasq, podkop + sing-box (domain-based VPN), b4 (DPI bypass) with bypass for Russian IPs, ECH catch-all, logs and cache on USB, logrotate, watchcat, cron and rc.local.

Task: Propose and, if needed, implement additional optimizations for this hardware and stack. Consider: RAM and tmpfs limits (/tmp is in RAM), USB flash wear, DNS stability when the proxy fails, cascading restarts, netlink/nft load. For each optimization briefly explain why it is needed and what effect it has. Use this guide’s sections “Стабильность”, “DNS”, “b4 и российские сайты” as the format template: describe the measure + a “Зачем” (why) block. Follow the practices from the guide (sysctl, staggered cron, checking free space before restarting dnsmasq, cache.db monitoring, noatime/commit on USB, log size thresholds). Do not change the overall logic (DNS independent of VPN, domains in UCI, b4_domestic, etc.) — only tuning, monitoring, and safeguards.
```
</details>

---
*Дисклеймер / Disclaimer: material is published for informational and research purposes. [Полный отказ от ответственности / Full disclaimer](https://notes.kazakov.xyz/legal/disclaimer/).*
