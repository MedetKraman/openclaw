# OpenClaw Paidax Patches

Патчтар OpenClaw fork-ына жасалған өзгерістер. Docker image қайта build кезінде осы файлды тексеру.

## Base Version

OpenClaw commit: `d76b224` (v2026.3.1 — DM topics, adaptive thinking, healthz endpoints)

Previously: `9231d7d` (v2026.2.21)

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
if (/^(ta:[a-z0-9]+:[ytan]|perm:(revoke|close)(:.+)?)$/.test(data)) {
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

### 4. python3-pip in Dockerfile

**Файл**: `Dockerfile`
**Мәселе**: `node:22-bookworm` base image-де pip жоқ — `install-diarize.sh` жұмыс істемейді.
**Шешім**: `apt-get install -y python3-pip` Dockerfile `ARG OPENCLAW_DOCKER_APT_PACKAGES` блогінде.
**Қашан қосылды**: v2026.3.1 merge (2026-03-03)

### 5. BuildKit cache mount for pnpm

**Файл**: `Dockerfile`
**Мәселе**: `pnpm install --frozen-lockfile` lock file өзгерген сайын барлық node_modules (~500MB) қайта жүктейді.
**Шешім**: BuildKit `--mount=type=cache` — pnpm store Docker volume-де (`openclaw-pnpm-store`) тұрақты сақталады. Lock file өзгерсе де жеке пакеттер қайта жүктелмейді.
**Build команда**: `build-local.sh` скриптін қолдану (`DOCKER_BUILDKIT=1 --cache-from` қосулы).
**Қашан қосылды**: 2026-03-03

## Docker Runtime Patches

Source patches are compiled into `openclaw-medet:local` via Docker build.
No runtime patching needed — all patches live in the source code.

**Build command (with caching):**
```bash
wsl -e bash /mnt/d/source/MedetCTO/08-integrations/openclaw/build-local.sh
```

**Verify patches are compiled in:**
```bash
wsl -e docker exec openclaw-repo-openclaw-gateway-1 sh -c 'grep -c "ytan" /app/dist/reply-*.js'
# Expected: 1
```
