---
{"dg-publish":true,"dg-enable-search":true,"dg-show-tags":true,"dg-permalink":"/ai/deepmind-delegation-framework/","url":"https://notes.kazakov.xyz/knowledge/ai/deepmind-delegation-framework/","title":"DeepMind — Delegation Framework для AI агентов","date":"2026-03-17","tags":["ai","agents","delegation","research","deepmind","public"],"source":"https://arxiv.org/abs/2602.11865","permalink":"/ai/deepmind-delegation-framework/","dgEnableSearch":true,"dgShowTags":true,"dgPassFrontmatter":true}
---


# DeepMind — Delegation Framework для AI агентов

**Авторы:** Nenad Tomašev et al. (Google DeepMind), февраль 2026  
**Paper:** [arXiv:2602.11865](https://arxiv.org/abs/2602.11865)  
**PDF:** [arxiv.org/pdf/2602.11865](https://arxiv.org/pdf/2602.11865)  
**HTML:** [arxiv.org/html/2602.11865v1](https://arxiv.org/html/2602.11865v1)  
**DOI:** [10.48550/arXiv.2602.11865](https://doi.org/10.48550/arXiv.2602.11865)

---

## Цитаты из оригинала

> *"Delegation is more than just task decomposition into manageable sub-units of action. Beyond the creation of sub-tasks, delegation necessitates the assignment of responsibility and authority and thus implicates accountability for outcomes. Delegation thus involves risk assessment, which can be moderated by trust."*

> *"To fully utilize AI agents, we need intelligent delegation: a robust framework centered around clear roles, boundaries, reputation, trust, transparency, certifiable agentic capabilities, verifiable task execution, and scalable task distribution."*

---

## Главная идея

Проблема не в том, что AI плохо работает — проблема в том, что люди не умеют правильно передавать задачи.

**Делегирование = управление риском**, а не просто промпт.

## 4 решения при делегировании

1. Нужно ли вообще отдавать задачу AI?
2. Как правильно её сформулировать?
3. Как проверить результат?
4. Что делать, если AI ошибся?

---

## Ключевые концепции

### Over/Under-delegation
- **Over-delegation** — отдаём AI задачи, к которым он не готов
- **Under-delegation** — делаем сами то, что AI уже умеет лучше
- Будущее эффективности — в правильном балансе

### Рынок AI-агентов
- Агенты соревнуются за задачи
- Оценивают свою способность выполнить их
- Подтверждают навыки **цифровыми сертификатами** (криптографически)
- Не рейтинг, а верифицированная компетенция

### Обязательная валидация
- Правила, когда ответ можно принять
- **Confidence score** — оценка уверенности модели
- Резервные сценарии при ошибке
- **Никогда не принимать результат AI без валидации**

### Динамическое делегирование
- Ответственность может передаваться в процессе
- Задачи перераспределяются при сбоях
- Система адаптируется к изменениям

### AI → AI → AI цепочки
- Сохраняется ответственность на каждом уровне
- Отслеживается кто за что отвечает
- Не теряется контроль при вложенных агентах

---

## Применение в наших проектах

### Social Warmup Bot
- **Конкурентная модель копирайтеров**: Leo И Tara оба получают задачу → Victor выбирает лучший вариант
- **Failover**: если Diana (Platform Adapter) упала → задача перераспределяется автоматически
- Victor (Chief Critic) = обязательный validation layer по DeepMind

### PRGate
- Добавить **confidence score** к каждому ревью агента
- Агент сообщает уверенность → Quinn решает нужна ли итерация

### Взаимодействие Кирилл ↔ Rurik
- Кирилл даёт **цель + контекст** → Rurik исполняет
- Финальные решения по рискованным вопросам — за Кириллом (PER-17, PER-18)
- Избегать over-delegation: не давать задачи без достаточного контекста
