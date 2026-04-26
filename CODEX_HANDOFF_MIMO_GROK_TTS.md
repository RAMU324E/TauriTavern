# MiMo / Grok TTS Handoff

## Scope

This branch is intended to contain only MiMo TTS and xAI Grok TTS work.

Anything unrelated to those providers was explicitly reverted before handoff.

Current branch:

- `codex/add-mimo-grok-tts`
- local HEAD: `5a82b1d39f86d07ab821e0b7224fe47087266f48`
- upstream main at handoff: `e1b67af0bf603fdd56ca3da536fb0e775cefc6ef`

## Files Changed Vs Upstream

These are the intended branch-level changes compared to `upstream/main`:

- `src/scripts/extensions/tts/grok.js`
- `src/scripts/extensions/tts/index.js`
- `src/scripts/extensions/tts/mimo.js`
- `src/scripts/secrets.js`
- `src/tauri/main/routes/index.js`
- `src/tauri/main/routes/tts-routes.js`
- `tests/tts-routes-contract.test.mjs`

No other files should be part of the final feature branch.

## Key Files

### Frontend providers

- `src/scripts/extensions/tts/grok.js`
  - Grok provider implementation
  - provider settings UI
  - voice lookup
  - preview generation

- `src/scripts/extensions/tts/mimo.js`
  - MiMo provider implementation
  - model selection
  - format selection
  - voice lookup
  - preview generation

### TTS registry

- `src/scripts/extensions/tts/index.js`
  - only minimal provider registration/import wiring should live here
  - no broad common-logic experiments should remain here

### Secret wiring

- `src/scripts/secrets.js`
  - adds the MiMo secret key id

### Tauri backend routes

- `src/tauri/main/routes/tts-routes.js`
  - backend proxy for Grok and MiMo
  - keeps API keys off the frontend
  - Grok voice list route
  - Grok audio generation route
  - MiMo audio generation route

- `src/tauri/main/routes/index.js`
  - route registration hook

### Tests

- `tests/tts-routes-contract.test.mjs`
  - backend route contract tests
  - Grok generate contract
  - Grok voice list contract
  - MiMo generate contract

## Current Intended Behavior

### Grok

- Provider exists in TTS provider list as `Grok`
- Voice list can be fetched from backend route instead of staying hardcoded only
- Fallback list still exists if remote voice fetch fails
- Current expected voices include:
  - `ara`
  - `eve`
  - `leo`
  - `rex`
  - `sal`
  - `una`

### MiMo

- Provider exists in TTS provider list as `MiMo`
- Supports:
  - `mimo-v2-tts`
  - `mimo-v2.5-tts`
- `mimo-v2-tts` voice list remains limited to the small official preset set
- `mimo-v2.5-tts` has the larger voice list
- Frontend settings include output format selection:
  - `wav`
  - `mp3`

## Local Working Tree At Handoff

There are currently uncommitted MiMo/Grok-only refinements on top of `5a82b1d`.

These are the only remaining local diffs:

- `src/scripts/extensions/tts/grok.js`
- `src/scripts/extensions/tts/mimo.js`
- `src/tauri/main/routes/tts-routes.js`
- `tests/tts-routes-contract.test.mjs`

### What those local diffs do

- `grok.js`
  - adds `una` to fallback voices
  - fetches voices dynamically from `/api/tts/grok/voices`

- `mimo.js`
  - exposes output format selection in UI
  - persists `format` changes

- `tts-routes.js`
  - adds `/api/tts/grok/voices`
  - removes static Grok voice whitelist enforcement in backend
  - adds lightweight retry logic for transient upstream failures
  - uses retry helper for Grok and MiMo upstream fetches

- `tts-routes-contract.test.mjs`
  - adds coverage for Grok voice list route
  - updates error-path test input to use `una`

## Explicitly Reverted Before Handoff

These were local debugging or broad behavior changes and were intentionally removed:

- `src/init.js`
  - bootstrap failure overlay
  - gray-screen diagnostics

- `src/scripts/openai.js`
  - unrelated `quietPrompt` rename

- `src/scripts/extensions/tts/index.js`
  - all common TTS behavior experiments
  - auto-play queue wake-up experiments
  - system-message filtering experiments
  - shared audio playback pipeline changes
  - debug logging

- `src-tauri/tauri.isolated.dev.json`
  - isolated local debug config

Those removals were done because the user requested that only MiMo/Grok work remain, and because broad TTS/common-layer edits were affecting unrelated providers.

## Important Constraint

If automatic playback is still broken for MiMo/Grok, do not immediately touch the shared logic in `src/scripts/extensions/tts/index.js` again.

First prove the failure path with targeted logs and compare:

- provider preview path
- provider `generateTts()` path
- common queue/audio playback path
- behavior on desktop vs mobile/WebView

The user specifically reported that broad common-layer changes caused regressions for other providers, including `GPT-SoVITS-Adapter`.

## Suggested Next Checks

1. Re-test on a clean branch state with only the intended MiMo/Grok files changed.
2. Verify manual preview and automatic playback separately.
3. If auto playback still fails only for MiMo/Grok:
   - inspect response MIME types
   - inspect whether generated responses differ from providers that already work
   - inspect mobile/WebView playback restrictions
4. Keep any future fixes tightly scoped to MiMo/Grok unless a shared-root cause is proven.

## Useful Commands

Check branch diff vs upstream:

```powershell
git diff --stat upstream/main...HEAD
```

Check local uncommitted diff:

```powershell
git diff --name-only HEAD
```

Run targeted route tests:

```powershell
node --test tests/tts-routes-contract.test.mjs
```

Run isolated local app if needed:

```powershell
corepack pnpm exec tauri dev --config src-tauri/tauri.isolated.dev.json
```

Note: the isolated config itself was intentionally removed from the final scoped work and should not be pushed unless someone explicitly decides to keep a debug-only config.
