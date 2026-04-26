# MiMo / Grok TTS Handoff

## Scope

This branch is intended to contain only MiMo TTS and xAI Grok TTS work.

Anything unrelated to those providers was explicitly reverted before handoff, except for one startup-safety fix and one isolated-launch config that were re-added later because the user could not boot the app reliably.

Current branch:

- `codex/add-mimo-grok-tts`
- branch cleaned/pushed commit: `50fc7ec`
- upstream main at handoff: `e1b67af0bf603fdd56ca3da536fb0e775cefc6ef`

## Files Changed Vs Upstream

These are the intended branch-level changes compared to `upstream/main`, plus the minimal startup fixes re-added later:

- `src/scripts/openai.js`
- `src/scripts/extensions/tts/grok.js`
- `src/scripts/extensions/tts/index.js`
- `src/scripts/extensions/tts/mimo.js`
- `src/scripts/secrets.js`
- `src/tauri/main/routes/index.js`
- `src/tauri/main/routes/tts-routes.js`
- `src-tauri/tauri.isolated.dev.json`
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

### Startup safety

- `src/scripts/openai.js`
  - contains a required syntax-fix for startup
  - without this fix the app can fail during boot with:
    - `SyntaxError: Identifier 'quietPrompt' has already been declared`
  - this is not a MiMo/Grok feature change, but it is required to avoid a gray screen / failed init in the user environment

- `src-tauri/tauri.isolated.dev.json`
  - isolated debug/dev config
  - restores the exact `tauri dev --config ...` workflow the user was using
  - uses a separate Tauri identifier to avoid colliding with the user's normal installed instance

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

At the time this file was updated for the startup issue, the working tree intentionally contains local, not-yet-pushed changes in:

- `src/scripts/openai.js`
- `src-tauri/tauri.isolated.dev.json`

### What those local diffs do

- `openai.js`
  - renames the local prompt variable from `quietPrompt` to `quietPromptEntry`
  - prevents a duplicate identifier declaration inside `populateChatCompletion(...)`
  - this is the specific fix for the boot-time syntax error / gray screen

- `tauri.isolated.dev.json`
  - reintroduces the isolated launch config
  - makes this command valid again:
    - `corepack pnpm exec tauri dev --config src-tauri/tauri.isolated.dev.json`
  - gives the isolated instance:
    - `productName = TauriTavern Isolated`
    - `identifier = com.tauritavern.client.isolated`

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

- `src/scripts/extensions/tts/index.js`
  - all common TTS behavior experiments
  - auto-play queue wake-up experiments
  - system-message filtering experiments
  - shared audio playback pipeline changes
  - debug logging

Those removals were done because the user requested that only MiMo/Grok work remain, and because broad TTS/common-layer edits were affecting unrelated providers.

Later, two of those removals were intentionally reversed because they directly blocked local startup:

- `src/scripts/openai.js`
- `src-tauri/tauri.isolated.dev.json`

Those two should stay unless someone proves a cleaner alternative.

## Startup / Gray Screen Notes

This section is important. It is the main source of confusion during the handoff.

### Problem 1: isolated launch command fails

Symptom:

- `corepack pnpm exec tauri dev --config src-tauri/tauri.isolated.dev.json`
- error says file not found

Cause:

- the isolated config file had been removed during scope cleanup

Fix:

- restore `src-tauri/tauri.isolated.dev.json`

Minimal file contents:

```json
{
  "$schema": "https://schema.tauri.app/config/2",
  "productName": "TauriTavern Isolated",
  "identifier": "com.tauritavern.client.isolated"
}
```

### Problem 2: app starts but goes gray / fails during boot

Observed root cause in this environment:

- `src/scripts/openai.js`
- function: `populateChatCompletion(...)`
- parameter list already contains `quietPrompt`
- later inside the same function a local `const quietPrompt = ...` is declared again

Failure symptom:

- startup can fail on import with:
  - `SyntaxError: Identifier 'quietPrompt' has already been declared`

Fix:

- rename the local variable to `quietPromptEntry`

Verification used:

```powershell
node --check src\scripts\openai.js
```

### Why this matters

Without the config file, the user's documented launch command is broken.

Without the `quietPrompt` rename, the app can still compile but fail at frontend boot and present as a gray screen or failed initialization.

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
5. Do not remove `src-tauri/tauri.isolated.dev.json` again unless you also update the documented launch workflow.
6. Do not revert the `quietPrompt` rename unless you also prove the duplicate-declaration startup fault is gone.

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

If `cargo` is missing from `PATH`, use:

```powershell
$env:PATH="$env:USERPROFILE\.cargo\bin;$env:PATH"; corepack pnpm exec tauri dev --config src-tauri/tauri.isolated.dev.json
```

Verify startup syntax fix:

```powershell
node --check src\scripts\openai.js
```
