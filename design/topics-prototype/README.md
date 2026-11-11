# 0xVector — Topics Design Prototype v5 (nostr-live)

A clickable, phone-and-desktop design prototype for **Telegram-style Topics on
0xVector** — forum groups whose messages split into named topics — built to be
100% implementable on Vector's existing Concord core, with **no new
cryptography**. v5 is a full motion + craft overhaul: spring physics,
swipe-to-reply, hold-to-record voice, and a CRT-phosphor cyberpunk shell over
the same real nostr pipeline.

Open `index.html` in any browser. No build step.

## Why this exists

Telegram's forum-topics UX, but on a messenger where relays see only
ciphertext. Topics ride as a **signed `["topic", id]` tag inside the sealed
Concord rumor** (CORD-01's binding mechanism anticipates exactly this) — so:

- **zero new relay-visible metadata** (one address per channel regardless of
  topic count or activity)
- **rekey stays O(1)** per channel on member removal (CORD-06)
- **members can create topics** (roster-gated `MANAGE_TOPICS`, unlike
  `MANAGE_CHANNELS`)
- **graceful degradation**: stock Vector / Armada clients render topic
  messages as ordinary channel messages until they adopt the convention
- topic id = the topic-creation rumor's id (same trick as Telegram's
  `message_thread_id`)

## New in v5 — motion & craft

- **Spring physics everywhere** — message pop-in with overshoot, sheets,
  modals, context menus, badges; shared `--spring` / `--spring-soft` tokens
- **Swipe-to-reply** with rubber-band pull + reply-hint pill (pointer events,
  mouse and touch)
- **Hold-to-record voice** — mic morphs into a live waveform recorder with
  timer and slide-to-cancel; voice bubbles play back with waveform progress
  and an on-device-Whisper transcription toggle
- **Typing indicator** bubbles before incoming replies land
- **Message status lifecycle** — sending clock → sent tick → glowing
  relay-ack double-tick (⇄ relay badge in LIVE mode)
- **Message grouping** — consecutive-sender bubbles collapse with tails and a
  last-in-group avatar; day pills + NEW MESSAGES divider; short conversations
  bottom-anchor like Telegram
- **CRT boot sequence** — phosphor terminal types keygen/HKDF/Tor checks with
  a progress bar (once per session, any key skips)
- **Ambient hacker shell** — drifting aurora glow, perspective grid,
  scanlines, vignette, faint screen flicker, theme-colored matrix rain behind
  the idle screen, glitching `0x` brand mark, RGB-split hover logo
- **Notched clip-path panels** — modals, context menus, toasts, inspector
- **Ripple** on every interactive surface; double-click quick-react with a
  flying emoji burst; context menus carry a quick-reaction row
- **Performance** — transform/opacity-only animation, `content-visibility`
  on list rows, rAF-throttled scroll handlers, zero external fonts, full
  `prefers-reduced-motion` support

## Carried over from v4

- **Topics panel** — icons, unread + mention badges, pins, mutes, closed
  state; announcement channels, discussion topics, voice rooms (CORD-07)
- **Full message ops** — reply / copy / pin / edit (kind 3302) / delete
  (kind 5) / reactions, pinned bar with cycling, per-topic search,
  disappearing-message timer (CORD-08 / NIP-40 expiration tags)
- **Real nostr pipeline, in-browser** — real secp256k1 keypair, real
  HKDF-SHA256 channel-key derivation (`"concord/channel" ‖ 0x00 ‖ id ‖ epoch`),
  real NIP-44 v2 seals, kind-1059 gift wraps; flip **DEMO → LIVE** in the
  protocol inspector to connect to real relays
- **Protocol inspector** — every UI action logs its rumor → seal → wrap chain
  with event kinds
- **Projects (NIP-34)** — repo grid with stat pills, 26-week activity
  heatmap, issues/patches/PRs with status gates, grid/list toggle, search
- **Login** — Create Account (real keypair), nsec/hex import (real bech32
  validation), Remote Signer with scannable `nostrconnect://` QR, Amber link
- **CORD-05 invites** — link + QR + direct-npub gift-wrap + pending revoke
- **Six themes** — `vector` (default), `armada` (Corsair), `terminal`
  (strict monochrome phosphor: mono font, text-shadow glow), `satoshi`,
  `monero`, `cyberpunk` — same CSS-variable override architecture
- **Mobile** — Telegram-style drill-down navigation + bottom tab bar;
  QA-tested at 414×880

## Files

- `index.html` — the whole prototype (single file + one vendored dep)
- `vendor/qrcode-generator.js` — qrcode-generator@2.0.4 (MIT), the same
  vendored lib Vector ships (`src/js/qrcode-generator.js`)

## Design source

Architecture blueprint: see `0xvector-topics-blueprint.md` (design notes) —
topic-as-tag vs topic-as-channel analysis, CORD citations, phased
implementation plan (M1 CORD-09 draft → M2 vector-core → M3 Tauri commands →
M4 UX → M5 SDK/MCP → M6 polish/interop).

_Prototype state is local mock data except the nostr layer, which is real._
