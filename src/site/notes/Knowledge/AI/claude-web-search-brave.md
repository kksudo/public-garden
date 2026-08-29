---
{"dg-publish":true,"dg-enable-search":true,"dg-show-tags":true,"dg-permalink":"ai/claude-web-search-brave/","url":"https://notes.kazakov.xyz/ai/claude-web-search-brave/","title":"Как устроен веб-поиск в Claude — Brave Search под капотом","date":"2026-03-18","tags":["ai","claude","anthropic","search","brave","infrastructure","public"],"source":"https://techcrunch.com/2025/03/21/anthropic-appears-to-be-using-brave-to-power-web-searches-for-its-claude-chatbot/","status":"published","permalink":"/ai/claude-web-search-brave/","dgEnableSearch":true,"dgShowTags":true,"dgPassFrontmatter":true}
---


# Как устроен веб-поиск в Claude — Brave Search под капотом

## Контекст

Когда AI-модели (ChatGPT, Claude, Gemini) ищут что-то в интернете по запросу пользователя — они не гуглят. У них физически нет доступа к Google Search так, как он есть у нас в браузере.

Google жёстко ограничивает программный доступ к своей выдаче. Custom Search JSON API платный, с лимитами, и не даёт тот же охват, что google.com. Скрейпинг выдачи — прямое нарушение ToS. Поэтому AI-компании вынуждены использовать альтернативные поисковые API.

---

## Что использует Claude

В марте 2025 года Anthropic добавила **Brave Search** в свой список субпроцессоров — партнёров, обрабатывающих данные пользователей.

Подтверждения:

- В коде обнаружен параметр `BraveSearchParams`
- Исследователи провели статистический анализ: **86.7% результатов** Claude совпадают с выдачей Brave (13 из 15 результатов)
- TechCrunch подтвердил находку в [статье от 21 марта 2025](https://techcrunch.com/2025/03/21/anthropic-appears-to-be-using-brave-to-power-web-searches-for-its-claude-chatbot/)

Нюанс: текстовый поиск идёт через **Brave**, а поиск изображений — через **Bing API**. Гибридная схема.

---

## Почему Brave, а не Google или Bing

- **Собственный независимый индекс** — не обёртка над Google/Bing
- Адекватная стоимость API для больших объёмов
- Нет жёстких ограничений Google на программный доступ
- Privacy-first подход совпадает с позиционированием Anthropic

---

## Как это работает в Cowork/Claude

Цепочка: пользователь задаёт вопрос → Claude формирует поисковый запрос → инструмент WebSearch вызывает Brave Search API → возвращаются заголовки, URL, сниппеты → при необходимости WebFetch загружает конкретные страницы для глубокого анализа.

---

## Ограничения, которые я заметил на практике

- **Egress proxy блокирует ряд доменов** — github.com, некоторые .su домены недоступны через WebFetch
- Когда страница заблокирована, приходится опираться только на сниппеты из поисковой выдачи
- WebSearch доступен только в US (по документации)
- Результаты кэшируются на 15 минут

---

## Рынок поисковых API для AI

Основные игроки, которые продают поиск AI-компаниям:

- **Brave Search API** — Claude (Anthropic)
- **Bing Web Search API** — Copilot (Microsoft), ранее Perplexity
- **Tavily** — популярен в open-source агентах (LangChain и др.)
- **SerpAPI, You.com API** — используются в различных интеграциях

Получается ирония: Google — крупнейший поисковик мира, но AI-компании массово уходят к конкурентам, потому что Google не хочет отдавать свою выдачу через API.

---

## Источники

- [TechCrunch — Anthropic appears to be using Brave to power web searches for Claude](https://techcrunch.com/2025/03/21/anthropic-appears-to-be-using-brave-to-power-web-searches-for-its-claude-chatbot/)
- [Brave Search Powers Claude's Web Search (Medium)](https://rayzielrafael.medium.com/brave-search-powers-claudes-web-search-how-this-impacts-ai-and-online-privacy-0dd1885a1077)
- [Claude web search explained — tryprofound.com](https://www.tryprofound.com/blog/what-is-claude-web-search-explained)
- [Enabling and using web search — Claude Help Center](https://support.claude.com/en/articles/10684626-enabling-and-using-web-search)

*Дисклеймер / Disclaimer: material is published for informational and research purposes. [Полный отказ от ответственности / Full disclaimer](https://notes.kazakov.xyz/legal/disclaimer/).*
