---
{"title":"Собираем локального AI агента на LibreChat, Langflow и MCP","date":"2026-04-08","tags":["#ai-agents","#mcp","#docker","#langflow","#librechat","#local-llm","#infrastructure"],"stream":"manual","capture_status":"processed","origin":"clipping","dg-publish":true,"dg-permalink":"local-ai-agent-librechat-langflow-mcp","permalink":"/local-ai-agent-librechat-langflow-mcp/","dgPassFrontmatter":true}
---


# Собираем локального AI агента на LibreChat, Langflow и MCP

> **Источник:** [Собираем локального AI агента на LibreChat, Langflow и MCP](https://tproger.ru/articles/sobiraem-lokalnogo-ai-agenta-na-librechat--langflow-i-mcp) — Николай Луняка, Tproger

Представьте: вы пишете агенту «Проверь отчеты в папке и выпиши главное». Агент сам открывает файлы, анализирует их и выдает результат. Никакого копипаста в чат, никакого пересказа содержимого. Это возможно с комбинацией LibreChat, Langflow и MCP.

## Архитектура: три компонента

Локальная агентная система состоит из трех основных частей:

1. **LibreChat** — open-source UI для работы с LLM (поддерживает разные провайдеры моделей и подключение MCP-серверов)
2. **Langflow** — low-code платформа с визуальным редактором для сборки flow как последовательности компонентов/шагов
3. **MCP-сервер** — сервис, который публикует tools/ресурсы по стандартному протоколу (HTTP/SSE/stdio)

Все примеры рассчитаны на локальный запуск в изолированной Docker сети. Для публичного деплоя потребуется дополнительная настройка безопасности (TLS, reverse proxy и т.д.).

## Этап 1: LibreChat + локальная LLM

LibreChat — это open-source веб UI чат для работы с различными LLM в одном интерфейсе. Поддерживает OpenAI-совместимые API и работает с агентами через протокол MCP.

На этом уровне запускаем три контейнера:
- **MongoDB** — хранение истории диалогов и настроек
- **Ollama** — запуск локальных LLM
- **LibreChat** — веб UI

### Структура проекта

```
ai-agent/
├── docker-compose.yml
├── librechat.yaml
├── .env
└── documents/
```

### docker-compose.yml

```yaml
services:
  mongo:
    image: mongo:7.0.5
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}
    volumes:
      - ./mongo_data:/data/db
    networks:
      - ai-network
    healthcheck:
      test: ["CMD-SHELL", "mongosh \"mongodb://${MONGO_INITDB_ROOT_USERNAME}:${MONGO_INITDB_ROOT_PASSWORD}@localhost:27017/admin\" --quiet --eval \"db.adminCommand('ping').ok\" | grep 1 >/dev/null"]
      interval: 10s
      timeout: 5s
      retries: 5

  ollama:
    image: ollama/ollama:latest
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ./ollama_data:/root/.ollama
    networks:
      - ai-network
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

  app:
    image: ghcr.io/danny-avila/librechat:latest
    restart: unless-stopped
    ports:
      - "3080:3080"
    depends_on:
      mongo:
        condition: service_healthy
      ollama:
        condition: service_started
    environment:
      MONGO_URI: mongodb://${MONGO_USER}:${MONGO_PASSWORD}@mongo:27017/?authSource=admin
      JWT_SECRET: ${JWT_SECRET}
      JWT_REFRESH_SECRET: ${JWT_REFRESH_SECRET}
      ALLOW_REGISTRATION: "true"
    volumes:
      - ./librechat.yaml:/app/librechat.yaml
    networks:
      - ai-network

networks:
  ai-network:
    driver: bridge
```

### librechat.yaml

```yaml
version: 1.3.3
cache: true

endpoints:
  custom:
    - name: "Ollama (Local)"
      apiKey: "ollama"
      baseURL: "http://ollama:11434/v1/"
      models:
        default:
          - "llama3.1"
        fetch: true
      titleConvo: true
      summarize: false
```

### .env

```
MONGO_USER=admin
MONGO_PASSWORD=CHANGE_ME_STRONG_PASSWORD_HERE

JWT_SECRET=CHANGE_ME_RANDOM_STRING_1
JWT_REFRESH_SECRET=CHANGE_ME_RANDOM_STRING_2
```

### Запуск

```bash
docker compose up -d
docker compose ps

# Скачиваем модель
docker compose exec ollama ollama pull llama3.1

# Проверяем
docker compose exec ollama ollama list
```

Открываем браузер: `http://localhost:3080/`. Создаем аккаунт, выбираем endpoint Ollama и модель llama3.1.

## Этап 2: Подключаем MCP Server (filesystem)

На этом шаге LibreChat подключается к MCP-серверу, получает список доступных tools, и модель решает вызывать инструмент или отвечать текстом.

**MCP** ([Model Context Protocol](https://modelcontextprotocol.io/)) — открытый протокол, который описывает, как LLM подключаться к инструментам (файлы, БД, API) через единый стандарт.

Два основных вида транспорта:
1. **stdio** — локальный транспорт (хост запускает MCP сервер как процесс)
2. **Streamable HTTP** — удаленный транспорт (MCP сервер работает как отдельный сервис)

Используем [supergateway](https://github.com/supercorp-ai/supergateway) как мост между stdio и HTTP.

### Обновляем docker-compose.yml

Добавляем сервис mcp-filesystem:

```yaml
mcp-filesystem:
  image: node:20-slim
  restart: unless-stopped
  volumes:
    - ./documents:/data:ro  # :rw если нужна запись
  networks:
    - ai-network
  command:
    - sh
    - -lc
    - |
        set -e
        npx --yes supergateway@3.4.3 \
          --port 8000 \
          --outputTransport streamableHttp \
          --stateful \
          --sessionTimeout 600000 \
          --streamableHttpPath /mcp \
          --healthEndpoint /healthz \
          --stdio "npx --yes @modelcontextprotocol/server-filesystem@2026.1.14 /data"
  healthcheck:
    test:
      [
        "CMD-SHELL",
        "node -e \"require('http').get('http://localhost:8000/healthz', r=>process.exit(r.statusCode===200?0:1)).on('error',()=>process.exit(1))\""
      ]
    interval: 10s
    timeout: 5s
    retries: 3
```

### Обновляем librechat.yaml

```yaml
mcpSettings:
  allowedDomains:
    - "mcp-filesystem"
mcpServers:
  filesystem:
    type: "streamable-http"
    url: "http://mcp-filesystem:8000/mcp"
```

### Запуск

```bash
docker compose up -d mcp-filesystem app
docker compose ps
```

В LibreChat: Настройки MCP → включаем filesystem.

## Этап 3: Подключаем Langflow

Langflow — платформа для сборки flow через визуальный интерфейс. Каждый flow можно запускать через REST API. Есть нативная поддержка MCP Tools.

Добавив Langflow, расчеты и проверки уходят в Python-компоненты, агент работает стабильнее и точнее.

### Обновляем docker-compose.yml

```yaml
langflow:
  image: langflowai/langflow:latest
  restart: unless-stopped
  ports:
    - "7860:7860"
  environment:
    LANGFLOW_AUTO_LOGIN: "True"
    LANGFLOW_CONFIG_DIR: /app/langflow
    LANGFLOW_DATABASE_URL: sqlite:////app/langflow/langflow.db
    LANGFLOW_COMPONENTS_PATH: /app/custom_components
  volumes:
    - ./langflow_data:/app/langflow
    - ./custom_components:/app/custom_components
    - ./documents:/data:ro
  networks:
    - ai-network
  healthcheck:
    test: ["CMD", "python", "-c", "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://localhost:7860/health_check').status==200 else 1)"]
    interval: 10s
    timeout: 5s
    retries: 5
```

### Запуск

```bash
docker compose up -d
docker compose ps langflow
```

Открываем: `http://localhost:7860/`

### Создаем Flow

1. New Flow
2. Добавляем компоненты: Chat Input → Agent → Custom Component → MCP Tools → Chat Output
3. Настраиваем Agent:
   - Ollama API URL: `http://ollama:11434/`
   - Model Name: `llama3.1`
   - Temperature: `0.3`

4. Подключаем MCP Server:
   - Type: Streamable HTTP/SSE
   - URL: `http://mcp-filesystem:8000/mcp`

5. Тестируем в Playground

## Этап 4: Подключаем Langflow к LibreChat как MCP Server

Настраиваем Langflow в режиме MCP-сервера, чтобы добавить его как инструмент LibreChat.

### Включаем MCP Server в Langflow

1. Langflow → MCP Server вкладка
2. Выбираем flow и транспорт (Streamable или SSE)
3. Копируем URL (замените localhost на langflow для Docker):
   - было: `http://localhost:7860/api/v1/mcp/project/<id>/streamable`
   - стало: `http://langflow:7860/api/v1/mcp/project/<id>/streamable`

### Обновляем docker-compose.yml

```yaml
app:
  # ... остальное
  depends_on:
    mongo:
      condition: service_healthy
    ollama:
      condition: service_started
    langflow:
      condition: service_healthy
  environment:
    # ... остальное
    LANGFLOW_API_KEY: ${LANGFLOW_API_KEY}
```

### Обновляем librechat.yaml

```yaml
mcpSettings:
  allowedDomains:
    - "mcp-filesystem"
    - "langflow"

mcpServers:
  filesystem:
    type: "streamable-http"
    url: "http://mcp-filesystem:8000/mcp"

  langflow:
    type: "streamable-http"
    url: "http://langflow:7860/api/v1/mcp/project/<PROJECT_ID>/streamable"
    headers:
      x-api-key: "${LANGFLOW_API_KEY}"
```

### Запуск

```bash
docker compose up -d app
docker compose logs -f app
```

В LibreChat: Настройки → MCP Servers → Add MCP server → вставляем URL Langflow.

## Масштабирование

Главная идея: **инструмент = MCP-сервер**.

Хотите добавить новый источник данных — добавляете/пишете MCP-сервер и подключаете его либо в LibreChat, либо в Langflow.

Три режима использования:

1. **Полностью локально** — приватные документы, отчеты по Excel, поиск по заметкам, разбор логов
2. **Закрытая сеть** — корпоративный контур, Jira/Confluence, анализ инцидентов
3. **Онлайн/гибрид** — почта/календарь, внешние API, задачи/трекеры

Но заранее настройте безопасность: TLS/reverse proxy, аутентификацию, ограничение прав инструментов.

---

## Инструкция для агента

```
Этот материал используется как основа для развертывания локальной агентной системы.

ОБЯЗАТЕЛЬНЫЕ ШАГИ:

1. АДАПТАЦИЯ КОНФИГОВ
   - Замените пути на актуальные для целевой системы
   - Обновите переменные окружения (.env)
   - Проверьте версии образов (особенно Ollama, LibreChat, Langflow)
   - Убедитесь, что порты (3080, 7860, 11434, 8000) свободны

2. ПОЭТАПНОЕ ТЕСТИРОВАНИЕ
   - Этап 1: запустить только MongoDB + Ollama + LibreChat
   - Проверить: http://localhost:3080 доступен, модель скачана
   - Этап 2: добавить mcp-filesystem, проверить подключение
   - Этап 3: добавить Langflow, создать простой flow
   - Этап 4: интегрировать Langflow в LibreChat
   - НЕ переходить на следующий этап, пока предыдущий не работает

3. PRODUCTION ТРЕБОВАНИЯ
   - Добавить TLS (самоподписанный сертификат минимум)
   - Настроить reverse proxy (nginx/traefik)
   - Включить логирование для всех сервисов
   - Настроить мониторинг (Prometheus/Grafana)
   - Ограничить доступ по IP/VPN
   - Использовать :ro для монтирования документов (read-only)

4. ДОКУМЕНТИРОВАНИЕ
   - Каждый custom компонент в Langflow должен иметь README
   - MCP серверы должны быть задокументированы (какие tools, какие ограничения)
   - Сохранять конфиги в git (без .env с секретами)
   - Вести changelog изменений

5. НАЧАЛО РАБОТЫ
   - Начните с LibreChat + Ollama (самое простое)
   - Добавьте MCP filesystem только когда базовый чат работает
   - Langflow добавляйте только если нужна сложная логика
   - Не усложняйте архитектуру без необходимости

КРИТИЧНЫЕ ПРОВЕРКИ:
- docker compose ps показывает все контейнеры в статусе Up
- docker compose logs не содержит ошибок
- Healthchecks проходят успешно
- MCP сервер отвечает на POST http://mcp-filesystem:8000/mcp
- Модель в Ollama скачана и доступна
```
