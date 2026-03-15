---
{"dg-publish":true,"permalink":"/knowledge/ai/model-selection/","title":"Руководство по выбору AI моделей","tags":["ai","модели","omniroute","openclaw","public"]}
---


# Руководство по выбору AI моделей

## Комбо OpenClaw (через OmniRoute)

### chat-balanced (ПО УМОЛЧАНИЮ)
**Когда использовать:** Обычный чат, анализ, управление задачами, мониторинг
**Модели:** cc/haiku → cu/haiku → cc/sonnet → groq/llama
**Скорость:** Быстро (<1 сек для Haiku)

### quick-response
**Когда использовать:** Хартбиты, простые вопросы, проверка статуса
**Модели:** cc/haiku → groq/llama
**Скорость:** Очень быстро (<1 сек)

### deep-thinking
**Когда использовать:** Архитектурные решения, сложный анализ, многоэтапное планирование
**Модели:** cc/opus-4.5 → cc/sonnet-4.5
**Скорость:** Медленнее (5-10 сек)

### coding-paid-fallback
**Когда использовать:** Генерация кода, отладка, реализация
**Модели:** cc/opus-4.6 → cu/opus → if/deepseek-r1 → kr/sonnet → qwen/coder
**Скорость:** Среднее (3-5 сек)

## Паттерн для под-агентов
Для задачи с конкретной моделью — запустить под-агента:
```
sessions_spawn({
  task: "...",
  model: "omniroute/deep-thinking",
  runtime: "subagent"
})
```

## Сравнение моделей для чата

| Модель | Скорость | Сильные стороны | Слабые стороны |
|---|---|---|---|
| Claude Sonnet 4.5 | 2-4 сек | Универсальность, русский, инструкции | Медленнее Haiku |
| Claude Haiku 4.5 | <1 сек | Быстрый, дешёвый, хартбиты | Слабее в рассуждениях |
| Claude Opus 4.5 | 5-10 сек | Лучшие рассуждения | Медленный, дорогой |
| Groq Llama 3.3 70B | 1-2 сек | Бесплатный, быстрый | Слабее на русском |
| Qwen3 Coder Plus | 2-3 сек | HumanEval 99%, код | Не для диалога |

## Текущая конфигурация OpenClaw
```json
"primary": "anthropic/claude-sonnet-4-6",
"fallbacks": ["omniroute/chat-balanced"]
```

## Связанные заметки
- [[Knowledge/Infrastructure/omniroute-setup\|omniroute-setup]]
