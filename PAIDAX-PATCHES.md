# OpenClaw Paidax Patches

Патчтар OpenClaw fork-ына жасалған өзгерістер. Docker image қайта build кезінде осы файлды тексеру.

## Base Version

OpenClaw commit: `9231d7d` (chore: bump version to 2026.2.21)

## Patches

### 1. grammY sequentialize deadlock fix

**Файл**: `src/telegram/bot.ts`
**Мәселе**: `callback_query` handlers deadlock — `sequentialize` middleware бір lane-да кезекте тұрады, callback пен негізгі хабар бір-бірін блоктайды.
**Шешім**: Callback updates үшін бөлек `:callback` lane key — `update.callback_query` check қосу.
**Өзгеріс**: sequentialize key function-да:

```typescript
// Before: all updates from same chat share one lane
// After: callback_query gets separate `:callback` lane
if (update.callback_query) {
  return chatKey + ":callback";
}
return chatKey;
```

### 2. Plugin approval callback hook-only bypass

**Файл**: `src/telegram/bot-handlers.ts`
**Import қосылды** (line 63): `import { getGlobalHookRunner } from "../plugins/hook-runner-global.js";`
**Мәселе**: `ta:*` callback patterns (plugin approval buttons) толық agent run (`processMessage`) іске қосады — қымбат LLM API calls, thinking spoilers, unwanted agent responses.
**Шешім**: `ta:*` pattern detect → `hookRunner.runMessageReceived()` тікелей шақыру, `processMessage` skip.
**Жолдар**: ~1002-1026 (catch-all `processMessage` алдында)

```typescript
if (/^ta:[a-z0-9]+:[yan]$/.test(data)) {
  const hookRunner = getGlobalHookRunner();
  if (hookRunner?.hasHooks("message_received")) {
    hookRunner.runMessageReceived(
      { from, content: data, timestamp: Date.now(), metadata: { to, provider: "telegram", ... } },
      { channelId: "telegram", accountId },
    );
  }
  return; // Skip processMessage entirely
}
```

### 3. inlineButtons capabilities config

**Файл**: OpenClaw config (`~/.openclaw/openclaw.json`)
**Мәселе**: Default `inlineButtonsScope = "allowlist"` → plugin inline button callbacks `isAllowlistAuthorized()` check-те блокталады (line 847).
**Шешім**: Telegram секциясына `"capabilities": { "inlineButtons": "all" }` қосу.
**Орны**: Config файл, source code емес.

## Docker Runtime Patches

Docker container ішіндегі compiled bundle (`/app/dist/reply-6VY-Jwh5.js`) runtime-да Python script арқылы патчталады. Source repo-дағы patches Docker image rebuild кезінде автоматты қолданылады.

**Маңызды**: Compiled bundle файл атауы build-ке байланысты өзгеруі мүмкін — `index.js` ішіндегі import-тан тексеру: `docker exec CONTAINER sh -c 'head -5 //app/dist/index.js'`
