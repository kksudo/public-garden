---
{"title":"OpenClaw vs Cowork: два подхода к персональному AI-ассистенту","date":"2026-03-25","status":"published","tags":["public","ai","ai/agents","openclaw","cowork","comparison"],"dg-publish":true,"dg-enable-search":true,"dg-show-tags":true,"dg-permalink":"ai/openclaw-vs-cowork-ai-assistants/","url":"https://notes.kazakov.xyz/ai/openclaw-vs-cowork-ai-assistants/","source":"personal-experience","permalink":"/ai/openclaw-vs-cowork-ai-assistants/","dgEnableSearch":true,"dgShowTags":true,"dgPassFrontmatter":true}
---


# OpenClaw vs Cowork: два подхода к персональному AI-ассистенту

![assets/projects/openclaw/openclaw-vs-cowork-cover.png](/img/user/assets/projects/openclaw/openclaw-vs-cowork-cover.png)

Уже около трёх месяцев гоняю два стека рядом: **OpenClaw** ([openclaw/openclaw](https://github.com/openclaw/openclaw), порядка **336k** звёзд на момент написания) и **Cowork** (research preview от Anthropic). Задача одна и та же: читать файлы, держать контекст, выполнять работу. Внутри разные звери.

Агента на OpenClaw зову **Rurik**: дневник, фон, напоминания. На Cowork это **Jarvis**: новый тред, чистый лист, жёстче контур, зато когда открываешь, инструменты под рукой без театра.

## Архитектура: файлы workspace vs скиллы

![assets/projects/openclaw/openclaw-vs-cowork-architecture.png](/img/user/assets/projects/openclaw/openclaw-vs-cowork-architecture.png)

**OpenClaw** опирается на markdown в [workspace](https://docs.openclaw.ai/concepts/agent-workspace) (по умолчанию `~/.openclaw/workspace/`): **SOUL.md**, **AGENTS.md**, **IDENTITY.md**, **USER.md**, плюс по желанию **HEARTBEAT.md**, **MEMORY.md** и т.д. Это попадает в системную часть сессии: личность и правила в обычных файлах, их можно править и версионировать.

У OpenClaw тоже есть **скиллы** (`workspace/skills/.../SKILL.md`). То есть контраст не «без скиллов / со скиллами», а акцент: у Cowork история продукта **skills-first** и шаринг; у OpenClaw по умолчанию ощущение «эта папка и голова агента».

**Cowork**: модульные [**скиллы**](https://support.claude.com/en/articles/12512180-use-skills-in-claude): `SKILL.md`, скрипты, ассеты. Поставил, обновил, передал. Авторинг через `/skill-creator`, eval viewer, бенчмарки, A/B, если углубляться.

По **SOUL.md**: в доках про инжект в системный контекст сессии; формулировка «перед каждым сообщением» легко уводит в неточность, я её не использую.

## Память и непрерывность

**Rurik:** `memory/YYYY-MM-DD.md` для сливов, `MEMORY.md` для длинной дуги. После компакции пишет сжатое важное; в новой сессии может подтянуть назад. **Heartbeat** + **HEARTBEAT.md**: слой «проверься, даже если я молчу»; это не то же самое, что дневник, но в жизни складывается вместе.

**Cowork:** между тредами «как в чате вчера» из коробки нет. Я подключил **Obsidian**: каждый новый разговор всё равно приземляется на реальные файлы. Это не равно «помнит наш диалог вторника», но и не пустота.

## Проактивность

**Rurik:** в *моей* конфигурации тянет календарные сигналы, задачи (в т.ч. через Paperclip), пишет в мессенджеры: это **настройка, хуки, каналы**, не «магия из коробки». Heartbeat как раз делает правдоподобным сценарий «сделай что-то без моего „привет“».

**Cowork:** без меня не оживает. [Scheduled tasks](https://support.claude.com/en/articles/13345190-get-started-with-cowork) есть; для меня это ближе к cron-напоминаниям, чем ко второму мозгу, который сам бегает по делам.

## Модели и маршрутизация

**OpenClaw**, multi-provider, если правильно провести провода. У меня трафик идёт через **[OmniRoute](https://github.com/diegosouzapw/OmniRoute)** (~**1.2k** stars). Отдельный **AI gateway**: OpenAI-совместимый endpoint, маршрутизация, ретраи, фолбэки, плюс политики, лимиты, кэш, observability, чтобы не сидеть над ключами и квотами вручную. К OpenClaw не относится; это просто то, что у меня стоит перед моделями.

**Cowork** заточен под Claude (Opus / Sonnet / Haiku), зато с **sub-agents**, параллельно разные модели/задачи.

## Экосистема и интеграции

OpenClaw, open source, куча [**каналов и инструментов**](https://github.com/openclaw/openclaw) в экосистеме проекта; сообщество собирает шаблоны (например [awesome-openclaw-agents](https://github.com/mergisi/awesome-openclaw-agents)). Это больше DIY + то, что сам прикрутишь.

Cowork, продукт Anthropic с [растущей экосистемой](https://findskill.ai/blog/claude-cowork-guide/): MCP-коннекторы, маркетплейс скиллов, docx/xlsx/pdf/pptx ближе к «всё включено».

## Безопасность

**OpenClaw:** в upstream прямо сказано: для **main**-сессии инструменты по умолчанию на **хосте**. Сила и ответственность на тебе. Для групп/каналов: pairing, allowlists, **Docker sandbox для non-main** при правильной конфигурации. В каком-то черновике у меня фигурировала отсылка к Gartner («insecure by default»); **первоисточник не нашёл**, в публичный текст не тащу. Реальная модель: читать **Security** в репозитории, гонять `openclaw doctor`, не открывать DM тем, кому не дал бы shell.

(Про аналитиков и чужие заголовки ок как фон, но не как цитата без первички. Например обзоры вроде [Sangfor про OpenClaw](https://www.sangfor.com/blog/tech/openclaw-ai-agent-2026-explained): на свой риск перепроверять тезисы.)

**Cowork:** Linux VM, allowlist каталогов, жёстче граница ОС. Скучно, в хорошем смысле.

## Гибрид: что я строю

**Rurik**, операционка: что решили, что просрочено, пингануть, когда это правда важно.

**Jarvis**, исполнение: черновики постов, код, презентации.

Два места за одним столом. Не догма «выбери одно».

## Сравнительная таблица

| Критерий | OpenClaw | Cowork |
|----------|----------|--------|
| **Тип** | Open source, self-hosted | Закрытый продукт Anthropic |
| **Архитектура** | Markdown в workspace + опционально `skills/.../SKILL.md` | Скиллы (SKILL.md + скрипты + ассеты), skills-first |
| **Модели** | Несколько провайдеров; у меня через [OmniRoute](https://github.com/diegosouzapw/OmniRoute) как gateway | Claude (Opus / Sonnet / Haiku), sub-agents |
| **Память между сессиями** | `memory/`, `MEMORY.md`, компакция | Нет «как в чате» из коробки; контекст через файлы (Obsidian и т.д.) |
| **Проактивность** | Heartbeat, фон (зависит от конфига/хуков) | Scheduled tasks, в основном реакция на пользователя |
| **Каналы** | Мессенджеры и др. через Gateway | Desktop app |
| **Изоляция** | Main на хосте по умолчанию; sandbox Docker для non-main при настройке | Linux VM, явные папки |
| **Интеграции** | Каналы, инструменты, DIY | MCP, маркетплейс, офисные форматы |
| **Стоимость** | Софт бесплатно + API / подписки провайдеров | Подписка Claude (актуальные тарифы на сайте Anthropic) |
| **GitHub stars** | ~336k ([репозиторий](https://github.com/openclaw/openclaw)) | N/A |

## Бонус: промпт для любого ассистента

**Зачем:** ниже не «напиши красиво», а **короткий бриф**. Заполняешь квадратные скобки один раз; модель обязана выдать **конкретное разделение** (один стек или две роли), **жёсткие границы** («никогда не X»), **5 первых файлов/скиллов** и **три риска на первый месяц**. Меньше шансов получить очередное «зависит от всего».

```
I'm choosing how to structure my personal AI setup. I read a comparison of two styles:
(1) workspace-first agents with long-term memory, diary files, and proactive checks;
(2) isolated, tool-heavy assistants with modular skills and strong OS-level boundaries.

Help me design a concrete plan for MY workflow.

Context to fill in for you:
- What I do for work: [role / stack / main outputs]
- What I want automated vs what I must approve: [examples]
- Where my truth lives (Notion, Obsidian, filesystem, email): [tools]
- My risk tolerance (full machine access vs sandbox only): [low / medium / high]
- Whether I need continuity across days (yes/no) and why:

Deliver:
1) One paragraph: which pattern fits me as primary vs secondary (or hybrid), and why.
2) If I use two agents (e.g. "context keeper" + "executor"), define each agent's scope, what it may NEVER do, and how handoffs work.
3) A starter checklist: 5 files or skills I should create first, with one-line purpose each.
4) Three risks I should monitor in the first month (security, context drift, tool overload).

Be specific. No generic advice. Ask at most two clarifying questions only if a choice is blocked.
```

(Промпт на английском: его проще вставлять в большинство ассистентов как есть; при желании переведите заголовки deliver-секции под свою модель.)

## Выводы

Сжато: OpenClaw, когда нужна **непрерывность** и «знает мой бардак». Cowork, когда нужен **острый инструмент в маленькой комнате**.

Если тянет оба, бери оба. Вопрос к читателю: у тебя чаще ломается **контекст** или **инструменты**?

## Ссылки

- [OpenClaw (GitHub)](https://github.com/openclaw/openclaw)
- [OpenClaw Agent Workspace](https://docs.openclaw.ai/concepts/agent-workspace)
- [SOUL.md Guide](https://clawdocs.org/guides/soul-md/)
- [OmniRoute (GitHub)](https://github.com/diegosouzapw/OmniRoute)
- [Awesome OpenClaw Agents](https://github.com/mergisi/awesome-openclaw-agents)
- [Cowork, анонс](https://claude.com/blog/cowork-research-preview)
- [Cowork Getting Started](https://support.claude.com/en/articles/13345190-get-started-with-cowork)
- [Claude Skills (Help Center)](https://support.claude.com/en/articles/12512180-use-skills-in-claude)
- [Cowork (VentureBeat)](https://venturebeat.com/technology/anthropic-launches-cowork-a-claude-desktop-agent-that-works-in-your-files-no)
- [OpenClaw, обзор Sangfor](https://www.sangfor.com/blog/tech/openclaw-ai-agent-2026-explained) (вторичный источник, перепроверять формулировки)

---

*Дисклеймер / Disclaimer: material is published for informational and research purposes. [Полный отказ от ответственности / Full disclaimer](https://notes.kazakov.xyz/legal/disclaimer/).*
