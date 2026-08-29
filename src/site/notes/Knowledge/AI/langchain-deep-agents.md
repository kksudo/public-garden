---
{"dg-publish":true,"dg-enable-search":true,"dg-show-tags":true,"dg-permalink":"ai/langchain-deep-agents/","url":"https://notes.kazakov.xyz/ai/langchain-deep-agents/","title":"LangChain Deep Agents: архитектура субагентов, планирование и что из этого можно взять для code review","date":"2026-03-19","status":"published","tags":["ai","ai/agents","ai/langchain","devops","prgate","public"],"source":"https://github.com/langchain-ai/deepagents","permalink":"/ai/langchain-deep-agents/","dgEnableSearch":true,"dgShowTags":true,"dgPassFrontmatter":true}
---


# LangChain Deep Agents: архитектура субагентов, планирование и что из этого можно взять для code review

Когда строишь multi-agent систему для code review, рано или поздно упираешься в три проблемы: как изолировать контекст между агентами, как планировать сложные задачи и как не упереться в лимит контекстного окна на длинных diff'ах. LangChain выпустили [Deep Agents](https://github.com/langchain-ai/deepagents) — open-source фреймворк, который решает именно эти задачи. Разбираю архитектуру и прикидываю, что из этого применимо в [PRGate](https://github.com/prgate).

## Что такое Deep Agents

Deep Agents — agent harness поверх [LangGraph](https://github.com/langchain-ai/langgraph). Не новый runtime и не новая reasoning-модель, а набор паттернов и встроенных инструментов вокруг стандартного tool-calling цикла. По сути — «batteries included» обёртка для создания агентов, способных работать с длинными, многошаговыми задачами.

Библиотека состоит из двух частей:

- **Deep Agents SDK** — пакет для построения кастомных агентов
- **Deep Agents CLI** — терминальный coding-агент, построенный на SDK (аналог Claude Code)

Философия — «trust the LLM». Агент может делать всё, что позволяют его tools. Границы — на уровне инструментов и sandbox'ов, а не на уровне промпта.

## Три кита архитектуры

### 1. Planning — write_todos

Встроенный инструмент `write_todos` позволяет агенту декомпозировать задачу на шаги, отслеживать прогресс и адаптировать план по ходу выполнения. Агент сам себе project manager.

Это не просто чеклист. Plan адаптивный — если на шаге 3 обнаруживается новая информация, агент пересматривает шаги 4-6. Ближайший аналог — как работает ReAct, но с явным планом вместо implicit reasoning chain.

### 2. Subagents — изоляция контекста через task tool

Ключевой паттерн. Основной агент через `task` tool спавнит дочерних агентов с изолированным контекстом. Каждый субагент видит только то, что ему передали, делает свою работу и возвращает результат.

Зачем это нужно:

- Контекст основного агента не загрязняется деталями подзадач
- Субагенты могут работать параллельно
- Каждый субагент может использовать свою модель (routing)
- Если субагент упал — основной агент изолирован от сбоя

### 3. Filesystem backend — виртуальная файловая система

Файловые инструменты (`ls`, `read_file`, `write_file`, `edit_file`) позволяют агенту скидывать большие блоки данных на «диск» вместо того, чтобы держать их в контексте. Бэкенд pluggable:

- **StateBackend** — ephemeral, живёт в state одного thread'а (по умолчанию)
- **LocalBackend** — запись на локальный диск
- **LangGraph Store** — персистентность между сессиями
- **Sandbox** — изолированное выполнение кода (Modal, Daytona, Deno)
- **CompositeBackend** — комбинация нескольких бэкендов с маршрутизацией

Это решает реальную проблему: diff на 2000 строк не влезает в контекст, но агент может записать его в файл и читать по частям.

## Middleware и управление контекстом

Deep Agents включает middleware для:

- Сжатия истории диалога (conversation compression)
- Offload больших tool results на filesystem
- Prompt caching для снижения latency и стоимости
- Изоляции контекста через субагентов

Middleware оборачивает вызовы модели и инжектит файловые инструменты автоматически. Сборка агента происходит в `libs/deepagents/deepagents/graph.py`, где собирается `CompiledStateGraph` с нужными хуками.

## Что из этого применимо в PRGate

[PRGate](https://github.com/prgate) — open-source AI code review для GitHub. Текущая архитектура уже использует multi-agent подход: оркестратор строит план ревью, собирает контекст репозитория, запускает специализированных агентов параллельно (logic, security, style), а критик фильтрует false positives.

Вот конкретные паттерны из Deep Agents, которые имеет смысл адаптировать:

### Filesystem backend для длинных diff'ов

Сейчас PRGate передаёт diff целиком в контекст агента. На больших PR это больно — контекст забивается, модель теряет фокус. Паттерн Deep Agents: записать diff в виртуальный файл, а агенту давать только summary + возможность читать конкретные chunks по запросу.

Это особенно актуально для `ContextPack` стадии, где собирается знание о репозитории: README, конфиги, связанные файлы. Вместо инжекта всего в промпт — filesystem с lazy loading.

### Адаптивное планирование

`write_todos` паттерн напрямую ложится на `ReviewPlan` стадию PRGate. Сейчас оркестратор строит план один раз. С адаптивным планированием: оркестратор строит начальный план → после первого раунда агентов обновляет план на основе findings → запускает дополнительные проверки если нужно.

Пример: security_agent находит SQL injection → оркестратор добавляет шаг «проверить все точки входа данных в этом модуле» → спавнит дополнительного агента.

### Изоляция контекста через субагенты

В PRGate агенты-специалисты уже работают параллельно. Но сейчас каждый получает полный контекст. Паттерн Deep Agents: давать каждому агенту только релевантный срез. Logic_agent получает только бизнес-логику и тесты. Security_agent — только точки входа данных и конфигурации. Style_agent — только изменённые файлы без context pack.

Меньше контекста = меньше шума = точнее findings = ниже FPR.

### Model routing на уровне субагентов

Deep Agents model-agnostic — каждый субагент может использовать свою модель. PRGate уже делает это (DeepSeek для планирования, GLM для security, Qwen для style). Но формализация через SDK-подобный интерфейс упростит добавление новых агентов и A/B-тестирование моделей.

## Чего в Deep Agents нет (и что PRGate решает иначе)

- **Нет stage-gate паттерна.** Deep Agents — это свободная декомпозиция задач. PRGate использует жёсткий 6-этапный pipeline, где каждый этап имеет определённый input/output контракт. Для code review детерминированный pipeline надёжнее свободного планирования.

- **Нет критика.** Deep Agents доверяют субагентам. В PRGate после агентов-специалистов идёт Finding Critic — отдельная модель, которая фильтрует false positives. Для code review это критично: один ложный комментарий разрушает доверие к инструменту.

- **Sandbox ≠ repository context.** Filesystem backend Deep Agents работает с файлами в вакууме. Для code review нужно понимание git истории, blame, зависимостей между модулями. Это отдельный слой, которого в Deep Agents нет.

## Ссылки

- [Deep Agents — GitHub](https://github.com/langchain-ai/deepagents)
- [Deep Agents — документация](https://docs.langchain.com/oss/python/deepagents/overview)
- [Deep Agents — блог LangChain](https://blog.langchain.com/deep-agents/)
- [Deep Agents JS](https://github.com/langchain-ai/deepagentsjs)
- [Deep Agents — PyPI](https://pypi.org/project/deepagents/)
- [PRGate — GitHub](https://github.com/prgate)
- [LangGraph](https://github.com/langchain-ai/langgraph)
- [MarkTechPost: разбор архитектуры Deep Agents](https://www.marktechpost.com/2026/03/15/langchain-releases-deep-agents-a-structured-runtime-for-planning-memory-and-context-isolation-in-multi-step-ai-agents/)

## Связанные заметки

- [[Knowledge/AI/openclaw-vs-cowork-ai-assistants\|openclaw-vs-cowork-ai-assistants]] — сравнение подходов к персональным AI-ассистентам
- [[Projects/PRGate/architecture-decisions\|Projects/PRGate/architecture-decisions]] — архитектурные решения PRGate

*Дисклеймер / Disclaimer: material is published for informational and research purposes. [Полный отказ от ответственности / Full disclaimer](https://notes.kazakov.xyz/legal/disclaimer/).*
