---
{"title":"Как создать скилл для OpenClaw","date":"2026-03-14","tags":["#openclaw","#скилл","#разработка","#инструкция"],"source":"rurik-agent","status":"published","publish":true,"dg-publish":true,"permalink":"/knowledge/ai/openclaw-skill-creation-guide/","dgPassFrontmatter":true}
---


# Как создать скилл для OpenClaw

**Аудитория:** разработчик / опытный пользователь OpenClaw
**Время:** ~1-2 часа на первый скилл

---

## Что такое скилл

Модульный пакет, расширяющий возможности OpenClaw:

```
skill-name/
├── SKILL.md          ← обязательно (описание + инструкции)
├── scripts/          ← исполняемые скрипты (Python, Bash)
├── references/       ← справочные материалы (загружаются по необходимости)
└── assets/           ← шаблоны, файлы для вывода
```

---

## Шаг 1 — Определить скилл

Ответить на вопросы:
- Что конкретно делает скилл? (1 задача = 1 скилл)
- При каких фразах пользователя срабатывает?
- Что скилл НЕ делает?

**Пример:**
> `media-notes`: скачивает видео → транскрибирует → сохраняет в Obsidian.
> Срабатывает: "сохрани в notes", "разбери этот рил".
> НЕ делает: PDF, текстовые статьи.

---

## Шаг 2 — Инициализировать структуру

```bash
python3 /opt/homebrew/lib/node_modules/openclaw/skills/skill-creator/scripts/init_skill.py \
  my-skill \
  --path /opt/homebrew/lib/node_modules/openclaw/skills \
  --resources scripts,references
```

Результат: готовая папка с шаблоном `SKILL.md`.

---

## Шаг 3 — Написать SKILL.md

### Frontmatter — критически важен (это триггер)

```yaml
---
name: my-skill
description: "Что делает скилл. Используй когда: конкретные фразы пользователя.
  НЕ для: что скилл не делает."
---
```

**Правила description:**
- Без `<` и `>` — валидатор не пропустит
- Включить триггерные фразы на русском и английском
- Указать что скилл НЕ делает

### Тело SKILL.md — писать кратко

```markdown
# Название

## Quick Start
[минимум для запуска — 3-5 строк]

## Workflow
1. Шаг один
2. Шаг два

## Ошибки
- Ошибка → решение
```

---

## Шаг 4 — Написать скрипты

```bash
#!/usr/bin/env bash
set -euo pipefail
# ... логика
```

**Правило:** тестировать каждый скрипт до упаковки.

```bash
bash /opt/homebrew/lib/node_modules/openclaw/skills/my-skill/scripts/my-script.sh
```

---

## Шаг 5 — Упаковать

```bash
python3 /opt/homebrew/lib/node_modules/openclaw/skills/skill-creator/scripts/package_skill.py \
  /opt/homebrew/lib/node_modules/openclaw/skills/my-skill \
  /tmp/dist

openclaw gateway restart
```

---

## Шаг 6 — Проверить

```bash
ls /opt/homebrew/lib/node_modules/openclaw/skills/my-skill/
# Написать в OpenClaw фразу-триггер и проверить что скилл сработал
```

---

## Частые ошибки

| Ошибка | Причина | Решение |
|---|---|---|
| `angle brackets` | `<` или `>` в description | Заменить на текст |
| Скилл не срабатывает | Слабый description | Добавить конкретные фразы |
| Скрипт не запускается | Нет `chmod +x` | `chmod +x scripts/*.sh` |
| Пути не работают | Относительные пути | Использовать абсолютные |

---

## Чеклист перед релизом

- [ ] `description` содержит триггерные фразы на русском
- [ ] Нет `<` и `>` в frontmatter
- [ ] Скрипты протестированы
- [ ] `SKILL.md` до 500 строк
- [ ] Валидатор проходит без ошибок

---

## Реальный пример

Скилл `media-notes` создан 2026-03-14:
- Путь: `/opt/homebrew/lib/node_modules/openclaw/skills/media-notes/`
- Инструменты: yt-dlp + Whisper
- Триггер: "сохрани в notes", "разбери этот рил"

## Связанные заметки
- [[Knowledge/AI/obsidian-skill-draft\|obsidian-skill-draft]]
- [[Knowledge/AI/obsidian-openclaw-integration\|obsidian-openclaw-integration]]

*Дисклеймер / Disclaimer: material is published for informational and research purposes. [Полный отказ от ответственности / Full disclaimer](https://notes.kazakov.xyz/legal/disclaimer/).*
