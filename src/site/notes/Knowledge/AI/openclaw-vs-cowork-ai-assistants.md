---
{"title":"OpenClaw vs Cowork: два подхода к персональному AI-ассистенту","date":"2026-03-18","status":"published","tags":["#public","#ai","#ai/agents","#openclaw","#cowork","#comparison"],"dg-publish":true,"dg-enable-search":true,"dg-show-tags":true,"dg-permalink":"ai/openclaw-vs-cowork-ai-assistants/","url":"https://notes.kazakov.xyz/ai/openclaw-vs-cowork-ai-assistants/","source":"personal-experience","permalink":"/ai/openclaw-vs-cowork-ai-assistants/","dgEnableSearch":true,"dgShowTags":true,"dgPassFrontmatter":true}
---


# OpenClaw vs Cowork: два подхода к персональному AI-ассистенту

![assets/projects/openclaw/openclaw-vs-cowork-cover.png](/img/user/assets/projects/openclaw/openclaw-vs-cowork-cover.png)

Последние несколько месяцев я параллельно использую два инструмента для одной и той же задачи — создания персонального AI-ассистента. OpenClaw (мой агент зовётся Rurik) и Anthropic Cowork (здесь я строю Jarvis). Оба умеют читать мои файлы, работать с Obsidian, выполнять задачи. Но архитектурно это два совершенно разных подхода. И разница не в возможностях — а в том, как ты описываешь *кто* твой агент и *как* он работает.

## Архитектура: конфиг-файлы vs скиллы

![assets/projects/openclaw/openclaw-vs-cowork-architecture.png](/img/user/assets/projects/openclaw/openclaw-vs-cowork-architecture.png)

OpenClaw определяет поведение агента через набор markdown-файлов в [рабочей директории](https://docs.openclaw.ai/concepts/agent-workspace) (`~/.openclaw/workspace/`):

- **[SOUL.md](https://clawdocs.org/guides/soul-md/)** — характер, тон, философия. Это не инструкция, а манифест: "Be genuinely helpful, not performatively helpful", "Have opinions", "Be resourceful before asking". SOUL.md инжектится в system prompt перед каждым сообщением.
- **AGENTS.md** — правила работы, безопасность, воркфлоу, протокол памяти
- **IDENTITY.md** — имя, роль, аватар (создаётся при bootstrap ритуале)
- **USER.md** — информация о пользователе, которую агент накапливает со временем

Cowork использует модульные [**скиллы**](https://support.claude.com/en/articles/12512180-use-skills-in-claude) — папки с SKILL.md и опциональными скриптами, ассетами, справочными файлами. Скилл — это пакет, который можно установить, обновить, передать другому пользователю. Создаётся через `/skill-creator` — целый фреймворк с eval viewer, бенчмарками, A/B-тестами.

Разница принципиальная. OpenClaw даёт агенту *личность*, которая живёт в workspace. Cowork даёт агенту *набор инструментов*, которые подключаются по необходимости.

## Память и непрерывность

Самое заметное отличие — как агент помнит контекст между сессиями.

В OpenClaw агент ведёт [дневник](https://docs.openclaw.ai/concepts/agent-workspace): `memory/YYYY-MM-DD.md` для сырых логов, `MEMORY.md` для кумулятивной памяти. При каждом старте читает последние записи и знает, что обсуждалось вчера. Heartbeat-механизм позволяет ему периодически ревизировать память — как человек, который перечитывает свои заметки и обновляет картину мира. MEMORY.md — это не лог, а *curated wisdom*, выжимка значимого.

В [Cowork](https://claude.com/blog/cowork-research-preview) из коробки нет памяти между сессиями. Каждый разговор начинается с нуля. Скилл может читать файлы при старте (и мой jarvis-assistant это делает — подтягивает daily notes из Obsidian), но это не то же самое, что агент, который сам ведёт дневник и помнит, что ты вчера решил сменить стратегию поиска работы.

## Проактивность

OpenClaw через heartbeat-механизм умеет действовать без запроса. Мой Rurik проверяет почту, смотрит календарь, следит за задачами в Paperclip (встроенный таск-трекер), уведомляет, если агент завис. Он знает, когда молчать (ночью, если ничего нового), а когда стоит написать ("через 2 часа встреча, ты готов?"). Работает через мессенджеры — Telegram, Discord, WhatsApp.

Cowork реактивен по природе. Есть [scheduled tasks](https://support.claude.com/en/articles/13345190-get-started-with-cowork) — можно настроить утренний брифинг по крону. Но это скорее напоминания, чем автономное поведение. Агент не мониторит твою жизнь в фоне — он ждёт, пока ты к нему придёшь.

## Модели и маршрутизация

OpenClaw model-agnostic — работает с Claude, GPT, DeepSeek и любой моделью через OpenAI-совместимое API. Через OmniRoute я маршрутизирую запросы на разные модели: `quick-response` для heartbeats, `deep-thinking` (Opus) для архитектурных решений, `coding-paid-fallback` для кодогенерации.

Cowork жёстко привязан к Claude (Opus 4.6 / Sonnet 4.6 / Haiku 4.5). Это и плюс (глубокая интеграция, оптимизированные скиллы), и минус (нет выбора модели под задачу). Зато доступны sub-agents — можно spawn'ить задачи на разных моделях параллельно.

## Экосистема и интеграции

OpenClaw — [open source](https://github.com/openclaw/openclaw) с 247k+ звёзд на GitHub. 50+ интеграций из коробки: чаты, умный дом, музыка, автоматизация. Сообщество создаёт [готовые конфиги агентов](https://github.com/mergisi/awesome-openclaw-agents) — 162 шаблона SOUL.md по 19 категориям.

Cowork — закрытый продукт Anthropic с [растущей экосистемой](https://findskill.ai/blog/claude-cowork-guide/): MCP-коннекторы (Slack, GitHub, Jira, Google Drive, Gmail, DocuSign), маркетплейс скиллов от Notion/Figma/Atlassian, встроенные скиллы для docx/xlsx/pdf/pptx. Плагины объединяют скиллы, коннекторы и sub-agents в пакеты.

## Безопасность

Тут важный нюанс. [Gartner назвал](https://www.sangfor.com/blog/tech/openclaw-ai-agent-2026-explained) дизайн OpenClaw "insecure by default", Cisco — "security nightmare". Агент имеет доступ к файлам, SSH, API-ключам — всё на совести пользователя. В AGENTS.md можно прописать правила (`trash` > `rm`, спрашивать перед доступом к секретам), но enforcement — на уровне промпта, не на уровне системы.

Cowork работает в изолированной [Linux VM](https://venturebeat.com/technology/anthropic-launches-cowork-a-claude-desktop-agent-that-works-in-your-files-no) на компьютере пользователя. Доступ к файлам — только к явно смонтированным директориям. Внешние действия требуют подтверждения. Системный промпт содержит жёсткие security rules, которые нельзя переопределить из контента.

## Гибрид: что я строю

На практике мне нужны оба подхода. Rurik — для фонового мониторинга, памяти и общения в чатах. Jarvis в Cowork — для конкретных задач: написать статью, спланировать день, подготовиться к интервью, создать презентацию.

Идеальный AI-ассистент — это не один инструмент, а оркестрация нескольких. Один следит, другой делает. Один помнит, другой создаёт.

## Сравнительная таблица

| Критерий | OpenClaw | Cowork |
|----------|----------|--------|
| **Тип** | Open source, self-hosted | Закрытый продукт Anthropic |
| **Архитектура личности** | Конфиг-файлы (SOUL.md, AGENTS.md, IDENTITY.md, USER.md) | Скиллы (SKILL.md + скрипты + ассеты) |
| **Модели** | Любые (Claude, GPT, DeepSeek) через OpenAI API | Только Claude (Opus / Sonnet / Haiku) |
| **Память между сессиями** | Встроенная (memory/, MEMORY.md) | Нет из коробки (workaround через файлы) |
| **Проактивность** | Heartbeats, фоновый мониторинг | Scheduled tasks (cron-like) |
| **Каналы общения** | Telegram, Discord, WhatsApp, Signal | Desktop app |
| **Песочница** | Нет (прямой доступ к системе) | Linux VM, изоляция |
| **Экосистема** | 50+ интеграций, community SOUL.md шаблоны | MCP-коннекторы, маркетплейс скиллов, плагины |
| **Разработка расширений** | Markdown конфиги + bash/python скрипты | skill-creator с eval framework, A/B-тестами |
| **Работа с документами** | Через внешние инструменты | Встроенные скиллы (docx, xlsx, pdf, pptx) |
| **Безопасность** | На уровне промпта (AGENTS.md) | Системная (VM + security rules + permission model) |
| **Стоимость** | Бесплатно + API costs | Claude Pro подписка ($20/мес) |
| **GitHub Stars** | 247k+ | N/A (закрытый продукт) |

## Выводы

Если выбирать один инструмент — зависит от приоритета. Хочешь агента, который *знает* тебя и работает в фоне — OpenClaw. Хочешь мощный инструмент для конкретных задач с экосистемой и безопасностью — Cowork.

Если можешь использовать оба — используй оба. Они не конкурируют, а дополняют друг друга. Как отдел из двух человек: один — операционный менеджер, второй — исполнитель.

## Ссылки

- [OpenClaw — GitHub](https://github.com/openclaw/openclaw)
- [OpenClaw — документация Agent Workspace](https://docs.openclaw.ai/concepts/agent-workspace)
- [SOUL.md Guide](https://clawdocs.org/guides/soul-md/)
- [Awesome OpenClaw Agents — 162 шаблона](https://github.com/mergisi/awesome-openclaw-agents)
- [Cowork — официальный анонс](https://claude.com/blog/cowork-research-preview)
- [Cowork — Getting Started](https://support.claude.com/en/articles/13345190-get-started-with-cowork)
- [Claude Skills — Help Center](https://support.claude.com/en/articles/12512180-use-skills-in-claude)
- [OpenClaw Security Analysis — Sangfor](https://www.sangfor.com/blog/tech/openclaw-ai-agent-2026-explained)
- [Cowork Enterprise — VentureBeat](https://venturebeat.com/technology/anthropic-launches-cowork-a-claude-desktop-agent-that-works-in-your-files-no)

---

*Дисклеймер / Disclaimer: material is published for informational and research purposes. [Полный отказ от ответственности / Full disclaimer](https://notes.kazakov.xyz/legal/disclaimer/).*
