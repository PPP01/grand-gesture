# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Grand Gesture is a **Chrome Manifest V3 extension** for mouse/rocker/wheel/touch gestures and super-drag, forked from the discontinued [smartUp Gestures](https://github.com/zimocode/smartup). There is also legacy compatibility code for Firefox/Edge (see the browser-detection shim below), but Chrome is the primary target.

## Critical fact: the extension code is NOT compiled

The shipped extension is hand-written JavaScript in `js/`. There is **no transpile/bundle step** for it — `manifest.json` loads `js/*.js` directly. Editing a file in `js/` and reloading the unpacked extension is the entire build loop.

TypeScript and the `launchpad`/`Makefile`/`tsconfig`/`biome` toolchain only exist for **build/dev tooling** under `bin/` (e.g. `bin/locales.ts`). Do not assume `js/*.js` is generated output — it is the source.

## Loading & testing locally

Load the repo root as an unpacked extension (Developer Mode on) — the maintainer runs it in **Brave** (`brave://extensions`); plain Chrome works identically (`chrome://extensions`). The manifest is at the repo root. After editing `js/`, click reload on the extension card. There is **no automated test suite** (`LP_CFG_TEST_RUNNER = disabled`).

Brave is Chromium-based, so the browser-detection shim treats it as Chrome (`"cr"` — it only special-cases `firefox`/`edge`). If a gesture misbehaves only in Brave, suspect Brave Shields blocking page content/scripts or a missing optional permission before the extension code.

## Commands

Build tooling runs through `make` (a launchpad-based Makefile) and `pnpm`/`tsx`:

- `make release` — assembles a release zip into `dist/`. Syncs `manifest.json`'s version from the top entry of `CHANGELOG.md`, then zips `_locales css html image js CHANGELOG.md LICENSE manifest.json README.md`.
- `make begin` / `make end` — start/finish a translation pass (`tsx bin/locales.ts begin-translation|end-translation`). `begin` injects unique per-message codes and seeds `[TODO]` entries from English into every locale; `end` strips the codes and drops untranslated keys.
- `make format` — Biome formatting (config extends `.launchpad/biome.default.json`; 4-space indent).

The version number is the source of truth in `CHANGELOG.md` (`## [x.y.z]` heading); `bin/update-version-number.sh` propagates it into `manifest.json`. Bump the changelog, not the manifest, when releasing.

## Architecture

Three runtime contexts, communicating over `chrome.runtime` messaging:

1. **Content script — `js/event.js`** (injected into `<all_urls>`, all frames, at `document_start`). Detects raw input (mouse moves, drags, rocker/wheel/touch/double-click), recognizes gestures, draws the on-screen gesture trail/tooltip, and sends the matched action to the background via `chrome.runtime.sendMessage`. The big `sue` object holds its state and handlers.

2. **Background service worker — `js/background.js`** (~5200 lines, the largest file). Receives action messages and performs the privileged work (tab/window/history/bookmark/download operations). Actions that must run page-side are performed by injecting a one-off script with `chrome.scripting.executeScript({ files: ["js/inject/<x>.js"] })`. Also owns the config: loads/migrates/saves it to `chrome.storage` (see `getDefault.value()` — `version: 46` is the **config schema version**, unrelated to the extension version).

3. **UI pages** — `html/` + matching `js/`:
   - `js/options.js` (~4600 lines) drives `options.html`, the full settings UI where gestures are bound to actions.
   - `js/popup.js` drives the toolbar popup.
   - `js/getpermissons.js` / `js/getbackup.js` handle optional-permission prompts and config backup/restore.

Supporting modules in `js/`: `actions.js` (the **action registry** — `actions.mges`/`mges_group` etc. define every bindable action, its group, and which option fields/checkboxes/selects it exposes; this is the schema the options UI renders from), `ui.js`, plus vendored libs (`purify.js`, `qrcode.js`, `sortable.js`, `md5.js`, `gbk.js`, `base64.js`).

**`js/inject/`** — the "mini apps" and page-side action helpers (tab/history/bookmark list widgets, base64/case/numeral converters, QR code, auto-reload, RSS, scroll/zoom, etc.). Each is injected on demand by the background worker, not declared in the manifest.

### Browser-detection shim

`js/namespace.js` (and inline copies at the top of `background.js`, `event.js`, etc.) detect Firefox/Edge via the user-agent and alias `chrome = browser`, mapping `chrome.storage.sync` to `chrome.storage.local` on non-Chrome. Keep this pattern when touching those entry files.

### Permissions

`manifest.json` requests a minimal base set and gates everything else (bookmarks, history, downloads, clipboard, sessions, etc.) behind `optional_permissions`, requested at runtime via `js/getpermissons.js`. When adding an action that needs a new API, prefer an optional permission and request it on use.

## i18n

User-facing strings live in `_locales/<lang>/messages.json` (`en`, `it`, `pt_BR`, `ru`, `zh_CN`, `zh_TW`), referenced as `__MSG_key__` in the manifest and via `chrome.i18n.getMessage(...)` in code (`en` is the source language). Edit English strings, then use the `make begin`/`make end` translation workflow rather than hand-editing other locales.
