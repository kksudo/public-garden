---
{"dg-publish":true,"dg-enable-search":true,"dg-show-tags":true,"dg-permalink":"infrastructure/openwrt-safe-helper-scripts/","url":"https://notes.kazakov.xyz/infrastructure/openwrt-safe-helper-scripts/","title":"Безопасные shell-скрипты для OpenWrt","date":"2026-03-19","status":"published","tags":["#infra/openwrt","#infra/router","#infra/automation","#content/public","#public"],"source":"https://openwrt.org/","permalink":"/infrastructure/openwrt-safe-helper-scripts/","dgEnableSearch":true,"dgShowTags":true,"dgPassFrontmatter":true}
---


# Безопасные shell-скрипты для OpenWrt

> **TL;DR:** на слабом роутере shell-скрипт должен сначала собрать новый конфиг, потом провалидировать его, и только после этого аккуратно применить изменения и проверить сервис.

## Главная идея

На обычном Linux неудачный shell-скрипт часто просто ломает один сервис.

На домашнем роутере под OpenWrt цена ошибки выше:

- можно потерять DNS для всей сети
- можно положить интернет дома из-за одного `service restart`
- можно забить `/tmp`, который живёт в RAM
- можно тихо начать писать на USB вместо planned path и долго не замечать деградацию

Поэтому хороший helper script для роутера должен работать не в стиле "поправил файл и рестартанул", а в стиле:

1. собрать candidate-конфиг
2. сравнить его с текущим
3. провалидировать
4. применить атомарно
5. сделать health check
6. откатиться, если что-то пошло не так

## Ключевые моменты

- Скрипт должен быть **идемпотентным**: повторный запуск не меняет систему, если входные данные не изменились.
- Нужны как минимум режимы `--check` или `--dry-run` и `--apply`.
- Live-файлы нельзя править напрямую через `sed -i` без staged-copy и проверки.
- `restart` или `reload` допустимы только если реально есть diff.
- До применения нужно использовать нативную проверку конфига: `dnsmasq --test`, `sing-box check`, `nft -c`.
- После рестарта нужен не только `pidof`, но и короткий функциональный тест.
- Если скрипт зависит от USB, он не должен тихо откатываться в `/tmp`.
- Cron должен вызывать только "безопасные" entrypoint-скрипты, а не сырые ad-hoc команды.

## Детали

### Почему staged apply важнее, чем "правка по месту"

Самая частая ошибка в таких скриптах: взять боевой конфиг, переписать его на месте и сразу сделать restart.

Проблема в том, что при любой синтаксической ошибке ты получаешь сразу две проблемы:

- файл уже испорчен
- сервис уже перезапущен и может не подняться

Надёжнее делать так:

1. собрать новый конфиг во временный файл
2. если он совпадает с текущим, выйти без изменений
3. прогнать проверку syntax/config test
4. сохранить backup текущей версии
5. атомарно заменить через `mv`
6. сделать restart только после успешной валидации

Этот подход особенно хорошо работает для `dnsmasq`, `nftables`, `sing-box` и любых generated snippets.

### Минимальный безопасный контракт для helper script

Каждый скрипт, который меняет боевую конфигурацию, должен поддерживать такие свойства:

- **Idempotent**: нет изменений -> нет restart
- **Check mode**: можно проверить candidate без применения
- **Apply mode**: реальное применение
- **Rollback**: возврат на предыдущую версию при неуспехе
- **Locking**: защита от параллельного запуска из cron и manual mode
- **Resource gate**: проверка `/tmp` и `/mnt/usb`
- **Structured logging**: `no changes`, `validated`, `applied`, `skipped`, `rollback`, `failed`

Если чего-то из этого нет, скрипт лучше не ставить в cron.

### Validate before reload

Для типовых сервисов OpenWrt можно использовать такие проверки:

- `dnsmasq`: `dnsmasq --test --conf-file=...`
- `sing-box`: `sing-box check -c /etc/sing-box/config.json`
- `nftables`: `nft -c -f <file>`

Важно: сначала validate, потом apply, потом restart.

Не наоборот.

### Restart только при реальном diff

Один из лучших паттернов для роутера: не трогать сервис, если состояние по сути не изменилось.

Это даёт сразу несколько плюсов:

- меньше ложных рестартов
- меньше шансов попасть в race condition
- меньше нагрузки на CPU и RAM
- меньше риска словить каскадную поломку DNS или proxy path

Практическое правило простое: если `candidate == current`, скрипт пишет в лог `no changes` и завершает работу.

### Health check после restart

Проверка только через `pidof` недостаточна.

Процесс может существовать, но сервис уже быть функционально сломанным. После restart нужен короткий smoke test:

- для `dnsmasq`: реальный DNS-запрос
- для `sing-box`: `check` плюс тестовый запрос через нужный outbound path
- для `nftables`: наличие нужной таблицы, цепочки или набора

Если smoke test не прошёл:

1. вернуть backup
2. повторно поднять сервис
3. если сервис всё равно не поднялся, остановиться и не делать дальнейших изменений

### Никакого тихого fallback в `/tmp`

Это особенно важно для OpenWrt на железе с 256 МБ RAM.

Если план был такой:

- логи на USB
- cache на USB
- generated files на USB

то при пропавшем `/mnt/usb` скрипт не должен молча писать в `/tmp`. Иначе ты получаешь:

- рост tmpfs
- нестабильность
- ENOSPC
- каскадные отказы соседних сервисов

Правильное поведение в таких случаях: `log critical`, `skip apply`, `leave direct connectivity alive`.

### Locking против параллельного запуска

На роутере очень легко случайно столкнуть:

- cron
- `rc.local`
- ручной запуск по SSH

Если один и тот же скрипт умеет менять конфиг и рестартить сервис, параллельный запуск опасен. Самая простая защита - `lockdir`:

```sh
LOCKDIR="/var/run/my-script.lock"
if ! mkdir "$LOCKDIR" 2>/dev/null; then
    logger -t my-script "another instance is already running; skipping"
    exit 0
fi
trap 'rmdir "$LOCKDIR" 2>/dev/null || true' EXIT INT TERM
```

Для OpenWrt этого обычно достаточно.

### Что считать хорошим эталоном

Хороший образец - скрипт, который:

- пишет candidate snippet во временный файл
- сравнивает с текущим
- запускает `dnsmasq --test`
- only-on-diff делает restart
- проверяет свободное место в `/tmp`
- логирует результат в syslog

Именно такой паттерн стоит использовать как базу для всех остальных helper scripts.

## Практический чеклист

Перед тем как поставить shell-скрипт в cron, стоит пройтись по такому списку:

- У скрипта есть `--check` или `--dry-run`
- Есть staged apply, а не правка live-файла по месту
- Есть config validation до restart
- Есть backup и rollback
- Есть lock от параллельного запуска
- Есть проверка `/tmp`
- Есть проверка `/mnt/usb`, если он нужен
- Restart происходит только при diff
- После restart есть functional test
- Логи читаемы через `logread`

Если на любой из этих вопросов ответ "нет", скрипт ещё сырой.

## Связанные заметки

- [[Knowledge/Infrastructure/Netis N6 Router/PUBLIC-GUIDE\|Knowledge/Infrastructure/Netis N6 Router/PUBLIC-GUIDE]]
- [[Knowledge/Infrastructure/yandex-reliability-principles\|Knowledge/Infrastructure/yandex-reliability-principles]]

*Дисклеймер / Disclaimer: material is published for informational and research purposes. [Полный отказ от ответственности / Full disclaimer](https://notes.kazakov.xyz/legal/disclaimer/).*
