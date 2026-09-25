# lisa — Design v4.1

**Status:** build design, 2026-09-24. Supersedes v3 (single-machine) and the v4 draft.
**Authority:** `requirements.md` (what) → this file (architecture) → `contract.md` v2.1 (build spec,
derived from this file). On conflict the higher document wins. v3-era files (`topology.md`,
`build-server-client.md`) move to `docs/archive/`; review notes stay as history.
**Owner decisions** are listed in §22 and are fixed inputs.
**Tags:** `[V]` measured · `[K]` known behaviour, verify on the machine · `[A]` assumption/tunable.
**Public copy:** hostnames are aliases (`llm-gateway`, `gpu-host`, `backup-host`, `<tailnet>`); real
values live only in the private `config.yaml`.

---

## 1. Purpose and acceptance

lisa records the owner's digital life, summarises each day, keeps a compounding memory he can query by
topic, and — only with his approval — acts for him. It is one system used from MBP#1, MBP#2 and Win11.

**Acceptance case (the v1 milestone).** A doctor's email about a body check is (1) collected,
(2) understood as a grounded request/commitment, (3) surfaced in the morning brief with the quoted source,
(4) approved from any machine, (5) turned into a real Apple Reminder that links back to lisa, and
(6) receipted. Asking in English "when is my body check due" finds the Chinese email.

## 2. Principles

1. **One writer.** lisa-core owns all state. Nodes read local sources, ship events, execute actions.
2. **Dumb collection, smart digest.** No LLM on nodes. Nodes normalise, redact and ship.
3. **The LLM proposes, code decides.** Every LLM output is validated (sources exist, quotes are verbatim,
   dates parse); whether an action is proposed is decided by deterministic rules.
4. **Everything is grounded.** Each fact, digest line and action links to the events it came from.
5. **Derived layers are caches.** Snapshots, digests and rendered markdown can be regenerated. The
   database is the only source of truth; markdown is a view.
6. **Honest coverage.** Every digest says which machines and sources were complete, partial or missing.
7. **Forget means delete** — through every layer, with tombstones that stop re-collection.
8. **One-person operations.** One config file; errors explained in plain Chinese with the fix; no routine
   manual maintenance. **Prefer the simpler setup** where a choice exists (owner, round 3).

## 3. Deployment

### 3.1 Two stages, one codebase

| | **Stage 1 — now** | **Target — when the mini arrives** |
|---|---|---|
| Core + Postgres | MBP#2 (Intel, macOS 13.7.8) | M1 mini |
| Apple-account sources (iMessage, Reminders, Calendar, Contacts, Photos, Notes) | node on MBP#2 | node on the mini |
| Machine-local sources (files, Edge, Office MRU) | node on MBP#2; MBP#1 node from P4 | nodes on MBP#2 and MBP#1 |
| Action executor (Reminders/Calendar) | MBP#2 | mini |
| Clients | browser on all three; CLI on the Macs | same |

The code is identical across the two stages; only `config.yaml` differs (`profile: stage1-mbp2` /
`target-mini`). Postgres is installed the same way on both (Postgres.app, no Homebrew). The move is a
planned cutover (§20), not a redesign.

### 3.2 Source ownership

Each source has **exactly one owner node** in config. Apple-account data is synced by iCloud to several
Macs, so if each Mac collected it the same message would arrive twice under different local row IDs.
With a single owner per source, cross-machine duplicate collection cannot happen. Event IDs use
iCloud-global identifiers wherever they exist (§5.2), so changing an owner later is a config change:
the new owner's events land on the same rows.

### 3.3 What stage 1 costs (accepted)

MBP#2 is a laptop acting as a server. When it sleeps, the API, the database and collection sleep with it.
Nothing is lost: readers resume from cursors, and the day state machine catches up on wake. The morning
brief may be late, and MBP#1/Win11 cannot reach lisa while MBP#2 sleeps. Coverage and health say so
explicitly. `sleep 0 / disksleep 0` is already set `[V]`. Because core and executor share a machine,
"target machine offline" cannot occur in stage 1.

### 3.4 Network and identity

- **Transport.** All traffic runs over the owner's tailnet. Core listens on `127.0.0.1:8443`, and
  `tailscale serve` publishes it as HTTPS on the tailnet (`[K]`; fallback: HTTP over the tailnet,
  which WireGuard already encrypts). Postgres listens only on a unix socket.
- **Tokens.** Every device has its own token with one role (`node`, `client`, `readonly`, `ops`).
  Tokens can be revoked individually, and only their hashes are stored.
- **Browser sessions.** Browsers pair once with a one-time code and then hold an HttpOnly session cookie.
- **Other agents.** They have no access by default. A scoped `readonly` token can be issued if ever
  needed, and every use is audited.

## 4. Data flow

```
 node (Apple owner) ─┐  imessage · reminders · calendar · contacts · photos(+OCR) · notes
 node (each Mac)   ──┤  fs · edge · office_mru
 core: IMAP        ──┤  mail (+ attachments → converter → document events)
 web UI "告诉 lisa" ─┤  owner_note
                     ▼  outbox → POST /v1/ingest (batched, idempotent, watermarked)
               raw.events / raw.object_state / raw.blobs(transient)
                     │
        day state machine (watermarks) ──► snapshot (deterministic)
                     │
      NIGHT lane: photo describe → unitize → map (per thread) → validate → reduce → propose → dream
      DAY lane (overflow only): leftovers · dirty days · missed nights — yields to interactive load
                     │
     memory: facts · entities · commitments · pages · embeddings
                     │
   brief · web UI · notifications ◄── ops.actions ──► executor on target node (EventKit)
                                                           │
                                       receipt + ops.lisa_objects (self-observation guard)
```

## 5. Record layer

### 5.1 lisa-node

`LisaNode.app` is Swift, a universal binary targeting macOS 13, and uses system frameworks only. It runs
as a user LaunchAgent inside the owner's login session, because EventKit, Contacts, Photos and TCC grants
require that session. It is built from these parts:

- **Readers:** one per source, each reading incrementally from a cursor and emitting a common event format.
- **Filters:** node-side redaction and exclusion (§5.4).
- **Outbox:** a local queue. Read, write to outbox, ship; delete and advance the cursor only after the
  core acknowledges.
- **Shipper:** batched sends with retries.
- **Heartbeat:** health, watermarks, timezone and version, every 5 minutes.
- **Executor:** long-polls the core for actions and performs them through EventKit.
- **Notifier:** local macOS notifications.

**What a node may keep:** cursors, object IDs and hashes (for change detection), and unacknowledged outbox
events. Nothing else of personal content.

**Signing and permissions.** The binary is signed with a stable self-signed certificate so that the Full
Disk Access grant survives rebuilds `[K]`. On start, the node runs a self-test for each source and
reports any failure in plain Chinese.

**v3 lock.** The node refuses to collect while any v3 LaunchAgent is loaded (§19).

### 5.2 Event identity

`(source, event_uid)` is unique. UIDs use global identifiers where they exist:

| Source | UID basis |
|---|---|
| iMessage | `message.guid` |
| Mail | `Message-ID` |
| Reminders, Calendar | `calendarItemExternalIdentifier` |
| Photos | `PHCloudIdentifier` `[K]` |
| Machine-local sources | include the node name |

`CNContact.identifier` is device-local, so people are keyed by normalised phone (E.164) and email. The
contact ID is only a cache that the new owner re-seeds after a move.

Raw events are immutable except for two cases: `seen_on` gains nodes, and a higher `parser_version`
may re-normalise body and payload.

### 5.3 Go-live without history (owner decision)

lisa records from go-live onward only; nothing earlier is imported or backfilled. On its first run each
reader stores a **baseline** cursor and emits no events. Current *state* is different, because some of it
is needed from day one:

- **Open reminders** are loaded so the reminder dedup rule (P4) works immediately.
- **Upcoming calendar items** are loaded for the agenda.
- **Contacts** are loaded to resolve people.

None of this is treated as history. The brain starts empty. Two things help it fill: people are resolved
from Contacts, and a "告诉 lisa" box lets the owner type facts (trust `owner`) that go through the
normal pipeline.

### 5.4 Node-side privacy filters (before anything leaves the machine)

Filters run in this order:

1. **Exclusion rules.** Excluded chats, people, mail domains, URL domains and paths (always including
   `~/lisa`, `~/lisa-core`, `~/Library`, `.git`, `node_modules`, virtualenvs, caches) are dropped and
   leave no trace.
2. **Tombstones.** Events matching a tombstone are dropped.
3. **One-time codes.** Verification codes are redacted.
4. **Size cap.** Oversized bodies are truncated.

Photo text is recognised on the node (Vision OCR) and redacted there too, so an unredacted code in a
screenshot never leaves the Mac.

### 5.5 Self-observation guard

Objects that lisa creates (for example a reminder) are registered in `ops.lisa_objects`. When they are
re-collected they are marked `origin='lisa'`. They are shown in snapshots but are never extracted into
facts or actions again.

### 5.6 Source health

Each (node, source) pair has one state: `ok`, `degraded`, `blocked`, `error`, `stale` or `disabled`.
`stale` means a source that normally produces events every day has been silent for N days. It catches
sources that break without reporting an error.

Health appears at the end of the brief, on the health page, and as a notification when it turns red.

## 6. Time model

- **The owner-day.** `day = date(local(occurred_at) − 04:00)` in the owner's timezone, which comes
  from a timezone timeline in config. Only this formula is used anywhere.
- **Watermarks.** Each (node, source) reports a watermark: "everything up to T has been shipped and
  acknowledged".
- **Day states.** `open → closing → ready → digesting → digested`, and later `→ dirty → digesting`
  again. A day becomes `ready` when every required watermark has passed the end of the day, or at the
  06:30 deadline. After the deadline the digest runs anyway and the coverage section names what was
  missing. Late events mark a digested day `dirty`, and only the affected units are re-mapped.

## 7. Storage

Postgres 16 with `pg_bigm` (Chinese and English substring search, including two-character words such
as 体检) and `pgvector` (semantic and cross-language search). It is installed from **Postgres.app** on
both stages. `pg_bigm` is compiled against that install's `pg_config`, and core checks at startup that
both extensions are present.

| Schema | Holds | Can be rebuilt? |
|---|---|---|
| `raw` | events, object state, transient blobs | no (sources expire) |
| `memory` | days, units, digests, entities, facts, pages, embeddings | mostly (LLM-derived, versioned) |
| `ops` | nodes, tokens, health, actions, receipts, tombstones, jobs, audit | no |

DuckDB is **not** used in v1; a second persistent store would be one more place forgetting must reach.

## 8. Inference

### 8.1 Services (all on the owner's own hosts)

| Service | Used for | Access |
|---|---|---|
| Gateway (`auto`, one lisa key, fixed parameters) | map, reduce, photo describe, dream, ask | chat completions |
| Embedding `:8013` (Qwen3-Embedding-0.6B, dim 1024) | vectors for search | direct |
| Reranker `:8014` (Qwen3-Reranker-0.6B) | reranking search results | direct |
| Small model `:8017` | **only** unit triage, JSON repair, file grouping | direct |
| File→markdown converter | mail attachments; later, selected files | direct; must retain nothing `[K]` |

The fleet verified that every gateway upstream is on localhost, so the "local only" rule holds by
construction `[V]`.

### 8.2 Gateway rules

- **Request shape.** lisa sends only `model`, `messages` and `stream`, because the gateway ignores every
  other parameter `[V]`.
- **Limits.** Budgets and ask limits are enforced by lisa from the reported `usage`, not by request
  parameters.
- **Parsing.** lisa parses only `message.content`, ignores reasoning content, and strips any `<think>`
  blocks.
- **Model changes.** Every output records `response.model`. A change of model triggers a notice
  (quality, not privacy).

### 8.3 Privacy gate

Real personal content goes to **no** inference service until two conditions hold:

1. The owner has confirmed that the gateway's body dump is switched off (the global dump-off decision).
2. The latest canary for every enabled service has passed. The canary is a unique string that lisa
   sends and a fleet script then searches for in dumps and logs.

Synthetic calls (tests, throughput measurement, canary) are always allowed. **Streaming** may be
enabled only while the gate is open, and it is off by default. If a canary is ever found, the gate
closes immediately and all real-data calls stop.

### 8.4 Throughput and lanes

Measured: about **44 tok/s idle** and **24 tok/s under real load**. The gateway is nearly idle from
02:00 to 06:00 `[V]`. A normal day (40–60 units × ~1k output tokens, plus about 10 photos) is 25–50
minutes of generation, so the bulk runs at night:

- **Night lane.** Starts when a day becomes `ready` (normally about 05:00) and runs until 07:45.
  It does photo description, map, reduce, propose and dream.
- **Day lane.** Overflow only: leftovers from the night, dirty days, and nights missed because the
  laptop slept. It runs with concurrency 1 and yields when the gateway is busy (using a busy probe if the
  fleet adds one, otherwise observed latency).

Budgets are measured in completion tokens, with a separate line per kind (map, photo, reduce, dream).

## 9. Snapshot

A deterministic rendering of the day: events grouped by source, the day's agenda and the coverage
section. It is JSON for the pipeline and markdown for reading, carries a `snapshot_hash`, and can be
regenerated at any time.

## 10. Digest

### 10.1 Units

A day is split into units, and each unit is mapped separately:

| Source | One unit per |
|---|---|
| iMessage | chat per day |
| Mail | thread (attachments included) |
| Web | browsing session |
| Files | file cluster |
| Calendar and Reminders | the day's changes |
| Notes, owner notes | note |
| Photos | the day's photos |

### 10.2 Map

The model extracts facts from one unit at a time. Its input is the unit's events (tagged by trust
level) plus short profiles of the related entities and their **open commitments**. The open commitments
let it recognise that something has been resolved: when the clinic's confirmation email arrives, the
body-check commitment closes and the pending proposal is withdrawn. Map outputs are cached by input hash,
so re-digesting a day only re-maps the units that changed.

### 10.3 Validation (in code)

A map output must pass all of these checks:

- It is valid JSON that matches the schema.
- Every reference points to an event inside the unit.
- Every quote, after normalisation, is a substring of the source text (for photos, of the OCR text only).
- Enums and dates are valid.
- Trust is computed by code, never declared by the model.
- A photo-derived due date is re-parsed from the quoted OCR text by code and must match.

A fact that fails is dropped on its own; the rest of the unit's output is kept.

### 10.4 Reduce and propose

**Reduce.** Writes the daily summary from validated facts only, cited as `[F3]`. The coverage section is
generated by code.

**Propose.** Deterministic rules decide which suggested actions become proposals:

- The due date has not passed.
- The fact is no more than 7 days old.
- No action with the same idempotency key has ever existed; a denied one is never proposed again.
- No similar open reminder already exists.
- The fact is not based only on web or inferred sources.
- A nightly cap applies.

## 11. Photos and documents

### 11.1 Photos (fewer than 10 a day)

The pipeline has four steps:

1. **On the node.** PhotoKit finds newly added photos. The node makes a 1280 px derivative, runs
   Vision OCR (zh-Hans and en-US), redacts one-time codes in the OCR text, and ships the event (with the
   OCR text as its body) followed by the image.
2. **On the core.** In the night lane, the 27B model sees the image plus the OCR text and returns only
   `kind` and a two-sentence description. It does not transcribe text.
3. **Grounding.** Quotes must come from confident OCR spans, and a due date must be parsed from the
   quoted text by code. A fact supported only by the description is `inferred` and cannot produce an
   action.
4. **Visual confirmation.** Any action whose sources are all photos keeps its image while pending. The
   approval card shows the image with the quoted span highlighted, and Approve is disabled until the
   owner ticks "我已核对图片中的日期和时间".

**Image lifecycle.** Images are deleted after description, or when the related action leaves the pending
states, and after 10 days at most. They are **excluded from backups**. Hidden photos, a `lisa-skip`
album and videos are not read. Screenshots are **off by default** (opt-in). Location is rounded to about
100 m and never reverse-geocoded. People are described but never identified.

### 11.2 Documents

- **Mail attachments** (≤20 MB, PDF/Office/images) are fetched by core, converted to markdown by the
  fleet converter, and stored as `document` child events of the mail. The original bytes are discarded
  (P3).
- **Selected folders** (`content_roots`) follow in P4, using the same path.

The converter must keep no copy, cache or index `[K]`.

## 12. Brain

- **Entities** have ULID ids. The slug is only a display name. Kinds: `self`, `person`, `org`,
  `project`, `place`, `topic`. Domains such as health or work are tags, so an entity can belong to
  several.
- **Resolution** is deterministic first (phone, email and contact lookup), then by the map's references.
  LLM fuzzy matching can only *propose* a merge (the `merge_entities` action).
- **Facts** carry a trust level and a claimant. Commitments and requests have a lifecycle:
  `open`, `done`, `moot`, `snoozed`.
- **Pages** are recompiled from the entity's facts, not patched incrementally, to avoid drift.
  Third-party claims are written as claims ("据…称"). The owner's `pinned` section is never touched by
  the LLM. The last 20 versions are kept for rollback, with no git (git history would conflict with
  forgetting).
- **Dream** runs nightly after the digest:
  1. Recompile dirty pages.
  2. Propose merges.
  3. Spot-check 5% of new citations.
  4. Ask "done?" about overdue commitments in the brief.
  5. Embed new items.

## 13. Search and ask

**Search** is hybrid:

1. `pg_bigm` handles literal matching; `pgvector` handles semantic and cross-language matching.
2. The two result lists are merged with RRF, and the top 50 are reranked by `:8014` down to the top 10.
3. Queries go out in both Chinese and English, so "body check" finds "体检" even without vectors.

**Ask** routes by question type. Exact questions ("when did I last…", "how many…") go to structured
queries on events. Synthesis questions ("prepare me for…") read entity pages first, then search.

Ask's tools are read-only, plus `propose_action`, which can only create pending proposals. Limits: 8
rounds and 40k tokens, counted by lisa. Answers cite sources as clickable links. Until streaming is
allowed, core calls the model non-streaming but still streams tool progress and citations to the browser.

## 14. Actions

**Verbs (v1).**

- `create_reminder`, `complete_reminder` and `create_calendar_event` run on the owner node of the target
  source through EventKit.
- `merge_entities` runs on the server.
- Nothing is sent to other people in v1.

**Lifecycle.**

- The main path is `awaiting_review → approved → dispatched → succeeded/failed`.
- Other states: `denied`; `expired`, which goes to a "missed" list and can be revived; `superseded`,
  when the commitment is resolved; `stale`, when not executed in time, needing reconfirmation;
  `outcome_unknown`, when the lease runs out without a report.
- Every transition is written to an append-only receipt log.

**Keys and expiry.**

- The idempotency key is built from stable facts only: verb + fact type + the anchor event's global
  UID + due date. It never includes LLM wording.
- Review window for digest proposals: 72 hours or the due date, whichever is earlier. Review window for
  ask proposals: 30 minutes.
- Approval is a compare-and-set against the arguments hash the owner saw.

**Execution.** Before executing, the executor searches for a marker: the reminder's URL
`…/a/<action_id>` and a `lisa:<action_id>` line in its notes. If the marker is found, it reports success
without acting again. If the lease expires with no report, a read-only reconcile searches for the marker
and turns `outcome_unknown` into a definite `succeeded`, or `failed` (safe to retry).

## 15. Security and privacy

| Threat | Mitigation |
|---|---|
| Prompt injection via mail, messages, web pages or photos | Source text is wrapped in `<source trust=…>` blocks and treated as data; trust comes from the source, not the model; a fixed action allowlist with schema-checked args; approval cards always show the source; nothing web-only or inferred can produce an action; no outbound-send tools in v1 |
| Inference hosts logging lisa content | Privacy gate plus weekly canary (§8.3) |
| Leaked device token | Per-device, role-scoped, revocable tokens; node tokens cannot read data |
| Other fleet agents | No access by default; Postgres on a unix socket; directories mode 0700 |
| Logs | IDs and counts only, never content |
| Backups | Encrypted restic to the owner's own host, 30-day retention, images excluded |
| Stolen mini (target) | FileVault plus a UPS |

## 16. Forget

**Scope.** One event, a thread, a person, a time range (optionally for one source), or items picked
from search results. All are in v1.

**Flow.** Forget is always two steps. A preview shows what each layer will lose; the commit uses the
preview token so that exactly the previewed scope is deleted. One transaction then:

1. Writes tombstones that store only HMACs of the selectors, never content.
2. Deletes the matching events, states, images, units, vectors and eval references.
3. Deletes facts that have no remaining source.
4. Recompiles the affected pages and deletes their old versions.
5. Replaces the arguments of affected actions with "[已遗忘]", keeping state and times.
6. Marks the affected days dirty and deletes their digests so they are regenerated.

**Cleanup.** Physical space is reclaimed within 24 hours. Nodes receive the new tombstones, purge their
outboxes and never ship matching items again.

**Receipt.** It states:

- when the data leaves the backups (after 30 days);
- how many lisa-created reminders the owner may want to delete himself;
- that the originals in Mail, Messages and Photos are untouched.

## 17. Clients and delivery

**Web UI.** Server-rendered HTML with a little plain JS, served by core. It is the main client on all
three machines. Pages: today (brief), a day, pending, receipts, ask, entities, search, health, forget,
告诉 lisa, settings.

**Morning brief at 08:00.** It contains:

- yesterday's highlights;
- pending proposals;
- missed proposals (each shown once);
- due commitments and "done?" questions;
- health;
- "still processing" if the night lane has not finished.

**Notifications.** Macs get a local notification with an "open" button only, because approval needs the
source in view. Win11 has no push in v1, since web push would depend on a third-party service.

**CLI.** A single-file, stdlib-only Python 3.9 helper for the Macs.

## 18. Operations

- **Processes.** Everything runs as user LaunchAgents. No sudo is needed except the one-time `pmset` on
  the mini.
- **Scheduling.** Core runs its own in-process scheduler, so there are no calendar-interval agents and a
  missed schedule is simply caught up after wake.
- **Backups.** Nightly `pg_dump` (images excluded) plus config plus a secrets bundle, sent with restic
  to `backup-host` and kept 30 days. A monthly restore drill is run automatically.
- **Upgrades.** `make deploy` from a tag; database migrations run automatically; nodes report their
  version.
- **Jobs.** Every job is recorded, and failures appear on the health page in Chinese.

## 19. Retiring v3

MBP#2 currently runs v3: a resident collector and a 04:00 digest agent, with data under `~/lisa/`.
Retirement goes in this order:

1. Unload both v3 agents and park their plists.
2. Verify that no v3 process is running.
3. Start the v4 node; it refuses to run if v3 is still loaded.
4. After acceptance test B passes, and **with the owner's explicit confirmation**, delete `~/lisa/`
   entirely. It is outside forget's reach and no history is wanted.
5. The owner removes the old binary's Full Disk Access entry.

v4 lives in a new directory, `~/lisa-core/`. TCC permissions do not carry over: the new node must be
granted again by the owner at the machine.

## 20. Migration to the mini

| Step | What happens |
|---|---|
| M0 Prepare | Install Postgres.app with the extensions, the uv Python, tailscale, FileVault and a UPS on the mini. The owner grants permissions and runs the iCloud checks. **The fleet issues the mini its own gateway key** (keys never move). |
| M1 Shadow | The mini's node reads the Apple sources in shadow mode for 24 hours or more. Only UIDs are compared, and a match of at least 99% is required to continue. |
| M2 Freeze | MBP#2's core enters maintenance; nodes buffer in their outboxes. Then `pg_dump`, and a secrets export (tombstone HMAC key, IMAP, restic only). |
| M3 Restore | Restore on the mini, switch the config to `target-mini` (Apple sources owned by the mini), re-render everything, start core. |
| M4 Cutover | The old core answers heartbeats with `core_moved_to`; nodes drain to the mini; the mini's node moves from shadow to owner. MBP#2's node keeps only its machine-local sources. |
| M5 Verify | All health checks green; acceptance B passes on the mini; a forget test passes. |
| M6 Decommission | Drop MBP#2's database and PGDATA, remove the core Keychain items and agents, revoke MBP#2's gateway key. This is required, not optional: a leftover copy is outside forget's reach. |

## 21. Phases

| Phase | Delivers | Done when |
|---|---|---|
| **P0 Foundation (MBP#2)** | v3 agents retired (data kept for now); Postgres.app + extensions; core skeleton, tokens and pairing; health page; backups; canary and privacy gate; signing certificate | The health page opens in browsers on all three machines, and the gate opens after D3 is confirmed and the canary passes |
| **P1 Collection** | Node sources: fs, edge, office_mru, imessage, reminders, calendar, contacts; core IMAP; filters, outbox, watermarks, baseline, tombstones | Events flow; an overnight sleep loses nothing; revoking a permission shows on the health page |
| **P2 Acceptance slice** | Snapshot; map/validate/reduce for mail and iMessage; facts; propose; `create_reminder`; executor; web approval; receipts; notification | Acceptance test B passes end to end |
| **P3 Memory, forget, photos** | Entities, pages, dream, hybrid search with rerank, ask, full forget, photos with OCR, mail attachments via converter; "告诉 lisa" | "body check" finds 体检; forget verified across all layers |
| **P4 Completion** | Notes, SFL, MBP#1 node, calendar and complete actions, `content_roots`, triage via `:8017` | 30 days of stable digests |
| **P5 Migration** | §20, whenever the mini arrives (possible any time after P2) | M5 passes |
| **P6 Optional** | Windows collector | If ever wanted |

## 22. Decisions

**Fixed (owner / fleet):**

- MBP#2 hosts stage 1; the mini is the target.
- Apple sources are owned by the Apple-owner node (MBP#2, then the mini).
- Postgres.app, no Homebrew; Python via uv.
- No history import or backfill.
- One gateway key, `auto`, fixed parameters.
- Embeddings direct to `:8013`; body dump off globally.
- The 27B describes photos (fewer than 10 a day).
- Bulk processing at night; the day lane is overflow only.
- No DuckDB.
- Screenshots opt-in.
- Forget covers all scopes.
- 04:00 rollover; 08:00 brief with Mac notifications.

**Open:**

- **D-11:** which mail accounts to collect.
- **Fleet:** a gateway busy signal, so lisa can yield on something better than latency.
- **Later:** a Windows collector.

## 23. Verification

Items that still need measurement are listed in `contract.md` §24. Each is marked **O** (owner physically
at the machine) or **F** (fleet agent over SSH), so P0 can be scheduled around the owner's presence.

## 24. Changes since v4

| Area | v4 | v4.1 |
|---|---|---|
| Stage 1 host | the mini | MBP#2 now; the mini later via runbook |
| History | 90-day digests + 2 years of facts | none; baseline only |
| LLM | per-role aliases, never `auto` | `auto`, one key, fixed parameters; privacy gate |
| Night vs day | continuous map proposed | night bulk; day lane is overflow only (measured) |
| Photos | metadata only | node OCR + 27B description + code-parsed dates + visual confirmation |
| Search | bigm + vector | + rerank `:8014`; bilingual query expansion |
| Postgres | Homebrew | Postgres.app on both stages |
| v3 | import its data | retire it in order; delete its data after go-live |
| Secrets on move | all | HMAC, IMAP and restic only; new gateway key for the mini |
| Backups | full dump | images excluded |
