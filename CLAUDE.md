# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Parable is a mobile-first chat client for Claude ("Chat for Fable 5"), built as a **single self-contained HTML file** — inline CSS, vanilla JS (`"use strict"`), no framework, no build system, no package.json, no tests, no linter. All images are inlined as base64 data URIs (the `ASSETS` object and the favicon), which is why the files are ~500 KB and why many lines are extremely long — prefer `grep -n` and line-capped reads (`awk '{print substr($0,1,160)}'`) over dumping whole sections.

## Files and versioning workflow

- `vNN.html` (currently `v40.html`) — the latest working version. Iteration happens by creating a new `vNN+1.html` and deleting the superseded one (see git history: "Add files via upload" / "Delete v39.html").
- `index.html` — the deployed copy, updated separately and usually a few versions behind (currently v36).
- The version lives in the `<title>` tag (`Parable — Chat for Fable 5 · v40`). Bump it when creating a new version. CSS/JS sections carry version-tagged banner comments (`/* ===== v25 · LONG-PRESS MENU ===== */`, `/* ===== v30 · LIVE BACKEND LAYER ===== */`) — keep that convention when adding features.
- `<script src="capacitor.js">` references a file **not in this repo**; it is injected by the Capacitor native shell when the app is packaged for mobile. In a plain browser it 404s harmlessly.
- Features hidden for the v1 release are marked with `data-deferred="v1.1 <name> — see PARABLE-LEDGER.md #DEF-n"` attributes (the ledger file itself is not in this repo). Re-enable one by deleting its `data-deferred` attribute.

## Running and testing

- Open the HTML file directly in a browser — no server needed.
- **Demo mode vs live mode:** `__LIVE()` returns true when `CONFIG.WORKER_URL` and `CONFIG.SUPABASE_ANON_KEY` are set (they are baked in), and nearly every interactive path branches on it. Demo mode is a scripted prototype (`routeDemo()`): type "fail" or "broke", use the paperclip, or long-press a reply to exercise canned flows.
- **Self-test:** append `?selftest` to the URL to open a built-in diagnostics panel (`selfTest()`, bottom of the file). It checks worker reachability, Supabase REST, `/v1/estimate`, and a live `/v1/chat` stream, and prints a screenshot-friendly report. This is the closest thing to a test suite — run it when touching the live layer.
- Global `error` / `unhandledrejection` handlers surface exceptions as toasts, so runtime errors are visible on-device without a console.

## Architecture (inside the single file)

Order within the file: CSS → HTML (main chat panel, `<!-- ONBOARDING -->`, `<!-- SHEETS -->`, `<!-- LIBRARY -->`) → JS.

### Theming
CSS custom properties on `:root` (dark, default) with a full override set under `[data-theme="light"]`. Any new color must be defined in both. `applyTheme()` also syncs the `#meta-theme` theme-color tag.

### Config and auth (v30 "LIVE CONFIG")
- `CONFIG` holds `SUPABASE_URL`, `WORKER_URL` (Cloudflare Worker: `parable-api.muttyrabb.workers.dev`), and `SUPABASE_ANON_KEY`. **As of v40 the baked values are the only truth** — legacy localStorage overrides (`parable_worker`, `parable_anon`) are purged on boot and must never be read again (a stale override once silently pointed worker calls at a dead address).
- Auth is Supabase email OTP (`/auth/v1/otp` → `/auth/v1/verify`); the session persists in localStorage as `parable_session`. A 401 anywhere calls `authExpired()`, which clears the session and reopens onboarding.

### Live backend layer (v30)
Two fetch wrappers, split by concern:
- `api(path, opts)` → the Cloudflare Worker, which proxies the Anthropic API and handles billing. Endpoints: `/v1/me` (balance/plan), `/v1/estimate` (cost preview), `/v1/chat` (SSE stream), `/v1/title` (auto-titles a "New chat" after the first exchange).
- `sbFetch(path, opts)` → Supabase REST (PostgREST) for persistence: `chats` (id, title, updated_at) and `messages` (chat_id, role, content, cost_micro) tables, rows scoped by `session.uid`.

`streamLive()` parses the worker's SSE stream by hand: Anthropic `content_block_delta` events render text incrementally; a custom `parable_billing` event carries `cost_usd` and the updated `balance_usd`. Costs display per-message (stored as `cost_micro`, micro-dollars).

### Modes and billing
The mode dial maps to `dial: "quick" | "standard" | "deep"` in worker requests, plus an **opt-in only** `smart_saver` flag (`mode === "saver"`); every other mode is guaranteed to hit Fable. Balance shows in the header (`setBalanceUsd`); pricing/top-up links open `PRICING_URL` (`parable-site.html`, external to this repo).

### Chat flow
`submit()` branches: live mode → `liveSend()` (ensures a chat row exists via `ensureChat()`, builds Anthropic-format content blocks in `buildContent()` — text plus base64 image/PDF attachments — streams the reply, saves both sides with `saveMsg()`); demo mode → `routeDemo()` canned responses typed out by `streamReply()`.

## Conventions

- Keep everything in the one file — no external JS/CSS files, no npm dependencies. The only external requests at rest are Google Fonts.
- Vanilla DOM style throughout: `$id()` / `$` helpers, `document.createElement`, event listeners wired at bottom of the script; match it.
- Backend failures degrade with `showToast(...)` messages, never blank UI; fire-and-forget writes (e.g. `saveMsg`) swallow errors with `.catch(() => {})`.
