# OpenClaw: Полный пример сессии с Tools, Skills, Subagents и Compaction

## Оглавление

1. [Как хранится сессия на диске](#1-как-хранится-сессия-на-диске)
2. [Связь Discord-канала и сессии](#2-связь-discord-канала-и-сессии)
3. [Полный пример: JSONL-файл сессии](#3-полный-пример-jsonl-файл-сессии)
4. [Что происходит при Compaction](#4-что-происходит-при-compaction)
5. [Схема потока данных](#5-схема-потока-данных)

---

## 1. Как хранится сессия на диске

### Файловая структура

```
~/.openclaw/                              # OPENCLAW_STATE_DIR
├── agents/
│   ├── main/                             # agentId = "main" (по умолчанию)
│   │   ├── sessions/
│   │   │   ├── sessions.json             # Индекс всех сессий агента
│   │   │   ├── agent.main.main.jsonl     # DM-сессия (main scope)
│   │   │   ├── agent.main.discord.group.948571234567890.jsonl
│   │   │   │                             # ^ Discord-канал #general (guild 948...)
│   │   │   ├── agent.main.discord.group.948571234567890-topic-1234567890.jsonl
│   │   │   │                             # ^ Тред внутри канала
│   │   │   ├── agent.main.discord.direct.user123456789.jsonl
│   │   │   │                             # ^ Личное сообщение от пользователя Discord
│   │   │   └── agent.main.subagent.abc123.jsonl
│   │   │                                 # ^ Субагент, порождённый основной сессией
│   │   └── skills/                       # Managed skills
│   └── coding-assistant/                 # Другой agentId
│       └── sessions/
│           └── ...
```

### Формат файла: JSONL (JSON Lines)

Каждая строка — отдельный JSON-объект. Права доступа `0o600` (только владелец).

**Типы записей:**

| type | Назначение |
|------|-----------|
| `"session"` | Заголовок (первая строка), версия, sessionId, cwd |
| `"message"` | Сообщение: user / assistant / toolResult |
| `"custom"` | Метаданные: compaction_summary, prompt-error, cache-ttl |

### Индекс сессий: `sessions.json`

```jsonc
{
  "agent:main:discord:group:948571234567890": {
    "sessionId": "agent.main.discord.group.948571234567890",
    "sessionFile": "agent.main.discord.group.948571234567890.jsonl",
    "updatedAt": 1740100000000,
    "compactionCount": 2,
    "inputTokens": 45230,
    "outputTokens": 12400,
    "totalTokens": 57630,
    "cacheRead": 8200,
    "cacheWrite": 3100,
    "modelProvider": "anthropic",
    "model": "claude-sonnet-4-20250514",
    "channel": "discord",
    "lastChannel": "discord"
  }
}
```

---

## 2. Связь Discord-канала и сессии

### Session Key = адрес сессии

OpenClaw строит session key из параметров канала. Это определяет,
какой JSONL-файл использовать для истории.

**Функция:** `buildAgentPeerSessionKey()` в `src/routing/session-key.ts:141-188`

```
Discord-канал #general (guild 948571234567890, channel 1055123456789)
   │
   ▼
buildAgentPeerSessionKey({
  agentId: "main",
  channel: "discord",
  peerKind: "group",          // group = канал/сервер, direct = DM
  peerId: "948571234567890",  // guild ID
})
   │
   ▼
Session Key: "agent:main:discord:group:948571234567890"
   │
   ▼
Session File: ~/.openclaw/agents/main/sessions/
              agent.main.discord.group.948571234567890.jsonl
```

### Разные типы сессий

```
DM с пользователем (dmScope: "per-peer"):
  → "agent:main:direct:user123456789"

DM с пользователем (dmScope: "main"):
  → "agent:main:main"                    # все DM в одной сессии!

Discord-канал:
  → "agent:main:discord:group:948571234567890"

Тред в Discord-канале:
  → "agent:main:discord:group:948571234567890:thread:1234567890"

Субагент:
  → "agent:main:subagent:abc123def"

Cron-задача:
  → "agent:main:cron:daily-report"
```

### Ключевой момент: Discord-канал ≠ история чата Discord

OpenClaw **не хранит** всю историю Discord-канала. Он хранит только:

1. Сообщения, на которые агент **реагировал** (входящие user-сообщения)
2. Ответы агента (assistant-сообщения)
3. Вызовы инструментов и их результаты (toolUse + toolResult)

Когда в Discord-канал приходит новое сообщение и OpenClaw решает ответить,
он загружает **свою сессию** (JSONL-файл), добавляет новое user-сообщение,
отправляет всю историю + system prompt в LLM API, получает ответ,
записывает его в сессию, и отправляет текст в Discord.

```
Discord-канал #general          OpenClaw-сессия (JSONL)
┌──────────────────────┐       ┌────────────────────────────┐
│ Alice: Привет!       │       │                            │
│ Bob: Как дела?       │       │ (не записывается — агент   │
│ Alice: @bot помоги   │──────▶│  не участвовал)            │
│   с деплоем          │       │                            │
│                      │       │ user: "@bot помоги с       │
│                      │       │   деплоем"                 │
│ Bot: Сейчас проверю  │◀──────│ assistant: "Сейчас         │
│   статус...          │       │   проверю статус..."       │
│                      │       │ assistant: [toolUse: exec] │
│                      │       │ toolResult: "deploy ok"    │
│ Bot: Деплой прошёл   │◀──────│ assistant: "Деплой прошёл  │
│   успешно!           │       │   успешно!"                │
│ Charlie: Спасибо     │       │                            │
│ Alice: Класс         │       │ (не записывается)          │
│ Alice: @bot а логи?  │──────▶│ user: "@bot а логи?"       │
│ Bot: Вот логи: ...   │◀──────│ assistant: "Вот логи: ..." │
└──────────────────────┘       └────────────────────────────┘
```

---

## 3. Полный пример: JSONL-файл сессии

Ниже — реалистичный пример содержимого файла
`agent.main.discord.group.948571234567890.jsonl`.

Сценарий: пользователь в Discord-канале просит агента настроить CI/CD,
агент использует tools (exec, read, write, web_search), вызывает skill,
порождает субагент, а затем происходит compaction.

### Строка 1: Заголовок сессии

```json
{"type":"session","version":"1.0","id":"agent.main.discord.group.948571234567890","timestamp":"2026-02-21T10:00:00.000Z","cwd":"/home/openclaw/workspace"}
```

### Строка 2: Первое сообщение пользователя

```json
{"type":"message","message":{"role":"user","content":[{"type":"text","text":"@bot Настрой CI/CD для нашего проекта на GitHub Actions. Репозиторий: github.com/team/webapp. Нужны тесты, линтинг и деплой на staging."}],"timestamp":1740132000000}}
```

### Строка 3: Ответ ассистента — чтение Skill

Агент находит подходящий скилл `github-actions` и читает его SKILL.md.

```json
{"type":"message","message":{"role":"assistant","content":[{"type":"text","text":"Настрою CI/CD пайплайн. Сначала прочитаю инструкции скилла."},{"type":"toolUse","id":"toolu_01A","name":"read","input":{"path":"~/.bun/install/global/node_modules/@openclaw/skills-devops/skills/github-actions/SKILL.md"}}],"timestamp":1740132005000,"api":"anthropic","provider":"anthropic","model":"claude-sonnet-4-20250514","usage":{"input":3200,"output":180,"cacheRead":2800,"cacheWrite":400,"totalTokens":3380},"stopReason":"tool_use"}}
```

### Строка 4: Результат чтения скилла

```json
{"type":"message","message":{"role":"toolResult","toolCallId":"toolu_01A","toolName":"read","content":[{"type":"text","text":"# GitHub Actions CI/CD Skill\n\n## Instructions\n1. Check existing workflow files in .github/workflows/\n2. Use reusable workflow patterns\n3. Always pin action versions to SHA\n4. Use matrix strategy for multi-version testing\n5. Separate jobs: lint → test → build → deploy\n6. Use environment secrets, never hardcode\n7. Add concurrency groups to prevent duplicate runs\n\n## Templates\n...(truncated for brevity)..."}],"isError":false,"timestamp":1740132006000}}
```

### Строка 5: Ассистент — поиск в интернете

```json
{"type":"message","message":{"role":"assistant","content":[{"type":"toolUse","id":"toolu_01B","name":"web_search","input":{"query":"GitHub Actions best practices 2026 Node.js deployment staging"}}],"timestamp":1740132008000,"api":"anthropic","provider":"anthropic","model":"claude-sonnet-4-20250514","usage":{"input":4100,"output":95,"cacheRead":3800,"cacheWrite":300,"totalTokens":4195},"stopReason":"tool_use"}}
```

### Строка 6: Результат web_search

```json
{"type":"message","message":{"role":"toolResult","toolCallId":"toolu_01B","toolName":"web_search","content":[{"type":"text","text":"1. docs.github.com/actions/guides - Official deployment guide (updated Jan 2026)\n2. github.blog/2025-11-actions-v5 - Actions v5 with improved caching\n3. stackoverflow.com/q/98765432 - Node.js 22 + Actions matrix strategy"}],"isError":false,"timestamp":1740132012000,"details":{"resultCount":10,"query":"GitHub Actions best practices 2026 Node.js deployment staging","provider":"brave"}}}
```

> **Примечание:** поле `details` содержит метаданные инструмента.
> При compaction оно **удаляется** (`stripToolResultDetails()`) из соображений
> безопасности — `details` может содержать непроверенные данные из внешних
> источников и не должно попадать в промпт суммаризации.

### Строка 7: Ассистент — проверяет текущую структуру

```json
{"type":"message","message":{"role":"assistant","content":[{"type":"text","text":"Проверю текущую структуру проекта."},{"type":"toolUse","id":"toolu_01C","name":"exec","input":{"command":"ls -la .github/workflows/ 2>/dev/null || echo 'No workflows yet'","timeout":10000}}],"timestamp":1740132014000,"api":"anthropic","provider":"anthropic","model":"claude-sonnet-4-20250514","usage":{"input":5200,"output":120,"cacheRead":4800,"cacheWrite":400,"totalTokens":5320},"stopReason":"tool_use"}}
```

### Строка 8: Результат exec

```json
{"type":"message","message":{"role":"toolResult","toolCallId":"toolu_01C","toolName":"exec","content":[{"type":"text","text":"No workflows yet"}],"isError":false,"timestamp":1740132016000}}
```

### Строка 9: Ассистент — читает package.json

```json
{"type":"message","message":{"role":"assistant","content":[{"type":"toolUse","id":"toolu_01D","name":"read","input":{"path":"/home/openclaw/workspace/package.json"}}],"timestamp":1740132017000,"api":"anthropic","provider":"anthropic","model":"claude-sonnet-4-20250514","usage":{"input":5500,"output":65,"cacheRead":5100,"cacheWrite":400,"totalTokens":5565},"stopReason":"tool_use"}}
```

### Строка 10: Результат чтения package.json

```json
{"type":"message","message":{"role":"toolResult","toolCallId":"toolu_01D","toolName":"read","content":[{"type":"text","text":"{\n  \"name\": \"webapp\",\n  \"version\": \"2.1.0\",\n  \"scripts\": {\n    \"test\": \"vitest run\",\n    \"test:e2e\": \"playwright test\",\n    \"lint\": \"eslint src/\",\n    \"build\": \"vite build\",\n    \"typecheck\": \"tsc --noEmit\"\n  },\n  \"engines\": { \"node\": \">=20\" }\n}"}],"isError":false,"timestamp":1740132018000}}
```

### Строка 11: Ассистент — создаёт workflow файл

```json
{"type":"message","message":{"role":"assistant","content":[{"type":"text","text":"Создаю CI/CD пайплайн."},{"type":"toolUse","id":"toolu_01E","name":"exec","input":{"command":"mkdir -p .github/workflows"}},{"type":"toolUse","id":"toolu_01F","name":"write","input":{"path":"/home/openclaw/workspace/.github/workflows/ci.yml","content":"name: CI/CD Pipeline\n\non:\n  push:\n    branches: [main, develop]\n  pull_request:\n    branches: [main]\n\nconcurrency:\n  group: ${{ github.workflow }}-${{ github.ref }}\n  cancel-in-progress: true\n\njobs:\n  lint:\n    runs-on: ubuntu-latest\n    steps:\n      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2\n      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0\n        with:\n          node-version: 22\n          cache: npm\n      - run: npm ci\n      - run: npm run lint\n      - run: npm run typecheck\n\n  test:\n    needs: lint\n    strategy:\n      matrix:\n        node-version: [20, 22]\n    runs-on: ubuntu-latest\n    steps:\n      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683\n      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af\n        with:\n          node-version: ${{ matrix.node-version }}\n          cache: npm\n      - run: npm ci\n      - run: npm run test\n\n  e2e:\n    needs: test\n    runs-on: ubuntu-latest\n    steps:\n      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683\n      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af\n        with:\n          node-version: 22\n          cache: npm\n      - run: npm ci\n      - run: npx playwright install --with-deps chromium\n      - run: npm run test:e2e\n\n  deploy-staging:\n    needs: [test, e2e]\n    if: github.ref == 'refs/heads/main' && github.event_name == 'push'\n    runs-on: ubuntu-latest\n    environment: staging\n    steps:\n      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683\n      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af\n        with:\n          node-version: 22\n          cache: npm\n      - run: npm ci\n      - run: npm run build\n      - name: Deploy to staging\n        env:\n          DEPLOY_TOKEN: ${{ secrets.STAGING_DEPLOY_TOKEN }}\n          DEPLOY_URL: ${{ vars.STAGING_URL }}\n        run: |\n          echo \"Deploying to $DEPLOY_URL\"\n          # Add actual deploy command here\n"}}],"timestamp":1740132020000,"api":"anthropic","provider":"anthropic","model":"claude-sonnet-4-20250514","usage":{"input":6200,"output":850,"cacheRead":5600,"cacheWrite":600,"totalTokens":7050},"stopReason":"tool_use"}}
```

> **Примечание:** ассистент вызвал два инструмента параллельно (exec + write)
> в одном сообщении. Оба `toolUse` блока — в одном массиве `content`.
> Каждый получит отдельную запись `toolResult`.

### Строка 12-13: Результаты параллельных tool calls

```json
{"type":"message","message":{"role":"toolResult","toolCallId":"toolu_01E","toolName":"exec","content":[{"type":"text","text":""}],"isError":false,"timestamp":1740132021000}}
```

```json
{"type":"message","message":{"role":"toolResult","toolCallId":"toolu_01F","toolName":"write","content":[{"type":"text","text":"File written: /home/openclaw/workspace/.github/workflows/ci.yml (1847 bytes)"}],"isError":false,"timestamp":1740132022000}}
```

### Строка 14: Ассистент — порождает субагент для E2E-конфига

Агент решает, что настройка Playwright-конфига — отдельная задача,
и порождает субагент для этого.

```json
{"type":"message","message":{"role":"assistant","content":[{"type":"text","text":"CI/CD пайплайн создан. Запущу субагент для настройки Playwright-конфигурации."},{"type":"toolUse","id":"toolu_01G","name":"sessions_spawn","input":{"agentId":"main","message":"Настрой playwright.config.ts для проекта в /home/openclaw/workspace. Нужна конфигурация для CI: headless Chromium, retries=2, reporter=html. Проверь существующий конфиг и обнови или создай новый.","sessionKey":"subagent:playwright-setup"}}],"timestamp":1740132025000,"api":"anthropic","provider":"anthropic","model":"claude-sonnet-4-20250514","usage":{"input":8100,"output":210,"cacheRead":7500,"cacheWrite":600,"totalTokens":8310},"stopReason":"tool_use"}}
```

### Строка 15: Результат sessions_spawn

```json
{"type":"message","message":{"role":"toolResult","toolCallId":"toolu_01G","toolName":"sessions_spawn","content":[{"type":"text","text":"Spawned sub-agent session: agent:main:subagent:playwright-setup\nThe sub-agent will announce completion automatically."}],"isError":false,"timestamp":1740132027000}}
```

> **Что происходит на диске:**
>
> Субагент создаёт **отдельный** JSONL-файл:
> `~/.openclaw/agents/main/sessions/agent.main.subagent.playwright-setup.jsonl`
>
> Он получает **минимальный** system prompt (`promptMode: "minimal"`) — без
> скиллов, памяти, heartbeats. Его инструменты ограничены профилем субагента
> (`resolveSubagentToolPolicy()`).

### Строка 16: Ассистент — финальный ответ пользователю

```json
{"type":"message","message":{"role":"assistant","content":[{"type":"text","text":"Готово! Вот что я настроил:\n\n**CI/CD пайплайн** (`.github/workflows/ci.yml`):\n- **Lint** — ESLint + TypeScript typecheck\n- **Test** — Vitest на Node.js 20 и 22 (matrix strategy)\n- **E2E** — Playwright с headless Chromium\n- **Deploy** — автоматический деплой на staging при пуше в main\n\nОсобенности:\n- Actions запинены на SHA (безопасность)\n- Concurrency group предотвращает дубли\n- npm cache для ускорения\n- E2E запускается только после unit-тестов\n\nСубагент настраивает `playwright.config.ts` — сообщит когда закончит.\n\nОсталось добавить секреты в GitHub:\n1. `STAGING_DEPLOY_TOKEN` — токен деплоя\n2. Переменную `STAGING_URL` — URL staging-окружения\n\nСделать это?"}],"timestamp":1740132030000,"api":"anthropic","provider":"anthropic","model":"claude-sonnet-4-20250514","usage":{"input":9200,"output":320,"cacheRead":8500,"cacheWrite":700,"totalTokens":9520},"stopReason":"stop"}}
```

### Строка 17: Новое сообщение пользователя (через 3 часа)

```json
{"type":"message","message":{"role":"user","content":[{"type":"text","text":"@bot Добавь ещё Dependabot для автообновления зависимостей"}],"timestamp":1740142800000}}
```

### Строка 18: Ассистент создаёт dependabot.yml

```json
{"type":"message","message":{"role":"assistant","content":[{"type":"toolUse","id":"toolu_01H","name":"write","input":{"path":"/home/openclaw/workspace/.github/dependabot.yml","content":"version: 2\nupdates:\n  - package-ecosystem: npm\n    directory: /\n    schedule:\n      interval: weekly\n      day: monday\n      time: \"09:00\"\n      timezone: Europe/Moscow\n    open-pull-requests-limit: 10\n    reviewers:\n      - team/frontend\n    labels:\n      - dependencies\n      - automated\n    groups:\n      dev-dependencies:\n        dependency-type: development\n        update-types:\n          - minor\n          - patch\n      production-dependencies:\n        dependency-type: production\n\n  - package-ecosystem: github-actions\n    directory: /\n    schedule:\n      interval: weekly\n    labels:\n      - ci\n      - automated\n"}}],"timestamp":1740142805000,"api":"anthropic","provider":"anthropic","model":"claude-sonnet-4-20250514","usage":{"input":10500,"output":350,"cacheRead":9800,"cacheWrite":700,"totalTokens":10850},"stopReason":"tool_use"}}
```

### Строка 19: Результат write

```json
{"type":"message","message":{"role":"toolResult","toolCallId":"toolu_01H","toolName":"write","content":[{"type":"text","text":"File written: /home/openclaw/workspace/.github/dependabot.yml (612 bytes)"}],"isError":false,"timestamp":1740142806000}}
```

### Строка 20: Финальный ответ

```json
{"type":"message","message":{"role":"assistant","content":[{"type":"text","text":"Добавил `.github/dependabot.yml`:\n\n- **npm**: еженедельно по понедельникам, группировка dev/production\n- **GitHub Actions**: еженедельно, отдельно\n- Автоматические PR с лейблами `dependencies` / `ci`\n- Лимит: 10 открытых PR одновременно\n- Ревьюер: team/frontend"}],"timestamp":1740142810000,"api":"anthropic","provider":"anthropic","model":"claude-sonnet-4-20250514","usage":{"input":11200,"output":150,"cacheRead":10500,"cacheWrite":700,"totalTokens":11350},"stopReason":"stop"}}
```

---

*...Далее проходит ещё 50+ обменов сообщениями за несколько дней...*
*Агент помогает с Docker, Terraform, секретами, код-ревью...*
*Контекстное окно приближается к лимиту...*

---

## 4. Что происходит при Compaction

### Триггер

Compaction срабатывает когда **оценочное количество токенов в истории**
приближается к контекстному окну модели.

**Код:** `src/agents/pi-extensions/compaction-safeguard.ts`,
`src/agents/compaction.ts`

```
Контекстное окно модели: 200,000 токенов
System prompt:             ~8,000 токенов
Текущая история:         ~185,000 токенов   ← ТРИГГЕР
Свободно:                  ~7,000 токенов   ← недостаточно для ответа
```

Также можно вызвать вручную командой `/compact` в чате.

### Алгоритм (src/agents/compaction.ts)

```
1. estimateMessagesTokens(messages)
   └── stripToolResultDetails(messages)   ← УДАЛЯЕТ details из toolResult
   └── estimateTokens(msg) для каждого    ← ~4 символа = 1 токен

2. computeAdaptiveChunkRatio(messages, contextWindow)
   └── BASE_CHUNK_RATIO = 0.4  (40% контекста на чанк)
   └── Если средний размер сообщения > 10% контекста → уменьшает ratio
   └── MIN_CHUNK_RATIO = 0.15

3. splitMessagesByTokenShare(oldMessages, parts=2)
   └── Делит на чанки по ~равному количеству токенов

4. summarizeChunks(chunk, model, apiKey, ...)
   └── Для каждого чанка:
       └── generateSummary(chunk, model, reserveTokens, apiKey)
           └── Отправляет чанк в LLM с инструкцией "summarize"
           └── Retry: 3 попытки, 500ms-5000ms backoff
   └── Если чанк слишком большой:
       └── summarizeWithFallback() → пропускает oversized сообщения
       └── "[Large assistant (~45K tokens) omitted from summary]"

5. Результат: краткое текстовое резюме всей старой истории
```

### Состояние JSONL-файла ПОСЛЕ Compaction

Compaction **не переписывает** старые строки в файле. Вместо этого:

1. Старые сообщения **заменяются в памяти** (через `agent.replaceMessages()`)
2. В JSONL **дописывается** custom-запись с резюме
3. При следующей загрузке сессии — `SessionManager` загрузит последнее
   актуальное состояние

```
ФАЙЛ ПОСЛЕ COMPACTION (добавляются новые строки):
```

### Строка 71: Custom-запись о compaction

```json
{"type":"custom","customType":"compaction_summary","data":{"previousMessageCount":68,"resultingMessageCount":4,"compactedAt":"2026-02-23T14:30:00.000Z","model":"claude-sonnet-4-20250514","provider":"anthropic","tokensBeforeCompaction":182400,"tokensAfterCompaction":4200}}
```

### Строка 72: Резюме-сообщение (role: user, синтетическое)

После compaction в историю вставляется **синтетическое user-сообщение**
с резюме предыдущего контекста:

```json
{"type":"message","message":{"role":"user","content":[{"type":"text","text":"[Previous conversation summary]\n\nThe user requested CI/CD setup for github.com/team/webapp using GitHub Actions. The following was completed:\n\n1. Created `.github/workflows/ci.yml` with pipeline: lint → test (matrix Node 20/22) → e2e (Playwright) → deploy-staging (on push to main). Actions pinned to SHA, concurrency groups enabled, npm caching configured.\n\n2. Spawned sub-agent for Playwright config setup (completed: playwright.config.ts with headless Chromium, retries=2, html reporter).\n\n3. Added `.github/dependabot.yml` for weekly npm + GitHub Actions dependency updates with grouping and auto-labeling.\n\n4. Created Dockerfile (multi-stage: deps → build → runtime) and docker-compose.yml for local development with PostgreSQL + Redis.\n\n5. Set up Terraform configuration for staging: AWS ECS Fargate with ALB, RDS PostgreSQL, ElastiCache Redis. State in S3 with DynamoDB locking.\n\n6. Configured GitHub repository secrets via `gh secret set`: STAGING_DEPLOY_TOKEN, AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, DATABASE_URL.\n\n7. Added branch protection rules for main: require PR reviews, status checks (lint, test, e2e), linear history.\n\nOpen items:\n- Production deployment pipeline not yet created (user said 'later')\n- Monitoring/alerting setup pending\n- User asked about adding Sentry but hasn't decided yet"}],"timestamp":1740318600000}}
```

### Строка 73: Ответ ассистента (подтверждение контекста)

```json
{"type":"message","message":{"role":"assistant","content":[{"type":"text","text":"Understood. I have the context of our previous work on the CI/CD setup, infrastructure, and configuration for the webapp project. What would you like to work on next?"}],"timestamp":1740318602000,"api":"anthropic","provider":"anthropic","model":"claude-sonnet-4-20250514","usage":{"input":4200,"output":45,"cacheRead":0,"cacheWrite":4200,"totalTokens":4245},"stopReason":"stop"}}
```

### Строка 74-...: Новые сообщения продолжают дописываться

```json
{"type":"message","message":{"role":"user","content":[{"type":"text","text":"@bot Теперь давай настроим Sentry для мониторинга ошибок"}],"timestamp":1740318700000}}
```

### Визуальная схема compaction

```
ДО COMPACTION (в памяти):                    ПОСЛЕ COMPACTION (в памяти):
┌──────────────────────────┐                 ┌──────────────────────────┐
│ msg 1: user "настрой CI" │                 │ msg 1: user "[Summary]   │
│ msg 2: asst [read skill] │                 │   CI/CD создан,          │
│ msg 3: toolResult        │                 │   Docker настроен,       │
│ msg 4: asst [web_search] │                 │   Terraform готов..."    │
│ msg 5: toolResult        │                 │                          │
│ msg 6: asst [exec ls]    │                 │ msg 2: asst "Understood. │
│ msg 7: toolResult        │                 │   What's next?"          │
│ ...                      │                 │                          │
│ msg 35: asst [write tf]  │  ══════════▶    │ msg 3: user "настрой     │
│ msg 36: toolResult       │  compaction     │   Sentry"                │
│ ...                      │                 │                          │
│ msg 67: user "а Sentry?" │                 │ (свежие сообщения        │
│ msg 68: asst "давайте"   │                 │  сохраняются как есть)   │
└──────────────────────────┘                 └──────────────────────────┘
  ~182,400 токенов                             ~4,200 токенов
  68 сообщений                                 3 сообщения
```

### На диске (JSONL-файл):

```
JSONL-файл НЕ усекается. Старые строки остаются,
но SessionManager знает, что после compaction_summary
нужно загружать только новые сообщения.

Строка 1:  {"type":"session",...}          ← заголовок
Строка 2:  {"type":"message",...}          ← msg 1 (старая)
Строка 3:  {"type":"message",...}          ← msg 2 (старая)
...
Строка 70: {"type":"message",...}          ← msg 68 (старая)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Строка 71: {"type":"custom","customType":"compaction_summary",...}  ← МАРКЕР
Строка 72: {"type":"message",...}          ← резюме (синтетическое)
Строка 73: {"type":"message",...}          ← подтверждение
Строка 74: {"type":"message",...}          ← новое сообщение
...
```

---

## 5. Схема потока данных

### Полный путь: Discord-сообщение → LLM API → ответ в Discord

```
┌─────────────────────────────────────────────────────────────────┐
│                        DISCORD                                  │
│  #general                                                       │
│  ┌─────────────────────────────────────────┐                    │
│  │ User: @bot Настрой CI/CD для проекта    │                    │
│  └──────────────────┬──────────────────────┘                    │
└─────────────────────┼───────────────────────────────────────────┘
                      │ Discord Gateway → OpenClaw Discord adapter
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                   OPENCLAW GATEWAY                               │
│                                                                  │
│  1. Определить session key:                                      │
│     buildAgentPeerSessionKey({                                   │
│       agentId: "main",                                           │
│       channel: "discord",                                        │
│       peerKind: "group",                                         │
│       peerId: "948571234567890"                                  │
│     }) → "agent:main:discord:group:948571234567890"              │
│                                                                  │
│  2. Найти/создать session file:                                  │
│     → ~/.openclaw/agents/main/sessions/                          │
│       agent.main.discord.group.948571234567890.jsonl              │
│                                                                  │
│  3. Enqueue в command queue (одна сессия = один lane)            │
└──────────────────┬──────────────────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│              runEmbeddedAttempt()                                 │
│              (src/agents/pi-embedded-runner/run/attempt.ts)       │
│                                                                  │
│  ┌────────────── ПОДГОТОВКА ──────────────┐                      │
│  │                                         │                     │
│  │  4. loadWorkspaceSkillEntries()         │                     │
│  │     → SkillEntry[] (github-actions,     │                     │
│  │       docker, terraform, ...)           │                     │
│  │                                         │                     │
│  │  5. resolveSkillsPromptForRun()         │                     │
│  │     → skills prompt string              │                     │
│  │                                         │                     │
│  │  6. resolveBootstrapContextForRun()     │                     │
│  │     → SOUL.md, CLAUDE.md, etc.          │                     │
│  │                                         │                     │
│  │  7. createOpenClawCodingTools()         │                     │
│  │     → [read, write, edit, exec,         │                     │
│  │        process, web_search, web_fetch,  │                     │
│  │        message, sessions_spawn,         │                     │
│  │        sessions_list, subagents, ...]   │                     │
│  │     → applyToolPolicyPipeline()         │                     │
│  │     → applyOwnerOnlyToolPolicy()        │                     │
│  │                                         │                     │
│  │  8. buildEmbeddedSystemPrompt()         │                     │
│  │     → buildAgentSystemPrompt({          │                     │
│  │         toolNames, skillsPrompt,        │                     │
│  │         contextFiles, runtimeInfo,      │                     │
│  │         promptMode: "full", ...         │                     │
│  │       })                                │                     │
│  │     → ~8000 токенов system prompt       │                     │
│  └─────────────────────────────────────────┘                     │
│                                                                  │
│  ┌────────────── СЕССИЯ ──────────────────┐                      │
│  │                                         │                     │
│  │  9. SessionManager.open(sessionFile)    │                     │
│  │     → Загрузка JSONL-файла              │                     │
│  │     → Парсинг всех строк               │                     │
│  │     → Если есть compaction_summary:     │                     │
│  │       загрузить только сообщения        │                     │
│  │       после маркера                     │                     │
│  │                                         │                     │
│  │  10. createAgentSession({               │                     │
│  │        tools: builtInTools,             │                     │
│  │        customTools: allCustomTools,     │                     │
│  │        sessionManager, ...              │                     │
│  │      })                                 │                     │
│  │                                         │                     │
│  │  11. applySystemPromptOverrideToSession │                     │
│  │      (session, systemPromptText)        │                     │
│  │                                         │                     │
│  │  12. sanitizeSessionHistory()           │                     │
│  │      → валидация turns                  │                     │
│  │      → limitHistoryTurns()              │                     │
│  │      → repairToolUseResultPairing()     │                     │
│  │      → agent.replaceMessages(limited)   │                     │
│  └─────────────────────────────────────────┘                     │
│                                                                  │
│  ┌────────────── ВЫЗОВ LLM ───────────────┐                      │
│  │                                         │                     │
│  │  13. activeSession.steer(userMessage)   │                     │
│  │      │                                  │                     │
│  │      ▼                                  │                     │
│  │  agent.streamFn(model, context, opts)   │                     │
│  │  │                                      │                     │
│  │  │  context = {                         │                     │
│  │  │    messages: [                       │                     │
│  │  │      { role: "system",               │                     │
│  │  │        content: systemPrompt },      │                     │
│  │  │      ...session.messages,            │                     │
│  │  │      { role: "user",                 │                     │
│  │  │        content: "Настрой CI/CD..." } │                     │
│  │  │    ],                                │                     │
│  │  │    tools: [                          │                     │
│  │  │      { name: "read",                 │                     │
│  │  │        parameters: {...} },          │                     │
│  │  │      { name: "exec",                 │                     │
│  │  │        parameters: {...} },          │                     │
│  │  │      ...                             │                     │
│  │  │    ]                                 │                     │
│  │  │  }                                   │                     │
│  │  │                                      │                     │
│  │  ▼                                      │                     │
│  │  streamSimple() → Anthropic API         │                     │
│  │  (обёрнут в cacheTrace +                │                     │
│  │   anthropicPayloadLogger +              │                     │
│  │   applyExtraParams)                     │                     │
│  │                                         │                     │
│  │  14. ← Streaming response               │                     │
│  │      ← tool_use: read SKILL.md          │                     │
│  │      → execute tool → toolResult        │                     │
│  │      ← tool_use: exec "mkdir -p"        │                     │
│  │      → execute tool → toolResult        │                     │
│  │      ← tool_use: write ci.yml           │                     │
│  │      → execute tool → toolResult        │                     │
│  │      ← text: "Готово! Вот что..."       │                     │
│  │      ← stop_reason: "stop"              │                     │
│  │                                         │                     │
│  │  15. Каждое сообщение записывается      │                     │
│  │      в JSONL через SessionManager       │                     │
│  └─────────────────────────────────────────┘                     │
│                                                                  │
│  ┌────────── COMPACTION (если нужно) ─────┐                      │
│  │                                         │                     │
│  │  16. Если context overflow:             │                     │
│  │      compactEmbeddedPiSession()         │                     │
│  │      → splitMessagesByTokenShare()      │                     │
│  │      → summarizeChunks() via LLM       │                     │
│  │      → agent.replaceMessages(summary)   │                     │
│  │      → Дописать compaction_summary      │                     │
│  │        в JSONL                          │                     │
│  │      → Обновить sessions.json:          │                     │
│  │        compactionCount++                │                     │
│  └─────────────────────────────────────────┘                     │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                        DISCORD                                   │
│  #general                                                        │
│  ┌─────────────────────────────────────────┐                     │
│  │ Bot: Готово! Вот что я настроил:        │                     │
│  │ **CI/CD пайплайн** ...                  │                     │
│  └─────────────────────────────────────────┘                     │
└─────────────────────────────────────────────────────────────────┘
```

### Жизненный цикл субагента

```
Основная сессия                          Субагент
(agent:main:discord:group:948...)        (agent:main:subagent:playwright-setup)
                                          │
sessions_spawn("playwright-setup") ──────▶│ Создаётся JSONL-файл
                                          │ agent.main.subagent.playwright-setup.jsonl
                                          │
                                          │ System prompt: promptMode="minimal"
                                          │ Tools: ограничены resolveSubagentToolPolicy()
                                          │   (обычно: read, write, edit, exec, process)
                                          │
                                          │ Работает автономно:
                                          │ → read playwright.config.ts
                                          │ → write playwright.config.ts (обновлённый)
                                          │ → exec "npx playwright test --list"
                                          │
                                          │ Завершение:
                                          │ → Автоматически отправляет результат
◀─────────────────────────────────────────│   в родительскую сессию
                                          │
"Sub-agent playwright-setup completed:    │
 Updated playwright.config.ts with..."    │
```

---

## Ключевые выводы

1. **Одна сессия = один JSONL-файл.** Discord-канал, DM, субагент — каждый
   получает свой файл. Session key определяет маппинг.

2. **OpenClaw НЕ хранит весь чат Discord.** Только сообщения, адресованные
   агенту, и его ответы. Контекст остальных сообщений теряется.

3. **Compaction не удаляет данные с диска.** Старые строки остаются в JSONL,
   но в память загружаются только сообщения после маркера `compaction_summary`.

4. **Tool results хранят `details`**, но при compaction и суммаризации
   `details` удаляется (`stripToolResultDetails()`) — это защита от инъекции
   непроверенных данных из внешних источников в промпт LLM.

5. **Субагенты полностью изолированы** — отдельный файл, урезанный промпт,
   ограниченные инструменты. Результат доставляется автоматически.

6. **sessions.json** — индекс с метаданными (токены, провайдер, счётчик
   compaction). Используется для `/status`, лимитов истории, UI.
