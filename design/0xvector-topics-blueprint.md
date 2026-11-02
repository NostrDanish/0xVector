# 0xVector — Telegram-Style Topics on a Concord Core
## Architecture & Implementation Blueprint

**Date:** 2026-09-19
**Repos:** `NostrDanish/0xVector` (fork of `VectorPrivacy/Vector`, Tauri v2 + Rust `vector-core` + vanilla JS), `concord-protocol/concord` (CORD specs), `soapbox-pub/armada` (reference Concord client), Soapbox `ditto` / `ditto-relay` (infrastructure).

---

## 0. Executive summary

Vector already *is* the censorship-resistant private messenger Telegram claims to be:
no phone number, no email, NIP-17 gift-wrapped DMs, embedded Tor, Concord E2E
communities where authority is a signed roster and relays see only ciphertext.
What it lacks is Telegram's *shape*: a group that opens into a list of **topics**
instead of one scrolling firehose.

The good news from the code and spec audit: **topics do not require new
cryptography.** Concord's CORD-01 binding mechanism ("commit the sub-context as
tags inside the signed rumor, check it against the coordinate whose key opened
the wrap") is exactly the primitive Telegram's `message_thread_id` is. We add a
`["topic", id]` sub-context tag to the sealed rumor layer, a client-side topic
registry, and a Telegram-faithful topics UI. Zero new relay-visible metadata,
zero rekey cost, graceful degradation to Armada and stock Vector clients.

**Recommended design: in-envelope topic streams (one channel key, many topics),
with a signed topic registry — proposed upstream as CORD-09.**

Rejected alternative: topics-as-channels. It is spec-legal today but costs one
relay-visible address per topic (leaks topic count and per-topic activity
timing), one subscription author per topic per client, burns against the
256-channel cap, and — worst of all — channel creation is gated on the
`MANAGE_CHANNELS` permission, while Telegram's whole point is that *members*
create topics. Per-topic key isolation is already available via Private
Channels for the rare case that needs it; Telegram topics have no per-topic
privacy either.

---

## 1. Ecosystem map (what you actually have)

| Piece | What it is | Role in this plan |
|---|---|---|
| **Vector / 0xVector** | Tauri v2 app. `crates/vector-core` = all business logic (single source of truth), `src-tauri` = thin command shell (150+ commands), `src/` = vanilla JS SPA (`main.js` 14.5k lines, `styles.css` 19.4k lines, `src/js/render/*` renderers), `crates/vector-sdk` = bot SDK v0.9.0 | The product |
| **Concord** | Protocol spec (CORD-01…08): private streams, communities, channels, roles, invites, rekeys, A/V, disappearing messages | The group-chat layer topics plug into |
| **Armada** | Soapbox's reference Concord client (React 19 + Nostrify). *Not a protocol* — "Armada" is the app; Concord is the spec. `blckbx/armada-bot` is just an OpenClaw AI-assistant plugin for Armada DMs | Interop partner: propose CORD-09 so Armada renders topics natively instead of as flat channel messages |
| **Ditto** | Soapbox's nostr stack: `ditto-relay` (OpenSearch-backed, horizontally scalable relay, powers `wss://relay.ditto.pub`), the Ditto client, and the legacy Mastodon-API community server | Recommended self-hostable relay for communities; already in Concord's stock relay dictionary |
| **Vector SDK** | `VectorBot` builder, unified `Channel` handle (DM or community channel, auto-detected), full community API: `channels()`, `create_channel`, `members()`, `ban/kick`, invites, slash commands | Extend with `topics()` — and note: unlike Telegram's Bot API (which *cannot list topics*), the Vector SDK reads local state, so 0xVector bots get full topic listing. A genuine "better than Telegram" SDK win |
| **Sentire** | VectorPrivacy's moderation bot (text screening, raid containment) | Becomes topic-aware for free via the SDK extension |

### Current UI reality (verified against `master` @ 0ca50a9)

- Every community channel is a chat object in `arrChats`, but **only the
  primary channel gets a chat-list row** (`isPrimaryChannelChat`,
  `src/js/render/chatlist/list.js:28`). There is **no channel sidebar, switcher,
  or picker anywhere** — a community row opens its primary channel directly.
- `VectorCore::create_channel/delete_channel/grant_channel_access` exist in core
  and SDK but are **not registered as Tauri commands** — channel management UI
  was never built.
- Message rendering is Discord-style rows (`.dmsg-*`, `message-row.js`) with
  reply context, swipe-to-reply, windowed scrolling (80-row window,
  `chat-scroll.js`), community catch-up via `sync_community_channel`.
- Themes are small override sheets over CSS custom properties
  (`src/themes/<name>/dark.css`), so the topics UI inherits every existing
  theme for free if built on the same variables.

**This is the opening:** a Telegram-topics UX is not a retrofit against an
existing Discord-style channel UI — it *is* the first channel-selection UI
0xVector gets. We design it Telegram-shaped from day one.

---

## 2. Target UX spec (Telegram-faithful)

Modeled on Telegram forum groups (Bot API 6.3/6.4 semantics, TDLib topic model):

1. **Forum mode per community.** Community settings get a "Topics" toggle
   (Telegram: Group Settings → Topics). Enabling stamps `forum: true` into the
   community metadata. Non-forum communities behave exactly as today.
2. **Community row → topic list.** Tapping a forum community in the chat list
   opens a **topic list** that reuses the chat-list visual language: per-topic
   row with icon, name, last-message preview, unread counter, unread-mention
   badge, pin state, mute state (mirrors TDLib's `forumTopic` fields).
3. **General topic.** Always exists, undeletable, renamable/hideable by staff.
   All pre-forum history lives there. On the wire: *no topic tag = General* —
   which is what makes this backward-compatible and migration-free.
4. **Topic lifecycle.** Create (name 1–64 chars, color + emoji icon — like
   Telegram's `icon_color`/`icon_custom_emoji_id`), rename, close/reopen
   (read-only for non-staff), pin/reorder, per-topic mute, delete (staff only,
   irreversible — same as Telegram).
5. **Permissions.** Telegram's `can_manage_topics` maps to a Concord role
   permission `MANAGE_TOPICS`. Default in casual groups: everyone creates
   topics; staff can restrict. Enforcement is client-side roster verification,
   as with all Concord authority — a forged topic-create is rejected by every
   honest client.
6. **Composer.** Inside a topic, the composer tags every send with the topic
   id. Reply-to (NIP-10-style `replied_to`, already implemented) composes
   orthogonally — a reply inside a topic carries both.

---

## 3. Protocol design (the CORD-09 proposal)

### 3.1 Wire format — one additive tag

Concord chat-plane rumors today: kind `9` message, `1111` reply, `7` reaction,
`3302` edit, `5` delete, all sealed (kind `20013`) inside kind-`1059` wraps
signed by the channel's derived group key, with mandatory binding tags
`["channel", channel_id]` + `["epoch", n]`.

Add:

```
["topic", "<topic_id>"]        // on kind 9/1111/7/3302/5 rumors
```

- `topic_id` = the rumor id of the topic's creation event (mirrors Telegram,
  where `message_thread_id` equals the topic's creation service-message id).
  Self-certifying, collision-free, no registry lookup needed to route.
- Checked with the same strict-equal discipline as `check_channel_binding` in
  `v2/stream.rs`: a rumor claiming a topic that doesn't exist in the registry,
  or created by someone without `MANAGE_TOPICS`, is dropped.
- Unknown-tag round-tripping is Concord's sanctioned additive-change path:
  Armada and stock Vector parse these messages as ordinary channel messages.
  **Graceful degradation is structural, not negotiated.**

### 3.2 Topic registry — signed rumors, not control-plane entities

Topic metadata rides the same sealed chat plane as lifecycle rumors (suggest
kind `3311` from the append-only registry):

```json
{ "kind": 3311, "content": "{\"name\":\"General\",\"icon\":\"💬\",\"color\":6}",
  "tags": [["topic-op","create"], ["channel","<id>"], ["epoch","<n>"]] }
```

`topic-op` ∈ `create | edit | close | reopen | delete | pin`. Every member
folds these into local state (same pattern as the roster fold on the control
plane, but channel-scoped). Delete is terminal; the topic's messages stay but
the topic unlists — same as Telegram.

**Why not ChannelMetadata `custom`?** CORD-02 §6 explicitly permits a `custom`
object on channel metadata, but editing it is `MANAGE_CHANNELS`-gated control-
plane work. Member-created topics need channel-plane authority, so the
registry lives where the authority is.

### 3.3 Privacy & cost analysis

| Property | Topics-as-tags (chosen) | Topics-as-channels (rejected) |
|---|---|---|
| Relay-visible metadata | **None** — one address per channel regardless of topic count | Topic count, per-topic activity timing/volume visible |
| Rekey on member removal (CORD-06) | **O(1)** — channel rekey covers all topics | O(retained members) blobs per private topic; one rekey-plane author per topic in every subscription |
| Channel cap | Unaffected | Burns against 256-channel cap (`CommunityV2::from_bundle`) |
| Member topic creation | Role-gated tag, works | Blocked by `MANAGE_CHANNELS` |
| Per-topic read isolation | None (= Telegram) | Possible via private channels |
| Protocol change | Additive tag + one kind | None, but authority change still needed |

The one honest tradeoff: a removed member keeps historical reads of all topics
in a channel, exactly as they do for the channel itself. Per-topic
post-removal secrecy is impossible without per-topic keys — and Telegram
doesn't have it either. Communities needing compartmentalization use Private
Channels, which already rekey properly.

---

## 4. Implementation plan (by layer)

### Phase 0 — Spec upstream (1–2 weeks, mostly writing)
Draft **CORD-09: Topics** against `concord-protocol/concord`: the tag, the
registry rumor kind, the `MANAGE_TOPICS` permission (CORD-04 amendment),
interop/degradation rules. Socialize with the Armada maintainers — if Armada
implements it, topics are cross-client from day one; if not, they degrade
cleanly. This is cheap and prevents a forked protocol.

### Phase 1 — vector-core (`crates/vector-core`)
- `community/v2/chat.rs`: `build_message_rumor` / `build_comment_rumor` accept
  `topic_id: Option<&str>`, emitted via a `topic_binding_tag()` next to
  `stream::channel_binding_tags`.
- `community/v2/stream.rs` + `inbound.rs`: parse + strict-check the topic tag;
  fold kind-3311 registry rumors into a `TopicRegistry` on `ChannelV2`
  (`community.rs`: `topics: Vec<TopicV2>` mirroring `ChannelV2`).
- `community/v2/roles.rs`: `MANAGE_TOPICS` permission bit; `can(permission)`
  check on registry ops (SDK `Member::can` already exists).
- `db/`: migration — `ALTER TABLE messages ADD COLUMN topic_id TEXT`, new
  `topics` table (channel_id, topic_id, name, icon, color, state, pinned_idx,
  unread denormals). **Heed the repo's landmines:** next id above
  `HIGHEST_MIGRATION_ID`, bump the constant (a test enforces it), never reuse
  burned ids 33–39/45–61. Per-account scoping is automatic on the `Session`.
- Unread counters per topic (the frontend's `update_unread_counter` command
  already exists per-chat; extend to per-topic-in-channel).
- Tests: golden-vector binding tests like `derive.rs`'s; a
  swap-during-topic-create abort test mirroring
  `a_swap_during_create_private_channel_aborts_without_a_write` (multi-account
  `SessionGuard` discipline is non-negotiable per CLAUDE.md); spawn-audit
  compliance (`db::spawn_bound`, no bare `tokio::spawn`).

### Phase 2 — Tauri shell (`src-tauri`)
New commands (each needs the **three registrations**: permission TOML in
`permissions/autogenerated/`, entry in `capabilities/default.json`,
`invoke_handler` in `lib.rs` — missing any = silent ACL rejection):
`list_channel_topics`, `create_topic`, `edit_topic`, `close_topic`,
`reopen_topic`, `delete_topic`, `pin_topics`, `set_topic_muted`,
plus `topic_id: Option<String>` added to `send_community_message` /
`send_community_files*` and topic filters on `get_message_views` /
`get_messages_around*` / `get_chat_message_count`.
`sync_community_channel` is unchanged — the channel cursor covers all topics.

### Phase 3 — Frontend (`src/`)
- **Topic list view:** new `src/js/render/topics/list.js` modeled directly on
  `render/chatlist/list.js` (same hash-gated re-render pattern, same row
  builder idioms from `render/chatlist/row.js`). Forum community rows in the
  chat list get an unread aggregate; tapping opens the topic list instead of
  the channel (`openChat` fork at `main.js:9073`, gated on
  `chat.metadata.custom_fields.forum`).
- **Topic rows:** icon (emoji in a colored circle, Telegram palette), name,
  preview via existing `preview.js`, unread + mention badges, pin/mute state,
  context menu reusing `_showChatRowContextMenu`'s pattern.
- **Conversation view:** opening a topic = `openChat(channelId)` with an active
  topic filter; `chat-scroll.js` windowing works unchanged (backend filters by
  topic_id); composer carries the open topic id through `message()` →
  `send_community_message`. Header shows `Community / Topic` breadcrumb with a
  back affordance to the topic list (mobile) — use the existing back-stack.
- **Topic management:** creation sheet (name + icon picker reusing the emoji
  pack picker), close/pin/mute in the row context menu, General-topic
  rename/hide in the Group Overview panel (`#group-overview`,
  `renderCommunityOverview`).
- **Community settings:** "Topics" toggle for owners, calling
  `update_community_metadata` with `forum: true` (custom_fields path already
  exists).
- **Theming:** build entirely on the existing CSS custom properties so all 7
  themes (vector, satoshi, monero, cyberpunk…) style topics automatically.
  Optional new "Telegram" theme as a `src/themes/telegram/dark.css` override
  sheet for familiarity during migration.

### Phase 4 — SDK (`crates/vector-sdk`) & agents
- `Community::topics() -> Vec<Topic>` (id, name, icon, color, state, pinned),
  `create_topic`, `edit_topic`, `close_topic`, `Channel::send_in(topic_id, …)`,
  `BotEvent::TopicCreated/Updated/Closed`, topic filter on `Channel::history*`.
- Slash-command args gain a `topic` choice type so bots can target topics.
- **Beating Telegram's bot API:** Telegram bots cannot list a group's topics
  (open API gap since 2022; bots reconstruct state from service messages).
  Vector SDK bots read local folded state, so `bot.community(id).topics()`
  just works — market this.
- The MCP server (`vector-agent`, 21 tools) gains topic tools; Sentire can
  screen per-topic.

### Phase 5 — Infrastructure & polish
- Recommend `wss://relay.ditto.pub` (or self-hosted `ditto-relay` /
  `armada-relay`) in community creation defaults — OpenSearch-backed,
  horizontally scalable, already the ecosystem default for Concord traffic.
- Per-topic notifications (Android background sync respects topic mute),
  per-topic unread in the app badge, topic icons in push payloads omitted for
  privacy (keep payload-free notification style).
- F-Droid/reproducible-build pipeline is delicate upstream (dedicated CI
  commits); keep topic assets deterministic (no new codegen-time hashing).

---

## 5. Risks & open questions

1. **CORD-04 permission granularity** — exact grant semantics of
   `MANAGE_CHANNELS`/`MANAGE_TOPICS` need a CORD-04 read before Phase 1 lands
   the permission bit (flagged by the protocol audit).
2. **Interop window** — between 0xVector shipping topics and Armada adopting
   CORD-09, Armada users see forum messages as one flat stream. Acceptable
   (Telegram's topics also collapse in clients without support), but coordinate
   early.
3. **Topic registry convergence** — registry folds from sealed history; a
   member joining mid-history must fold topic ops before rendering the topic
   list. The existing channel sync cursor handles ordering, but topic-delete
   tombstoning needs care so deleted topics don't resurrect on re-fold.
4. **`service.rs` is ~1 MB** — the v2 orchestration layer is huge. Keep topic
   logic in `chat.rs`/`stream.rs`/`community.rs` and touch `service.rs` only
   for wiring.
5. **Multi-account safety** — every new per-account cache goes on the
   `Session` via `scoped::<Marker>()`, every Tauri command that mutates state
   gets a `SessionGuard`. The repo's test suite enforces this; do not fight it.

---

## 6. Milestones

| # | Deliverable | Depends on |
|---|---|---|
| M1 | CORD-09 draft PR to concord-protocol | — |
| M2 | vector-core: tag, registry, DB migration, tests | M1 (feedback-tolerant) |
| M3 | Tauri commands + minimal topic list UI (forum toggle, create/read topics, send into topic) | M2 |
| M4 | Full Telegram UX: close/pin/mute, badges, breadcrumbs, General topic mgmt | M3 |
| M5 | SDK + MCP + Sentire topic awareness | M2 |
| M6 | Theme polish, notifications, docs, Armada interop test | M4, M5 |

---

*Appendix: verified code anchors available in the swarm research notes —
frontend map (chatlist/openChat/chat-scroll/commands), Concord channel model
(CORD-01…06 citations), Telegram topics UX/API (Bot API 6.3→9.4, TDLib
`forumTopic` model), Armada/Ditto roles.*
