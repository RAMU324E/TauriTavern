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
- `src/scripts/extensions/tts/gpt-sovits-adapter.js`
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

### Non-MiMo/Grok runtime safety fix

- `src/scripts/extensions/tts/gpt-sovits-adapter.js`
  - fixes a provider-endpoint input selector bug
  - without this fix, the GPT-SoVITS adapter UI can appear to accept a custom endpoint but still keep using the default endpoint internally
  - this affected testing in the isolated instance and can affect normal usage too

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

One more fix was added later because it directly affected network TTS testing:

- `src/scripts/extensions/tts/gpt-sovits-adapter.js`

That fix should also stay.

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

## Isolation Details

The isolated instance does not change application logic. It changes identity and storage location.

### What isolation changes

- Tauri app identifier
  - normal app: `com.tauritavern.client`
  - isolated app: `com.tauritavern.client.isolated`

- persistent app-data directory
  - normal app data and settings are separate from isolated app data and settings

- WebView profile
  - because the app identifier changes, the isolated instance gets its own separate WebView state instead of sharing the normal installed app's state

### What isolation does not change

- frontend code paths
- backend route code paths
- outbound HTTP request logic
- MiMo route logic
- Grok route logic
- GPT-SoVITS adapter fetch logic

If a provider works in the normal app but not in isolated mode, first suspect:

- different settings
- different saved secrets
- different voice map
- different provider endpoint value

Do not first assume isolation changed the provider code path.

### Important isolated data locations

In this environment, isolated instance data lives under:

- `C:\Users\Admin\AppData\Roaming\com.tauritavern.client.isolated\data\default-user\settings.json`
- `C:\Users\Admin\AppData\Roaming\com.tauritavern.client.isolated\data\default-user\secrets.json`
- `C:\Users\Admin\AppData\Roaming\com.tauritavern.client.isolated\data\default-user\tauritavern-settings.json`
- `C:\Users\Admin\AppData\Roaming\com.tauritavern.client.isolated\logs`

This means:

- changing provider settings in the normal app does not update the isolated app
- adding API keys in the normal app does not add them to the isolated app
- changing TTS voice maps in the normal app does not update the isolated app

### Recommended isolated launch commands

Primary:

```powershell
corepack pnpm exec tauri dev --config src-tauri/tauri.isolated.dev.json
```

If `cargo` is missing from `PATH`:

```powershell
$env:PATH="$env:USERPROFILE\.cargo\bin;$env:PATH"; corepack pnpm exec tauri dev --config src-tauri/tauri.isolated.dev.json
```

## GPT-SoVITS Adapter Notes

This section is important because the user specifically hit a custom endpoint failure.

### Key distinction from MiMo and Grok

MiMo and Grok use Tauri backend proxy routes:

- MiMo: `/api/tts/mimo/generate`
- Grok voices: `/api/tts/grok/voices`
- Grok generate: `/api/tts/grok/generate`

GPT-SoVITS-Adapter does not use a Tauri backend proxy. It does direct frontend fetches to the configured endpoint.

Relevant requests:

- `GET {provider_endpoint}/speakers`
- `POST {provider_endpoint}/`

Relevant file:

- `src/scripts/extensions/tts/gpt-sovits-adapter.js`

### Conclusion: is `192.168.31.206:9881` failure caused by isolation?

Usually, no.

Isolation does not rewrite the GPT-SoVITS endpoint or route its traffic differently. The provider fetches directly from the frontend.

What isolation can change is only the saved value of `provider_endpoint`, because isolated settings are stored separately.

### Actual bug found in this repo

There was a selector mismatch in `gpt-sovits-adapter.js`:

- settings HTML input id:
  - `gpt_sovits_adapter_tts_endpoint`
- `loadSettings()` was incorrectly binding to:
  - `#tts_endpoint`

Impact:

- the endpoint field could display one value while the actual bound setting logic did not initialize or save correctly
- custom endpoints like `http://192.168.31.206:9881` could appear to be entered, but the provider could continue behaving as if the default endpoint was still in effect

Fix:

- bind `loadSettings()` to `#gpt_sovits_adapter_tts_endpoint`

### Other likely failure causes for GPT-SoVITS custom endpoint

If it still fails after the selector fix, check these in order:

1. The isolated instance may still have old provider settings in its own `settings.json`.
2. The GPT-SoVITS adapter service may only be bound to `127.0.0.1` on the server machine instead of `0.0.0.0`.
3. Windows firewall on the machine at `192.168.31.206` may block port `9881`.
4. The adapter may not expose exactly:
   - `GET /speakers`
   - `POST /`
5. The service may be reachable from browser on the host machine but not reachable from the Tauri desktop machine over LAN.
6. CORS may be blocked if the adapter does not allow the Tauri dev origin.

### Fast manual checks for GPT-SoVITS endpoint

From the same machine running TauriTavern:

```powershell
Invoke-WebRequest http://192.168.31.206:9881/speakers
```

If that fails, the problem is below the app layer.

If it succeeds, next verify that the adapter accepts the expected POST shape from `gpt-sovits-adapter.js`.

### Why this matters for future debugging

If GPT-SoVITS fails in isolated mode, do not immediately blame:

- MiMo changes
- Grok changes
- Tauri route changes

First separate these categories:

- startup problem
- isolated settings problem
- provider endpoint save/load problem
- LAN reachability problem
- adapter API contract problem

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
7. If GPT-SoVITS custom endpoints stop working, inspect `gpt-sovits-adapter.js` before touching isolation, MiMo, or Grok code.

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
