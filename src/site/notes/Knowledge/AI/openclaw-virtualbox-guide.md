---
{"title":"Самый простой способ пощупать OpenClaw через VirtualBox","date":"2026-03-25","tags":["#public","#openclaw","#virtualbox","#guide","#ai"],"source":"https://habr.com/ru/articles/1001992/","author":"OpenClaw_Lab","dg-publish":true,"dg-enable-search":true,"dg-show-tags":true,"dg-permalink":"ai/openclaw-virtualbox-guide/","url":"https://notes.kazakov.xyz/ai/openclaw-virtualbox-guide/","stream":"manual","capture_status":"triaged","origin":"habr/OpenClaw_Lab","permalink":"/ai/openclaw-virtualbox-guide/","dgEnableSearch":true,"dgShowTags":true,"dgPassFrontmatter":true}
---


# Самый простой способ пощупать OpenClaw через VirtualBox

![](https://habrastorage.org/r/w1560/getpro/habr/upload_files/d07/78f/faf/d0778ffaf26a86c482dbfd838ad48881.png)

После бума люди побежали устанавливать OpenClaw на сервер, Mac mini, на всё что угодно.

Но мы забыли о старой доброй виртуалке — любой может поставить и настроить OpenClaw за несколько минут.

## Плюсы

- Бесплатно
- Полный контроль и безопасная среда
- Графический интерфейс и лёгкая автоматизация браузера
- Простой доступ к Dashboard

## Минусы

- ПК должен быть включён
- Из РФ может потребоваться VPN

## Что потребуется

1. [VirtualBox](https://www.virtualbox.org/)
2. [ISO-образ Ubuntu](https://ubuntu.com/download/desktop)

## Установка

Создаём виртуалку, указываем ISO Ubuntu, запускаем, обновляем систему:

```bash
sudo apt update && sudo apt upgrade
```

Устанавливаем OpenClaw одной командой:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

Сразу пройдёте onboarding: выбор LLM, канала связи, skills.

![Выберете LLM](https://habrastorage.org/r/w1560/getpro/habr/upload_files/6c1/d79/999/6c1d799996e460b8c8739cd4429bdb59.png)

> **Совет:** Установите Whisper сразу — голосовой ввод реально удобен.

## Проверка установки

```bash
openclaw status
```

Выведет состояние системы и Security audit. Всё что `critical` — исправьте сразу или попросите OpenClaw сделать это.

```bash
openclaw doctor
```

Исправляет ошибки конфигурации автоматически.

---

## Где живёт «душа» агента

Всё живёт в одной папке:

```bash
~/.openclaw/
```

### openclaw.json — главный конфиг

Настраивается всё:
- Модель (Claude, GPT, Gemini, локальные) + fallback
- Каналы: WhatsApp, Telegram, Discord, Slack
- Gateway, Sandbox, Heartbeat, Skills

Минимальный конфиг — одна строчка с моделью. Бэкап создаётся автоматически в `openclaw.json.bak`.

### .env — переменные окружения

API-ключи: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` и др. Ссылаться через `${VAR}`.

### credentials/ — ключи провайдеров

Закрывайте права доступа.

---

## Рабочее пространство — ~/.openclaw/workspace/

Всё это обычные Markdown-файлы, которые инжектируются в системный промпт при каждой сессии.

| Файл | Назначение |
|------|-----------|
| `AGENTS.md` | Рабочая инструкция: поведение, память, правила |
| `SOUL.md` | Душа: персональность, тон, стиль общения |
| `USER.md` | Информация о вас: предпочтения, контекст |
| `IDENTITY.md` | Визитка агента: имя, эмодзи |
| `TOOLS.md` | Заметки об инструментах и окружении |
| `HEARTBEAT.md` | Чек-лист для периодических проверок |
| `MEMORY.md` | Долгосрочная память (только в основной сессии) |
| `memory/YYYY-MM-DD.md` | Ежедневные логи, пишутся при компакции |
| `skills/<name>/SKILL.md` | Скиллы агента |
| `cron/jobs.json` | Расписание задач |

Это просто файлы. Редактируйте чем угодно, версионируйте в Git, ищите через Obsidian. Или попросите агента изменить себя:

> «Измени SOUL.md так, чтобы ты общался со мной на русском, был краток и по делу»

---

## Dashboard

```bash
openclaw dashboard
```

Веб-интерфейс прямо в браузере: состояние агента, сессии, логи, каналы. На виртуалке доступен без танцев с SSH-туннелем.

---

## Мульти-агенты

Несколько агентов с разными личностями и моделями:

```bash
openclaw agents add <имя>
openclaw agents list --bindings
```

Переключение в чате: `/agent <id>`. Каждый агент привязывается к своему каналу.

---

## Sandbox

Для групповых чатов и доступа других людей включите sandbox в `openclaw.json`:

```json
"agents": {
  "defaults": {
    "sandbox": { "mode": "non-main" }
  }
}
```

Все сессии кроме вашей основной выполняются в Docker-песочнице.

---

## Память и компакция

Когда контекст достигает ~40k токенов, агент сжимает сессию и записывает важное в `memory/YYYY-MM-DD.md`. Рутина не сохраняется.

Режим `cache-ttl` держит кэш промпта валидным 6 часов — экономит деньги на повторной обработке контекста.

---

## Hooks

Кастомные TypeScript-скрипты на события. Лежат в `workspace/hooks/` и `~/.openclaw/hooks/`.

Без hooks — агент просто отвечает. С hooks — реагирует автоматически на входящие сообщения, запуск сессий, изменения памяти.

---

## Сообщество

- Канал: [t.me/openclaw_lab](https://t.me/openclaw_lab)
- Чат: [t.me/openclaw_lab_community](https://t.me/openclaw_lab_community)

*Дисклеймер / Disclaimer: material is published for informational and research purposes. [Полный отказ от ответственности / Full disclaimer](https://notes.kazakov.xyz/legal/disclaimer/).*
