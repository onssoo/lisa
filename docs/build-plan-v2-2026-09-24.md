# lisa v4.1 build — execution plan, v2

**Supersedes:** `build-plan-v41-2026-09-24.md` (v1, commit `a19b0de`).
**Inputs folded in:** the fleet round-1 review (`build-plan-review-2026-09-24.md`), the
reviewer's round-2 response + fleet verification (`build-plan-review-round2-2026-09-24.md`),
the change request (`build-plan-change-request-2026-09-24.md`), and the owner decisions
(`decisions.md`, 2026-09-24). Every change-request item A–G is applied — see the
application table. Nothing here overrides `design.md` v4.1 or `contract.md` v2.1.

## Context

Implement `lisa` v4.1 per `docs/design.md` (v4.1) and `docs/contract.md` (v2.1) — the
build spec. Ignore v3 entirely except the mandatory retirement order (contract §22). Stage 1
runs on stage1-host (MBP#2, macOS 13.7.8 Intel); the M1 mini migration (P5) is a runbook,
executed later when the mini arrives.

Authority order: requirements.md → design.md → contract.md. Where this plan pins a value the
contract left open, the pin is listed in the Decision log and must be honored.

**One-URL rule (B6):** every browser uses `public_url` — the tailnet HTTPS URL served by
`tailscale serve` — on **every** machine, including stage1-host itself. The loopback address
(`127.0.0.1:8443`) is for the node and the CLI only. The CSRF check requires
`Origin == public_url` (contract §9.3); opening loopback in a browser fails every
approve/decide POST.

**No real hostnames, addresses or account names in this document (B7).** Aliases:
`stage1-host` (MBP#2), `gpu-host` (the the GPU host), `llm-gateway` (the gateway service on gpu-host),
`backup-host` (the old mini; also hosts the fleet git host), `<tailnet>`. Real values live only in the
private `~/lisa-core/config.yaml` and in `ops/` deploy scripts.

## Verified environment facts (measured 2026-09-24 on stage1-host)

- v3 running: LaunchAgents `com.lisa.collector` (resident) + `com.lisa.digest` (04:00);
  data in `~/lisa/` (SQLite, snapshots, brain). v3 dev clone at `~/Projects/lisa`.
- Postgres: **absent** (no server, no client, no Postgres.app). Homebrew absent (owner declines).
- Python 3.12.13 via `uv` 0.11.6 (`~/.local/bin/uv`). Ports 5432, 8443 free. `sleep 0 / disksleep 0`.
- Swift 5.8.1, CommandLineTools, SDK 13.3. **arm64 cross-compile verified**
  (`swiftc -target arm64-apple-macos13.0` produced a valid arm64 Mach-O) → universal binary is
  buildable on this Intel machine.
- **P0.1 deviation (2026-09-24):** the node was built on the existing toolchain
  (Swift 5.8.1, CLT 14.3.1, SDK 13.3) — not CLT 15.x. The "CLT 15.x blocker" was a
  misdiagnosis: the only macOS-14 API in use (`Process.run(_:arguments:stdout:stderr:)`)
  was replaced with the property-based API, and `ISO8601DateFormatter` (not
  `ISO8601Formatter`) is available since macOS 10.15. `swiftc -typecheck` is clean on
  all of `Sources/LisaNode/**/*.swift` and `make node` produces a universal
  (x86_64 + arm64) binary. This deviates from contract §1's "Swift 5.9+" one line;
  the node uses no 5.9-only features.
- **`softwareupdate --list` (owner-approved read-only check, 2026-09-24): "No new software
  available"** → CLT 15.x cannot come via softwareupdate; the path is the developer.apple.com
  Command Line Tools download (owner's Apple ID) or App Store Xcode 15 (B3).
- `tailscale` installed on stage1-host (tailnet identity in `config.yaml` only);
  `tailscale serve` not yet configured.
- Inference (all on gpu-host, reachable):
  - llm-gateway `:9000/v1` — model `auto` only; `/stats` exists (no clean busy signal → use
    observed latency for day-lane yield, per contract fallback; E3).
  - embed `:8013` — `POST /v1/embeddings`, OpenAI-shaped, dim 1024 (verified).
  - rerank `:8014` — **API shape verified**: `POST /v1/rerank`
    `{"model":"qwen3-reranker-0.6b-q8_0.gguf","query":…,"documents":[…]}` →
    `{"results":[{"index":n,"relevance_score":f}]}`.
  - small `:8017` — model id `minicpm5-2b`, OpenAI-compatible chat.
- the fleet git host (on backup-host, SSH) reachable from stage1-host; repo `lisa` main branch holds
  v3 code (`src/collector`, `src/digest`, `src/agent`, `launchd/com.lisa.{collector,digest}.plist`,
  `scripts/install.sh`, old `Makefile`).
- Keychain: no `lisa.*` items exist yet.
- backup-host: SSH works, `/` has 301 GB free, restic **not** installed (install during P0.5,
  owner approval pending).
- Postgres.app download: **via the fleet mirror** (direct GitHub downloads are unreliable on
  this network — fleet review small item).

## Decision log

### Owner decisions (recorded in `decisions.md`)

| # | Item | Decision (2026-09-24) |
|---|---|---|
| D-11 | Mail accounts | **The owner's icloud.com address, single account** — the owner's answer to an open requirement, not a coder default. `sources.mail.accounts: [{name: icloud, host: "imap.mail.me.com", user: <owner>, keychain: "lisa.imap.icloud"}]`; the address value lives only in `config.yaml`. App-specific password created by the owner in the one session. |
| v3 | Stop v3 (R1/R2) | **Approved.** Executed in the P1.0 install session. |
| session | One-session plan | **Agreed:** trust signing cert → `tailscale serve` → iCloud app password → same sitting: R1/R2, install the P1 node, grant permissions. |
| clt | `softwareupdate --list` | **Agreed and executed:** no new software available (see env facts). |
| gpu | gpu-host changes (gateway dump off; issue lisa its own gateway key) | **Approval pending** — the owner's reply named only "stop v3" from the cross-machine batch. Confirm before P0.4 (canary) and before key issuance. |
| restic | Install restic on backup-host | **Approval pending** — same as above. Confirm before P0.5 (backup). |

### Coder pins (contract-open values; not owner decisions)

| # | Item | Pin |
|---|---|---|
| backup | restic target | `sftp:owner@backup-host:/srv/restic/lisa`. restic installed on backup-host in P0.5. |
| convert | file→markdown converter | `inference.convert_url: null` → `documents.mail_attachments: false` stays off; the attachment code path exists but is inert until the owner points `convert_url` at a verified no-retention converter (fleet-side verification, contract §24-15, is the precondition). |
| clt | Swift toolchain | CLT 15.x (Swift 5.9+, SDK 14+) installed **before P0.3** via the developer.apple.com Command Line Tools download (softwareupdate offers nothing — measured). After install: verify `swiftc --version` ≥ 5.9, `xcrun --show-sdk-version` ≥ 14.0, re-verify the arm64 cross-compile, write the result back into contract §1. |
| key | `lisa.gateway` key | A dedicated lisa key issued by the fleet (keys never move). Store in Keychain as `lisa.gateway`. If the fleet cannot issue one, stop and ask the owner — never reuse another agent's key. |
| D3 | gateway body dump | Verify current dump state on gpu-host first (dump dir rolling ~2000 files was active per fleet notes); if still on, the fleet switches it off (owner approval pending), then the owner confirms via `POST /health/confirm-d3`. |
| dev | dev clone path | `~/lisa-v4` (clone of the the fleet git host repo). Runtime dirs per contract §4 (`~/lisa-core/`, `~/Applications/LisaNode.app`). |
| rerank | `:8014` shape | Verified (see above) — write it into contract §2 table and §19.2 as `[V]` in the P3 doc pass. |
| worker | where `lisa-core worker` runs | Same host as `serve` for now (C4). Placement is an **open design item**, decided with the storage question when the mini arrives (E2). |
| busy | gateway busy probe | `busy_probe_url` stays null; day lane yields on observed p50 latency > `backoff_latency_s` (contract's stated fallback). Revisit only if the fleet adds a probe (E3). |

## Change-request application (A–G)

| Item | Status | Where in v2 |
|---|---|---|
| A1 R1/R2 → P1 install day | Applied | P1.0; removed from P0 |
| A2 contract §22 order inside the session | Applied | P1.0 (R1 → R2 → R3 → R4) |
| A3 P0 acceptance replacement | Applied | P0 acceptance: `V3_ACTIVE` guard test with a fake v3 label |
| B1 absolute socket path | Applied | P0.1 |
| B2 Info.plist usage strings | Applied | P0.3 |
| B3 CLT 15.x before the node build | Applied | P0.3 prerequisite + decision log (softwareupdate measured: nothing available → developer.apple.com download) |
| B4 Postgres.app GUI never launched; pg_bigm arm64 rebuild | Applied | P0.1 + P5 M3 |
| B5 restic password offline copy | Applied | P0.5 owner action |
| B6 one-URL rule | Applied | header rule + P0 verification + P2 acceptance B step 4 (iPhone on the tailnet) |
| B7 aliases in prose | Applied | throughout |
| C1 mail collection from P0.2 | Applied | P0.2 (activates when the app password is in Keychain; gate-independent) |
| C2 canary retry / run-on-wake / `CANARY_OVERDUE` | Applied | P0.4 |
| C3 `tailscale serve` hard prerequisite | Applied | P0.4 (before the first canary and any gate-open acceptance) |
| C4 worker/serve split | Applied | P0.2 (two entry points, two LaunchAgents, same host) |
| C5 catch-up rule per job | Applied | P0.2 scheduler (brief, backup, canary, maintenance) |
| D1 `attributedBody` cross-check < 1% | Applied | P1 verification |
| D2 grants survive rebuild + re-sign | Applied | P1 verification |
| D3 PhotoKit change tokens on macOS 13 | Applied | P3 verification |
| D4 EventKit external ID stability + marker round-trip | Applied | P2 verification |
| D5 Postgres.app verification set | Applied | P0.1 + P0 verification |
| E1 D-11 not assumed | Applied | owner decision (icloud.com, 2026-09-24) |
| E2 worker placement open | Applied | decision log (open design item) |
| E3 busy probe null | Applied | decision log |
| fleet-small: `uv pip sync` → explicit venv | Applied | Makefile |
| fleet-small: Postgres.app via fleet mirror | Applied | P0.1 |
| fleet-small: contract §24 verification items | Applied | D1–D5 rows above |

No items declined.

## Phase 0 — Repo setup (before P0)

1. `git clone gitea@backup-host:lisa.git ~/lisa-v4`.
2. Archive v3 per design §1: move `src/` → `docs/archive/v3-src/`, `launchd/` →
   `docs/archive/v3-launchd/`, `scripts/install.sh` → `docs/archive/v3-install.sh`.
   Keep the review notes at `docs/` top level (design says reviews stay top level).
3. Create the v4 skeleton directories exactly per contract §3:
   `core/lisa_core/`, `core/migrations/`, `core/tests/`, `node/Sources/LisaNode/`,
   `node/Resources/`, `node/Tests/`, `cli/`, `ops/launchd/`.
4. Replace `Makefile` with v4 targets (see below). Extend `.gitignore` with
   `core/tests/fixtures/`, `node/build/`, `*.app/`.
5. Commit + push as `v4: repo layout, v3 archived`.

**Makefile (v4)** targets:
- `venv` — `uv venv ~/lisa-core/venv --python 3.12 && uv pip sync --python
  ~/lisa-core/venv/bin/python core/requirements.lock` (the venv must be explicit — otherwise
  uv resolves against the system interpreter; fleet small item).
- `node` — build universal binary: `swiftc -O -target x86_64-apple-macos13.0 …` and
  `-target arm64-apple-macos13.0` over `node/Sources/LisaNode/**/*.swift`,
  `lipo -create` → `node/build/LisaNode`, `codesign --sign "Lisa Code Signing"`,
  bundle into `node/build/LisaNode.app` (Info.plist from `node/Resources/`).
- `test` — `cd core && ~/lisa-core/venv/bin/pytest tests/ -x -q`
- `deploy` — `git fetch --tags && git archive <tag>` → `~/lisa-core/app/`, then
  `launchctl kickstart -k gui/$UID/com.lisa.v4.core` **and** `gui/$UID/com.lisa.v4.worker`.
- `install-node` — copy app to `~/Applications/`, install `ops/launchd/com.lisa.v4.node.plist`.

---

## Phase P0 — Foundation (stage1-host)

### P0.1 Postgres
- Download Postgres.app 16 **via the fleet mirror**; install to `/Applications/Postgres.app`.
  **The app is a source of binaries only — never launch the GUI (B4):** it would start its
  own server on its own cluster, and the LaunchAgent must own the only server. Disable
  auto-update (app setting) — pg_bigm must be rebuilt after any PG update.
- `ops/build-pg-bigm.sh`: download pg_bigm source,
  `make PG_CONFIG=/Applications/Postgres.app/Contents/Versions/16/bin/pg_config`,
  `make install`. Run it.
- Create `~/lisa-core/{pg,run}` (0700). Init cluster with `initdb`, apply
  `postgresql.conf` overrides exactly per contract §4: `listen_addresses=''`,
  `unix_socket_directories = '<abs>/lisa-core/run'` — **an absolute path, expanded at deploy
  time; Postgres never sees `~` (B1)** — plus the log settings. Create db `lisa` + user.
- Install `ops/launchd/com.lisa.v4.postgres.plist` (KeepAlive, RunAtLoad, absolute paths)
  and boot it.
- Verify: `psql -h ~/lisa-core/run -d lisa -c "select 1"`; both extensions load
  (`CREATE EXTENSION pg_bigm; CREATE EXTENSION vector;` in a scratch db).
- **D5 verification set:** version pinned (recorded in health), auto-update off, GUI never
  launched (no Postgres.app process running its own cluster), `pgvector` bundled, `pg_bigm`
  built against its `pg_config`.

### P0.2 Core skeleton
Files (all new, under `core/lisa_core/`):
- `migrations/0001_init.sql` — the full schema from contract §6, verbatim (all three schemas,
  all tables, the bigm indexes, the HNSW index, the `ON DELETE CASCADE` clauses).
  `db.py` applies migrations in a transaction, records applied versions in `ops.meta`.
- `config.py` — load `~/lisa-core/config.yaml` (contract §5 keys; expand `~`; on invalid key →
  exit with a Chinese message naming the key). Ship `config.example.yaml` in repo (aliases
  only); the real `~/lisa-core/config.yaml` (0600) is created at deploy time with the real
  values (llm-gateway/embed/rerank/small URLs, `public_url`, `pg_dsn`) — **`config.yaml` is
  the only place real hostnames/addresses live (B7)**.
- `auth.py` — Bearer tokens (sha256 lookup in `ops.tokens`), session cookies
  (`lisa_session`, HttpOnly, SameSite=Strict, Max-Age 90d, Secure when HTTPS), CSRF check
  (`Origin == public_url`) on state-changing browser POSTs, per-role route guards
  (node tokens can only hit node/executor endpoints — cannot read data).
- `app.py` — Starlette app, binds `127.0.0.1:8443`; maintenance middleware
  (503 `MAINTENANCE` on everything except heartbeat while `ops.meta.maintenance.on`);
  JSON error shape `{"error":{"code","message_zh","detail"}}`.
- `worker.py` + `__main__.py` — **two entry points (C4):** `serve` (API + web only) and
  `worker` (the scheduler loop: day state machine, night/day lanes, brief, backup, canary
  state, maintenance). Same codebase, same host for now; separate LaunchAgents
  (`ops/launchd/com.lisa.v4.core.plist`, `com.lisa.v4.worker.plist`). Where the worker runs
  is decided at migration (E2). CLI verbs: `serve`, `worker`, `pair --node --role`,
  `maintenance on|off`, `secrets export|import`, `render --all`, `redigest --from --to`
  (stub), `move-announce` (stub), `forget-verify` (stub), `selftest --throughput`.
- `api/` — P0 implements fully: `pair.py` (`GET/POST /pair`, `POST /logout`), `ops.py`
  (`POST /health/confirm-d3`, `POST /ops/canary-report`), `node.py`
  (`GET /node/config`, `POST /node/heartbeat`). Stubs returning 501 with a clear code for
  ingest/executor/client endpoints land in P1/P2 — the routes exist so the node can be built
  against them.
- `health.py` — health code templates from contract §18 (all codes, zh messages) **plus
  `CANARY_OVERDUE` (C2)**; `GET /health` aggregates nodes, sources, jobs, gate, canary,
  backup, restore drill.
- `llm.py` — gateway client per contract §19.1: request body exactly
  `{"model":"auto","messages":[…],"stream":<bool>}`; headers `Authorization: Bearer <lisa.gateway>`
  (from Keychain) + `X-Lisa-Client: core/<version>`; timeout/retry 30s→2min→10min→`GATEWAY_DOWN`;
  parse `choices[0].message.content` only, strip `…` blocks, extract first balanced `{…}`;
  record `response.model` + `usage` into `ops.jobs`; `MODEL_CHANGED` notice when model differs
  from `ops.meta.last_model`. Direct-service clients: embeddings (`POST {embed_url}/v1/embeddings`,
  ≤32 inputs/call), rerank (verified shape above), small model (OpenAI chat).
- `scheduler.py` (in the worker) — in-process 5-minute loop with per-day advisory lock; P0
  wires only: day state transitions (open→closing→ready per contract §10.2–10.3), brief build
  at 08:00 (P0 brief = coverage + health only), backup job at `backup.at`, canary state at
  Monday 05:30, maintenance VACUUM at 03:30. **Catch-up rule (C5), explicit per job — brief,
  backup, canary, maintenance:** if the slot was missed (host slept, lid closed), the job
  runs on the next wake; the scheduler records last-run per job so no scheduled task can be
  skipped by a closed lid without being picked up. Night/day lanes stubbed (no-op until P2).
- `sources/imap.py` — **wired in P0, not P1 (C1):** stdlib `imaplib` over SSL; INBOX +
  `\Sent`; live lane only: UIDs > `high_uid` every 5 min, batches of 25, one connection/
  account, never parallel; backoff 1→5→15→60 min on BYE/UNAVAILABLE/LIMIT/timeout;
  UIDVALIDITY change → `high_uid = UIDNEXT−1` (no rescan); body = text/plain else html→text;
  strip quoted replies (`>` lines, "On … wrote:", "在 … 写道："); ≤20k chars; `thread_key` =
  first References, else In-Reply-To, else own Message-ID; OTP redaction runs in core for
  mail. Collection never sends content to a model, so it does **not** depend on the privacy
  gate (contract §24 scopes the gate to inference services). It activates when
  `lisa.imap.icloud` exists in Keychain — the owner creates the app-specific password any
  time; in the one-session plan it is created during the session. Real mail accumulates
  while the pipeline is built → acceptance B is much easier.
- `web/` — Jinja2 templates + plain JS, no build step. P0 pages: **health** (full), **pair**
  (form), and shell pages (today/day/pending/ask/…) as minimal placeholders that render.
- `requirements.lock` — pinned: `psycopg[binary]==3.2.*`, `starlette`, `uvicorn`, `jinja2`,
  `pyyaml`, `cryptography` (contract §1 — nothing else).

### P0.3 Node skeleton (Swift)
**Prerequisite (owner action, B3):** install CLT 15.x (Swift 5.9+, SDK 14+) via the
developer.apple.com Command Line Tools download (owner's Apple ID; softwareupdate offers
nothing — measured 2026-09-24). App Store Xcode 15 is an acceptable alternative. Then verify
`swiftc --version` ≥ 5.9, `xcrun --show-sdk-version` ≥ 14.0, re-verify the arm64
cross-compile, and write the result back into contract §1. The SDK 13.3 that ships with the
current CLT cannot compile the `if #available(macOS 14, *)` branch the mini will need —
waiting for a compile error means rebuilding mid-migration.

Files under `node/Sources/LisaNode/`:
- `main.swift` — arg parsing: `run` (LaunchAgent mode), `--pair <code>`, `--selftest [source]`.
- `Config.swift` — fetch `GET /v1/node/config` (cached slice: sources with mode, filters, HMAC
  key, tombstones, `config_version`); no config file on the node.
- `Ids.swift` — event UID builders per source (contract §7.3 table); phone E.164 normalization
  with `default_phone_region`; email lowercasing; `me` for owner.
- `Time.swift` — Apple epoch / Chromium epoch / Unix conversions.
- `Outbox.swift` — `node.db` (SQLite WAL) schema per contract §8; batch ship with
  `cursor_after` advance only on core ACK; blob-after-event-ack rule; 50k-row/200 MB cap →
  `OUTBOX_FULL`.
- `Shipper.swift` — batched `POST /ingest` (≤500 events / 5 MB), retries, idempotent batch_id.
- `Heartbeat.swift` — every 5 min: health, watermarks, tz, version; handles `core_moved_to`.
- `Filters.swift` — exclusion rules (config `exclude_*` + defaults), tombstone cache (HMAC of
  `source:event_uid`, `thread_key`, identifiers, ranges), OTP redaction regex per contract §7.5,
  20k-char size cap.
- `V3Guard.swift` — on every start: `launchctl list` must not contain `com.lisa.collector` /
  `com.lisa.digest` and `pgrep -f lisa-collector` empty; else report `V3_ACTIVE` and refuse to
  collect.
- `SelfTest.swift` — per-source permission/readability probes, plain-Chinese failure output.
- `Notifier.swift` — UserNotifications; P0: one notification type (brief ready).
- `Executor.swift` — long-poll `POST /executor/claim` loop, 25 s; P0: claim + report plumbing
  only (no verbs until P2).
- `Resources/Info.plist` — bundle id `com.lisa.node`, min macOS 13.0, **and the permission
  strings (B2):** `NSRemindersUsageDescription`, `NSCalendarsUsageDescription`,
  `NSContactsUsageDescription`, `NSPhotoLibraryUsageDescription`; for macOS 14+:
  `NSRemindersFullAccessUsageDescription`, `NSCalendarsFullAccessUsageDescription`. Without
  them macOS terminates the process at the moment it asks for access, instead of prompting.
  (Also belongs in contract §3/§5 and §24-2 — reviewer doc pass.)
- `ops/launchd/com.lisa.v4.node.plist` — `LimitLoadToSessionType Aqua`, KeepAlive, absolute
  binary path.
- `ops/make-signing-cert.sh` — create self-signed "Lisa Code Signing" identity.
- `ops/install-node.sh` — build (Makefile `node`), install app + plist, boot.

Owner actions this phase — **the one session, in this order (G.5):**
1. Trust the "Lisa Code Signing" certificate in Keychain Access (contract §24-1).
2. `tailscale serve` HTTPS for :8443 on stage1-host + tailnet HTTPS certs (§24-4) — **hard
   prerequisite of P0.4 and of every acceptance test that needs the gate open (C3)**.
3. Create the iCloud app-specific password → Keychain `lisa.imap.icloud` (§24-5).
4. Same sitting, continue into P1.0: R1/R2, install the P1 node, grant permissions.

### P0.4 Canary + privacy gate
- **Prerequisite: `tailscale serve` is live (C3)** — the canary check runs on gpu-host and
  reports over the tailnet; core binds loopback.
- `ops/inference-canary-check.sh` — deployed **on gpu-host** (fleet side, SSH; owner approval
  pending for gpu-host changes): searches the gateway dump directory + the three direct
  services' log dirs for the canary nonce, reports via `POST /v1/ops/canary-report` with the
  `ops` token. Cron on gpu-host: Mondays 05:30, **retrying hourly until the report lands
  (C2)** — a sleeping stage1-host must not lose the slot.
- **`CANARY_OVERDUE` (C2):** the worker raises this health code (zh message naming the
  reason) when the last passing canary is older than the 8-day window. The gate never closes
  without a health message naming the reason.
- Gate logic in core (`ops.meta.privacy_gate`): closed by default; open only when
  `d3_confirmed` set **and** latest canary per enabled service passed within 8 days with
  `found=false`; `found=true` → close + `GATEWAY_POLICY`. While closed: night/day lanes,
  photo describe, conversion, embeddings, ask all refuse with `PRIVACY_GATE_CLOSED`;
  synthetic calls (tests, canary, `selftest --throughput`) always allowed.
- Run the first canary; verify gate opens.

### P0.5 Backup
- Install restic on backup-host (official static binary, or `apt` — backup-host is
  Ubuntu-based). **Owner approval pending.**
- **Owner action: keep an offline copy of the restic password (B5).** The secrets bundle is
  Fernet-encrypted with a key derived via scrypt from the restic password, and that bundle
  holds `lisa.hmac` (the tombstone key). Losing the password loses the backups **and** that
  key copy.
- `ops/backup.sh` per contract §21.1: `pg_dump -Fc --exclude-table-data=raw.blobs
  --exclude-table-data=ops.shadow_uids` → secrets bundle via `lisa-core secrets export
  --for-backup` (lisa.hmac + lisa.imap.*; never lisa.gateway) Fernet-encrypted with scrypt
  from the restic password → restic push (dump + config.yaml + bundle) →
  `forget --keep-daily 30 --prune` → delete staging.
- `ops/restore-drill.sh` — monthly: restore into a temp db, compare row counts, drop, record
  in health.
- Wire both into the worker scheduler (backup at `backup.at`; catch-up on wake per C5).

### P0 verification
- `make test` green (P0 unit tests: config load, day math, token auth, CSRF, maintenance
  middleware, gate open/close logic, canary expiry incl. `CANARY_OVERDUE`, LLM parse incl.
  `…` + brace-in-reasoning).
- Health page opens in browsers on stage1-host, MBP#1 (via tailnet) and Win11 (via tailnet)
  after pairing — **all via `public_url`, never loopback (B6)**.
- Gate: closed state blocks a real-content map call (assert `PRIVACY_GATE_CLOSED`); after D3
  confirm + passing canary, gate open.
- **V3 guard (A3):** with a fake v3 label loaded (a dummy `com.lisa.collector` LaunchAgent,
  or a process matching `lisa-collector`), the node reports `V3_ACTIVE` and refuses to
  collect. The full v3-guard integration test stays in P2.
- Backup: one full run completes; restore drill passes (row counts match).
- D5: the Postgres.app verification set passes (P0.1).

### P0 acceptance
Health page reachable from all three machines (via `public_url`); gate opens after D3 +
canary; the `V3_ACTIVE` guard behaves (fake-label test); Postgres extensions present (core
startup check passes). **v3 keeps running through P0** — the P0 node has no readers (they
arrive in P1.1), collects nothing, and runs safely beside v3; the switchover is P1.0 (A1).

---

## Phase P1 — Collection

### P1.0 The install session (owner present; same sitting as the P0 owner actions)
Contract §22 order, in one session (A1/A2):
- **R1:** `launchctl bootout gui/$UID/com.lisa.collector` and `…/com.lisa.digest`; move both
  plists to `~/lisa/retired-launchagents/`. (Owner approved, 2026-09-24.)
- **R2:** verify `launchctl list` shows neither label and `pgrep -f lisa-collector` is empty.
- **R3:** install + boot the v4 node (`ops/install-node.sh`); the V3Guard passes now.
- **R4:** owner grants FDA, Reminders, Calendar, Contacts, Notifications to `LisaNode.app`
  (contract §24-2).
- (R5 — deleting `~/lisa/` — happens after P2 acceptance B, owner confirms in writing.
  R6 — removing the old FDA entry — with R5.)

Why here: baselines are "wherever the source stands when the reader first runs" (contract
§7.4) and nothing is backfilled, so every source that uses a cursor — mail, iMessage, fs,
Edge, Office MRU, photos — has a hole covering the whole interval between the v3 shutdown
and the first v4 read. Doing the P0 owner actions and this session in one sitting collapses
the gap to a single session.

### P1.1 Node readers (`node/Sources/LisaNode/Readers/`)
Each reader: incremental from cursor, isolated error → health code, baseline-on-first-run per
contract §7.4 (baseline emits no events; reminders/calendar/contacts seed `object_state`
only).

- `FSEvents.swift` — flags `FileEvents|UseExtendedData|NoDefer`, latency 2 s,
  `sinceWhen` = cursor; **`eventPaths` is `char**`, bridge with
  `assumingMemoryBound(to: UnsafeMutablePointer<CChar>?.self)`** (v3 SIGSEGV lesson — do NOT
  bridge as CFArray); `lstat` for existence; inode-paired renames; >50 events/10 s/dir → one
  event with `payload.batch_count`; ID jump / `MustScanSubDirs` → `FS_GAP`; extension
  allowlist per contract; `kMDItemWhereFroms` → `payload.from_url`; default excluded paths.
- `IMessage.swift` — copy `chat.db` + `-wal` + `-shm` to `$TMPDIR/lisa/`, open read-only,
  delete after; `message` LEFT JOIN `handle`, JOIN `chat_message_join` → `chat`; UID
  `imsg:{guid}` (edits `:edit:{date_edited}`); cursor = `message.ROWID` + rescan last 200
  rows for edits/unsends; `date` > 1e12 → ns else s (from 2001); NULL `text` → decode
  `attributedBody` typedstream `[K]` (on failure: body null + `payload.decode_error=true`,
  counted in health); `is_from_me` → sent/received; `associated_message_type ≠ 0` →
  `reacted`.
- `Edge.swift` — every 15 min; copy `History` + `-journal`; `visits` JOIN `urls`;
  `visit_time` µs from 1601; drop subframes (`transition & 0xFF` ∈ {3,4}); cursor
  `visits.id`.
- `OfficeMRU.swift` — every 60 s; the three v3 MRU paths (Word/Excel/Powerpoint
  `Documents_en-CN` JSON); `seen_hash` last 2000; action `opened`; OneDrive URLs kept as-is.
- `Reminders.swift` — EventKit, every 5 min: incomplete + completed-in-last-30-days; UID
  `rem:{externalId}:{action}:{discovered_ms}`; full state also in `states[]`;
  `requestFullAccessToReminders` (macOS 14+ path; on 13 use `requestAccess` — verify at
  build, write result back).
- `Calendar.swift` — EventKit `predicateForEvents`, window `window_days [0,180]`, every
  5 min; UID `cal:{externalId}:{occurrenceStart}:{action}:{discovered_ms}`;
  `occurred_at` = discovery time; `payload.start/end` = scheduled times; agenda read from
  `object_state`.
- `Contacts.swift` — CNContactStore, hourly diff; `states[]` only,
  `object_uid = {node}:{identifier}`; E.164 phone + lowercase email identifiers.

### P1.2 Core side
- `api/ingest.py` — full ingest processing, one transaction per batch, contract §9.4 steps
  1–8 (schema validation → tombstones → shadow → self-observation via `ops.lisa_objects` →
  day compute → upsert with `seen_on`/parser_version rules → late-event dirty marking →
  states/watermarks/health upserts). Idempotent: same `batch_id` returns the same result.
- `api/executor.py` — claim/report endpoints (P1: plumbing + lease fields; verbs land in P2).
- `GET /node/config` full implementation (node slice: sources+mode, filters, HMAC key,
  tombstones, `config_version`).
- Baseline + watermark + coverage logic (`days.py`) per contract §10.2.
- (Mail: already running since P0.2 — no new P1 work beyond what the IMAP reader needs.)

### P1 verification
- Unit: time conversions (3 epochs, tz timeline, 04:00 rollover); event UIDs per source;
  baselines emit no events; OTP redaction incl. OCR text; phone/email/quote normalization;
  idempotent ingest (same batch twice → same result, no dup rows).
- Live: `LisaNode --selftest` passes per source; events flow into `raw.events` for fs/edge/
  office_mru/imessage/reminders/calendar; contacts in `object_state`.
- **Overnight sleep test:** sleep stage1-host overnight (or stop the node 36 h in tests) →
  wake → no loss; coverage partial→complete; day marked dirty and re-digestable.
- **Permission revocation test:** revoke one grant (e.g. Reminders) → health page shows the
  zh `EK_DENIED` message within one heartbeat.
- **D1:** `attributedBody` decode cross-check against `imessage-exporter` on the real
  `chat.db`, counts only, difference < 1% (contract §24-11).
- **D2:** after a rebuild and re-sign, `LisaNode --selftest` still passes — i.e. the grants
  survived; if not, the owner re-grants (contract §24-3).

### P1 acceptance
Events flow from all P1 sources; overnight sleep loses nothing; a revoked permission shows
on the health page in plain Chinese; D1 and D2 pass.

---

## Phase P2 — Acceptance slice (the doctor's-email path)

### P2.1 Snapshot
- `snapshot.py` — deterministic day rendering per contract §11: JSON (coverage, agenda from
  `object_state`, events by kind, counts) + markdown (fixed order, local times,
  `origin='lisa'` items labelled "lisa 创建"); `snapshot_hash` = sha256 over sorted
  `(id, content_hash)` pairs + agenda hash; render to `data/rendered/snapshots/`.

### P2.2 Digest pipeline (`digest/`)
- `unitize.py` — grouping per contract §12.1 (P2: mail thread, iMessage chat-per-day,
  calendar/reminders day-unit, owner_note; web/file/photo/notes land in P3/P4);
  `unit_id = sha256(day|type|group_key|part)`; token estimate UTF-8 bytes/3; split at largest
  >2 h gap else fixed chunks; exclude `origin='lisa'`, `reacted`, contacts.
- `map.py` — request per contract §12.2: system prompt `prompts/map_system.md`
  (source blocks are data; JSON only; source language; verbatim quotes; never invent dates);
  user content with `<context>` (entity summaries + open commitments) and `<unit>`;
  output schema per §12.2; parse failure → `:8017` JSON repair → one map retry → unit
  `invalid` (`DIGEST_INVALID`). Map outputs cached by input hash.
- `validate.py` — V1–V9 exactly per contract §12.3 (schema; refs inside unit/context;
  normalized-quote substring with `normalize` = NFKC + strip whitespace + lowercase +
  full-width→ASCII, 4–200 chars, photo events match OCR body only within conf ≥ 0.5 spans;
  enums/confidence/due ranges; claim_by owner rule; trust = min over sources
  web < third_party < system < owner, inferred rules; suggested_action allowlist + args
  schema; resolutions target context commitments; photo due re-parse via `dateparse`).
  Failing facts dropped individually into `units.output.rejected`.
- `dateparse.py` — `YYYY年M月D日`, `M月D日`, `M/D`, `D Month`, `Month D`, ISO, `周X`/`星期X`
  relative to `taken_at`.
- `reduce.py` — input: unit summaries + validated facts F1…Fn + coverage; output markdown
  with exactly `## 今天 / ## 重要的事 / ## 决定与承诺 / ## 待你处理`; only `[F<n>]`
  citations allowed (unknown n → retry once → strip); code appends `## 覆盖情况`.
- `schemas.py` — JSON schemas for map output, photo describe, ask tool results.
- `prompts/map_system.md`, `prompts/reduce.md`, `prompts/photo_describe.md` (P3),
  `prompts/ask_system.md` (P3).
- `prompt_version` = first 12 hex of prompt file sha256; record `model` + `prompt_version`
  on units/facts/digests.
- Worker night lane wired: ready → unitize → map (priority order) → validate → reduce →
  propose → (dream stub until P3); budgets from `usage`, per-kind lines, `skipped_budget`
  beyond.

### P2.3 Facts + propose + actions
- `brain/facts.py` — upsert by `stable_key` (contract §16.1:
  `sha256("v1|" + fact_type + "|" + anchor_event_uid + "|" + due_or_none)`);
  commitment/request start `open`; re-digest retraction rule (fact not returned →
  `retracted` unless a `succeeded` action references it).
- `digest/propose.py` — rules P1–P7 per contract §16.2 (P4 reminder dedup: bigram-Jaccard ≥
  0.6 on title **and** due within ±1 day, both-null matches; P5 same-entity open-action
  similarity; P7 nightly cap, overflow listed in brief as "lisa 还注意到").
  `idempotency_key` per §16.1 (never LLM wording); `action_id = "ac_" + sha256(key)[:24]`;
  `requires_visual_confirm` iff all sources are photos; review windows per §16.2.
- `actions.py` — full state machine per contract §16.5 (every legal transition; all others
  rejected; every transition appended to `action_events`); CAS on `args_hash_seen` for
  approval; `edited_args` re-validated; visual-confirm enforcement (422
  `VISUAL_CONFIRM_REQUIRED`); expiry/supersede/stale/cancel paths; `POST /actions/{id}/decide`.
- `api/executor.py` verbs: `create_reminder` — node `Executor.swift` executes via EventKit:
  claim → pre-check marker (`url == {public_url}/a/<action_id>` or notes line
  `lisa:<action_id>` → report `succeeded {"reconciled": true}`) → `save(commit:true)` with
  marker URL + notes line → report with `external_id` (never re-execute on report failure);
  core adds `ops.lisa_objects` row + `object_state` upsert on success; lease expiry →
  `outcome_unknown`; reconcile task after `reconcile_delay_minutes`.
- Web: **pending** page (approval cards with quoted sources, decide buttons, args hash
  shown), **receipts** page (action + full `action_events` chain), **today** page (brief:
  highlights, pending, missed, due commitments + "done?", health, "仍在处理 N 个片段" when
  night lane unfinished).
- Notification: brief-ready notification on the Mac node with an "open" button only.

### P2.4 CLI
- `cli/lisa.py` — single file, stdlib-only, Python 3.9: `status`, `brief [day]`, `actions`,
  `approve <id>`, `deny <id>`, `ask "<q>"` (P3), `forget` (P3). Talks to the API with the
  paired client token (via `public_url`).

### P2 verification
- Unit (contract §23.2): V1–V9 pass+fail each; dateparse; stable_key/idempotency_key stable
  under LLM rewording; P1–P7; action state machine (all legal + ≥12 illegal transitions);
  CAS conflict; visual-confirm requirement.
- Integration (contract §23.3): offline node 36 h; lease expiry + reconcile; self-observation
  (lisa-created reminder re-collected → no new fact); **full v3-guard integration test**
  (fake v3 label → node refuses); migration dry run (two cores, move-announce → drain, no
  duplicates).
- **D4:** EventKit external IDs are stable for reminders across re-reads, and the marker URL
  plus notes line round-trip (contract §24-12).
- **Acceptance A** (automatic, synthetic fixtures + canned-LLM stub server, temp Postgres):
  1. inject "请在9月30日前预约年度体检" email → night lane →
  2. exactly one `create_reminder` proposal in `awaiting_review` citing the email with a
     valid quote;
  3. approve → mock executor → `succeeded`, `lisa_objects` row, full transition chain in
     `action_events`;
  4. re-digest same day → no new proposal;
  5. inject clinic confirmation → digest → commitment `done`;
  6. synthetic appointment-card photo variant → `requires_visual_confirm`; approve without
     `visual_confirmed` → 422.

### P2 acceptance — test B (real, manual, owner present; gate must be open)
1. From another address, send a Chinese body-check email to the owner's icloud.com mailbox.
2. Next morning: Mac notification opens the proposal with the quoted source.
3. Approve from Win11's browser — **via `public_url` (B6)**.
4. stage1-host's node creates the reminder; on iPhone the reminder's URL opens lisa's
   receipt page — **the iPhone must be on the tailnet for the link to open (B6)**.
5. Ask "when is my body check due" in English → answer cites the email (ask lands in P3;
   step 5 verified then — P2 gate is steps 1–4).
After test B passes and the owner confirms in writing: R5 (`rm -rf ~/lisa`) + R6 (remove the
old FDA entry).

---

## Phase P3 — Memory, forget, photos

### P3.1 Brain
- `brain/entities.py` — ULID ids; resolution order per contract §14.1 (exact identifier →
  contact-lazy create → map ref → new only with identifier or ≥2 appearances → LLM fuzzy →
  `merge_entities` proposal only).
- `brain/pages.py` — recompile dirty pages from ≤200 active facts (older summarized first);
  every sentence cites `[fa_…]`; single-source third-party claims phrased "据…称";
  `pinned_md` never read or modified by the compiler; keep last 20 `page_versions`.
- `brain/dream.py` — nightly after digest: recompile dirty pages → propose merges →
  spot-check 5% of new citations ("does the source support the statement? yes/no", failure
  rate in health) → "done?" questions on overdue commitments in the brief → embed new items.
- `brain/render.py` — `rendered/brain/<kind>/<slug>.md` (front-matter, `## Pinned /
  ## 编译结果 / ## 时间线`), temp-file-then-rename.
- Web: **entities** pages + `PUT /entities/{id}/pinned` (audited, marks dirty).

### P3.2 Search + ask
- `search.py` — hybrid per contract §15.1: bigm (events/facts/pages) + vector (top 50 each),
  query in zh **and** en (translation = one cached gateway call); RRF k=60; rerank top 50
  via `:8014` (verified API shape) → top 10; embedding rules per §15.1 (event kinds, first
  2000 chars; facts; pages; query prefix per config, frozen by eval).
- `ask.py` — route by question type (exact → structured queries; synthesis → entity pages
  first, then search); read-only tools `search, get_events, count_events, agenda,
  get_entity, get_page, get_facts, get_digest` + write-only `propose_action` (origin `ask`,
  pending); tool results wrapped in `<source>` blocks; 8 rounds / 40k tokens counted from
  `usage`; answers cite `ev_<id>`/`fa_<id>` rendered as links; SSE events `status, tool,
  citation, answer, proposal, done` (model called non-streaming; progress still streams).
- Web: **ask** page, **search** page.

### P3.3 Forget
- `forget.py` — preview/commit per contract §17 (selector kinds event/events/thread/person/
  range; preview token 10 min; commit = one transaction: tombstones as HMACs only → delete
  events (cascades to fact_sources, blobs, child docs) → orphaned facts → units intersecting
  E → embeddings → person-scope contact rows/entity/identifiers/page/versions → mark pages
  dirty + delete versions → actions args/rationale/result → `{"forgotten": true}` → days
  dirty + digests deleted → eval jsonl rewrite → rendered files removed → audit (counts
  only) → zh receipt per §17.11 → `config_version` bump so nodes refresh tombstone cache
  and purge outbox rows).
- `lisa-core forget-verify <receipt>` — asserts zero matching rows in every layer.
- 03:30 maintenance: `VACUUM (FULL)` on forget-touched tables + `CHECKPOINT`.
- Web: **forget** page (preview → commit → receipt).

### P3.4 Photos
- Node `Photos.swift` + `Ocr.swift` — `PHPhotoLibrary.fetchPersistentChanges(since: token)`;
  skip hidden / `exclude_albums` / non-representative bursts; videos → metadata-only event;
  screenshots off by default (metadata-only, no derivative, no OCR); daily cap
  `max_per_day` (overflow metadata-only, counted in coverage); 1280 px derivative, JPEG 0.8,
  EXIF stripped, ≤800 KB; Vision OCR `.accurate`, `["zh-Hans","en-US"]`, revision 3, keep
  conf ≥ 0.3, reading order; `payload.ocr = {engine:"vision/3", n_lines, mean_conf, spans}`;
  OTP redaction on OCR text; event `photo:{PHCloudIdentifier}` (fallback
  `photo:{node}:{localIdentifier}`); location rounded to `location_dp`, never
  reverse-geocoded; derivative shipped to `/ingest/blob` only after event ACK.
- Core `media/photos.py` — night lane step 1, one photo per call: image + OCR text to
  llm-gateway (non-streaming); output `{"kind","description"}` only; write
  `title = description`, `payload.describe`, trust per kind (third_party for
  screenshot/document/receipt/whiteboard, else owner); bump `parser_version`, recompute
  `content_hash`; blob deletion rules (keep while action pending; 10-day max); 3 failures →
  `PHOTO_DESCRIBE_FAILED`, blob deleted.
- `GET /actions/{id}/image` — JPEG for pending photo actions; approval card shows image with
  quoted span highlighted; Approve disabled until "我已核对图片中的日期和时间" is ticked.
- `api/ingest.py` blob endpoint (`POST /ingest/blob`, ≤800 KB photos).

### P3.5 Owner note
- `POST /v1/notes` — owner note event (trust `owner`, actor `me`, OTP redacted); web "告诉
  lisa" box.

### P3 verification
- Unit: entity resolution order; forget cascade counts per layer; `forget-verify`; photo
  trust rules; RRF merge; ask round/token limits.
- **Cross-language search:** "body check" finds the 体检 email (fixture + live).
- **Forget verified across all layers:** forget a synthetic person → preview counts match
  commit → `forget-verify` zero rows in events/facts/pages/embeddings/units; tombstone stops
  re-collection (node re-ships the event → `blocked`).
- **D3:** PhotoKit persistent change tokens and `PHCloudIdentifier` verified on macOS 13
  (contract §24-13).
- Acceptance A step 6 (photo visual confirm) passes live with the synthetic card.
- Acceptance B step 5 (English ask cites the Chinese email) passes.

### P3 acceptance
"body check" finds 体检; forget verified across all layers; photo card → visual-confirm flow
works; ask answers cite sources.

---

## Phase P4 — Completion

- `Notes.swift` — `NoteStore.sqlite` copy, gunzip + protobuf text, 1 h overlap cursor, locked
  notes title-only (format verification §24-17 first).
- `SFL.swift` — `*.sfl2/sfl3` via NSKeyedArchiver (format verification first).
- MBP#1 node: install `LisaNode.app` + LaunchAgent on MBP#1, pair, enable in config
  (`nodes.mbp1.enabled: true`); machine-local sources only (fs/edge/office_mru/sfl).
- `create_calendar_event` + `complete_reminder` verbs (executor + args schema §16.4).
- `content_roots` — node ships file bytes as `file_content` blobs; core converts (converter
  must be live by then), writes markdown into the file event body, deletes blob.
- Triage via `:8017` (`inference.triage_enabled: true`) — web/file units only,
  `{"keep": true|false}` → `skipped_triage`.
- `lisa-core render --all` full implementation (all rendered views).

### P4 acceptance
30 days of stable digests (health green, no `GATEWAY_DOWN`/`DIGEST_INVALID` spikes, coverage
honest, budgets within config).

---

## Phase P5 — Migration runbook (executed when the mini arrives)

Code built now, executed later per contract §20:
- `lisa-core move-announce --to URL` — heartbeats return `core_moved_to`; nodes drain
  outboxes, check new core `/health`, rewrite `core_url`, re-register.
- Node shadow mode — `sources.<x>.mode: shadow` → ingest writes UIDs to `ops.shadow_uids`
  only; `lisa-core shadow-report --node mini` → per-source match rate vs live event_uids
  (reminders/calendar vs `object_state` UIDs); ≥99% required to continue (M1).
- `secrets export --for-backup` (HMAC + IMAP + restic only; **never the gateway key** — the
  fleet issues the mini its own key, M0).
- **M3:** pg_bigm is rebuilt against the arm64 Postgres.app on the mini (B4); the
  Postgres.app GUI must never start a second server on the mini either.
- `ops/` runbook doc for M0–M6 (owner grants on the mini; M6 decommission is mandatory:
  drop stage1-host db + PGDATA, remove Keychain items + agents, revoke stage1-host gateway
  key).
- Integration test (contract §23.3): two cores on one machine, move-announce → drain → no
  duplicates.

### P5 acceptance
M5: all health green on the mini; acceptance B passes on the mini; a forget test passes.

---

## Test infrastructure (built in P0, grown per phase)

- `core/tests/make_fixtures.py` — synthetic-only fixtures: `chat.db` (incl. `attributedBody`
  rows), Edge `History` + `-journal`, Office MRU JSON, `.eml` files (zh/en/mixed/HTML + one
  PDF attachment), synthetic images (appointment card with Chinese date; screenshot with
  verification code), canned-LLM transcript set. Generated into `core/tests/fixtures/`
  (gitignored).
- Canned-LLM stub: a local HTTP server (stdlib) serving the transcript set, pointed at via
  test-only `config.yaml` — used by Acceptance A and digest unit tests.
- Temp Postgres for integration/acceptance: separate `initdb` dir + socket dir + port in
  `$TMPDIR`, created by a pytest fixture; never touches `~/lisa-core/pg`.
- pytest layout: `tests/unit/`, `tests/integration/`, `tests/acceptance/`.

## Assumptions & contingencies

- **Mail:** the owner's icloud.com address, single account (owner decision, 2026-09-24 — not
  a coder default). The app-specific password must exist in Keychain before the IMAP reader
  activates; if the owner adds accounts later, it's a config change only.
- **CLT:** softwareupdate offers nothing (measured 2026-09-24) → CLT 15.x via the
  developer.apple.com download (owner's Apple ID) or App Store Xcode 15, before P0.3. If the
  download is blocked, stop and ask the owner — do not build the node on 5.8.1 with the
  `#available(macOS 14)` branch left uncompiled.
- **Gateway key:** if the fleet cannot issue a dedicated lisa key, stop and ask the owner —
  never reuse another agent's key.
- **D3 dump state:** if the gpu-host dump is still on when P0.4 runs, the fleet switches it
  off first (one-line flag + proxy restart; owner approval pending), then the owner confirms
  via the health page.
- **Converter:** stays null until the fleet verifies a no-retention converter (contract
  §24-15); attachment conversion is inert meanwhile (`documents.mail_attachments: false`).
- **EventKit on macOS 13:** `requestFullAccessToReminders` is macOS 14+; on 13.7.8 use
  `requestAccess(to:)` — verify at P1 build time and write the result back into contract §7.3.
- **Busy probe:** llm-gateway `/stats` has no clean interactive-load signal; day lane yields
  on observed p50 latency > `backoff_latency_s` (contract's stated fallback). If the fleet
  adds a probe later, `busy_probe_url` is already a config key.
- **Worker placement:** same host as `serve` for now; open design item, decided at
  migration with the storage question (E2).
- **v3 data:** `~/lisa/` is preserved untouched through P2; deletion (R5) requires the
  owner's written confirmation after acceptance B.

## Owner action list

**The one session, in this order (G.5):**
1. Trust the "Lisa Code Signing" certificate in Keychain Access (contract §24-1).
2. `tailscale serve` HTTPS for :8443 on stage1-host + tailnet HTTPS certs (§24-4).
3. Create the iCloud app-specific password → Keychain `lisa.imap.icloud` (§24-5).
4. R1: stop the two v3 LaunchAgents; move plists to `~/lisa/retired-launchagents/`.
5. R2: verify both labels gone + no `lisa-collector` process.
6. R3: install + boot the P1 node.
7. R4: grant FDA, Reminders, Calendar, Contacts, Notifications to `LisaNode.app`.

**Outside the session:**
- CLT 15.x install (before P0.3) — developer.apple.com download, owner's Apple ID.
- Keep an offline copy of the restic password (P0.5).
- Confirm the two pending approvals: gpu-host changes (gateway dump off + lisa's own
  gateway key) and restic install on backup-host.
- D3 confirmation click (`POST /health/confirm-d3`) after the fleet switches the dump off.
- R5/R6 after P2 acceptance B (written confirmation).
