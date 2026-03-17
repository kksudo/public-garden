---
{"dg-publish":true,"dg-enable-search":true,"dg-show-tags":true,"dg-permalink":"ai/deepmind-delegation-framework/","url":"https://notes.kazakov.xyz/ai/deepmind-delegation-framework/","title":"DeepMind — Delegation Framework для AI агентов","date":"2026-03-17","tags":["#ai","#agents","#delegation","#research","#deepmind","#public"],"source":"https://arxiv.org/abs/2602.11865","permalink":"/ai/deepmind-delegation-framework/","dgEnableSearch":true,"dgShowTags":true,"dgPassFrontmatter":true}
---


# DeepMind — Delegation Framework для AI агентов

![DeepMind Delegation Framework visual](/img/user/assets/deepmind-delegation-framework.jpg)

**Авторы:** Nenad Tomašev et al. (Google DeepMind), февраль 2026  
**Paper:** [arXiv:2602.11865](https://arxiv.org/abs/2602.11865)  
**PDF:** [arxiv.org/pdf/2602.11865](https://arxiv.org/pdf/2602.11865)  
**HTML:** [arxiv.org/html/2602.11865v1](https://arxiv.org/html/2602.11865v1)  
**DOI:** [10.48550/arXiv.2602.11865](https://doi.org/10.48550/arXiv.2602.11865)

## Цитаты из оригинала

> *"Delegation is more than just task decomposition into manageable sub-units of action. Beyond the creation of sub-tasks, delegation necessitates the assignment of responsibility and authority and thus implicates accountability for outcomes. Delegation thus involves risk assessment, which can be moderated by trust."*
>
> **Перевод:**  
> *"Делегирование — это не просто разбиение задачи на управляемые подзадачи. Помимо создания подзадач, делегирование предполагает распределение ответственности и полномочий, а значит, и подотчётность за результат. Делегирование неизбежно связано с оценкой рисков, которая может быть смягчена доверием."*


---

## Главная идея

Проблема не в том, что AI плохо работает — а в том, что люди не умеют правильно передавать\формулировать задачи.

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

## Реальный кейс: AI в первой линии поддержки

Самый массовый кейс сегодня — AI-агент в customer support.

Что делает агент:
- отвечает на типовые вопросы про доставку, возвраты, статусы заказов и тарифы;
- ищет информацию в базе знаний;
- собирает контекст перед передачей диалога человеку.

Где здесь delegation framework:
- агенту отдают только стандартные сценарии с низким риском;
- спорные, финансовые или эмоционально чувствительные кейсы эскалируются человеку;
- качество ответа проверяется по правилам: корректность, тон, соблюдение политики, успешное закрытие обращения;
- если уверенность низкая или запрос выходит за рамки полномочий, агент не импровизирует, а передаёт задачу дальше.

Это и есть живая версия идеи DeepMind: дело не в том, что AI "умеет отвечать", а в том, что система заранее решает, где ему можно доверять, а где нельзя.

---

Тут я бы сказал, спасибо КЭП, мы догадывались, но не были уверены.

*Дисклеймер / Disclaimer: material is published for informational and research purposes. [Полный отказ от ответственности / Full disclaimer](https://notes.kazakov.xyz/legal/disclaimer/).*
