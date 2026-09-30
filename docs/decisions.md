# Owner decisions — log

Append-only. Records decisions the owner has made, the reason, and whether the design documents
already cover them. The reviewer folds these into the next revision of `design.md` / `contract.md`.

## 2026-09-30 — work files: collected per machine, content owned by the file-brain

**Decision (reviewer's wording).** *Work files are collected by LisaNode on each Mac (roots per machine;
iCloud Drive owned by m2). Metadata goes to LISA as file events; content of whitelisted types is uploaded
by the node directly to fibr `/upload`, which owns storage, parsing (docreader on m2), chunks and vectors.
LISA links by `(machine_id, rel_path, content_hash)`. Deleting a file on disk keeps its history; only an
explicit forget purges it.*

Full spec: `file-collector-spec-v1.md`. The collector is an **extension of LisaNode's existing FSEvents
reader**, not a new program; content goes **node → fibr directly**, never through core.

> ⚠️ **The last sentence is the intent, not the current behaviour.** Verified in `fv_gc.py` and against the
> live database (spec §10): the GC's 7-day trash grace is timed from **`first_seen_at`**, and there is **no
> `trashed_at` column** — so anything older than a week that loses its last reference is purged on the
> **next GC run**, and the purge **deletes the blob from disk**. Live right now: **23 contents sit at
> `refcount = 0`**, first seen 2026-09-03…09-10, already past that test.
> **Two requirements follow:** (1) a disk-delete must **never** remove the `files` row — mark it instead,
> so the trigger never releases the content; (2) **`fv_gc.py` needs a `trashed_at`** so "trashed" is timed
> from the trash. Without (2), any future reconcile step that removes rows starts destroying history
> silently, on the first GC cycle.

## 2026-09-30 — review round 2: four settled decisions, and two exposures found

The external reviewer accepted the coordinator's corrections and settled the four open items. Recorded
here because each closes something earlier documents left ambiguous.

### 1. Frontend routes stay **flat** — no `/api` prefix

**Decision.** The client API keeps the paths `contract.md` already reserves. `GET /days/{day}` is
**reshaped into the full `DayPage`** and carries a **`shape_version`** field so the shape can evolve
without a route change. `POST /cards/{id}/mark` is a **new route, deliberately separate** from
`POST /actions/{id}/decide` — *a card is a note to the owner; an action is something the executor does.*
`POST /input` only classifies the typed text and delegates to the existing `/notes` or `/ask`.
One set of routes is easier to keep straight than two.

**Applied to** `frontend-spec-v1.md` §7.1/§7.3/§10/§14 and its verification note §2 (the conflict table
is now resolved).

### 2. **No `reply` card kind** — the contract's four stand

**Decision.** Keep `{commitment, health, change, prep}`. A reply the owner owes is a **`commitment`**
card whose next step is **`draft_reply`**. The *next-step type* is what later links to the send rule, so
the kind list must not grow for it.

**Applied to** `frontend-spec-v1.md` §5 and §7.2.

### 3. Sandbox host: **the serving host, as its own VM**

**Decision.** The sandbox runs on the serving host (not the storage host, whose ≈4 GB free and already
shed parsing), as **its own VM — not the one the document-parse service uses**. Outbound network blocked
by default; **no host mounts**; **budget ≈1 GB and 1–2 cores per browser task, capped at 2 running at
once to start**.

**Why:** on macOS there are **two** walls (container → Linux VM → macOS) versus one for a Linux-native
container; the Linux VM is already paid for. The coordinator's objection — an escape lands next to the
orchestrator — is answered by giving the sandbox its own VM rather than by moving it off the machine.

**This restores the original rule, and it is recorded on its own line:**
> **The sandbox never shares a machine with the database.**

It also supersedes the reviewer's earlier suggestion of a trial on the storage host.

**Not built until the daily page is trusted** (the roadmap's gate).

**Also: do NOT carry out the topology's “remove the llama.cpp LaunchAgents from the serving host” item.**
That item assumed the model endpoints belong on the GPU host; they deliberately do not. Removing the
agents would revert the verified 2026-09-27 migration and break every consumer.

### 4. Backups: **restic to the NAS as copy 2**, and the gap stated plainly

**Decision.** Set up the Synology (reachable, already on the tailnet) as **copy 2 now**. For the offsite
copy, use an encrypted repo in a cloud bucket or the NAS's own cloud backup.

> **✅ BUILT AND VERIFIED 2026-09-30 — with one deviation from the plan, forced by the hardware.**
> The NAS is a **DS216j**: its volume is **ext4** (so **Active Backup for Business cannot run** — it needs
> Btrfs) and its DSM build **has no `sftp-server`** (so **restic-over-SFTP fails**, and so does rsync's ssh
> transport, while a plain ssh command works). The backup therefore ships as **`ssh` + `tar`**, not restic:
> nightly `pg_dump -Fc` of every database → `/volume1/backup_0/lisa-db/YYYY-MM-DD.tar`, 14 kept, written to
> `.partial` first and renamed so a half-written file can never be mistaken for a good one.
> **Restore drill passed**: fetched the archive back from the NAS, unpacked it, `pg_restore`d into a scratch
> database and compared row counts — `memory.days` 39=39, `raw.events` 444=444, `memory.digests` 6=6.
> **What this closes:** copy 2 is real. **What it does not close:** the offsite copy — both copies are still
> in the same building, so a fire or a burglary still loses everything.

> **Until that offsite copy exists, the record is protected against one disk failure and nothing else.**
> Both current copies are in the same building: **a fire or a burglary loses everything.** This entry says
> so plainly rather than leaving it implied.

### Exposures found in this round (actions accepted, not yet done)

**A. The model and parse ports are reachable from the public host.** Measured: the public host can open
`:8013`, `:8014` and `:50051` on the serving host. No login, bound to `0.0.0.0`. On a private tailnet this
is acceptable, but it means **the internet-facing machine is inside the trust boundary of the model
ports**. Decision: **add a Tailscale policy so only the serving host, the MBP and the storage host reach
those ports.** Note: the tailnet currently has **no tags applied**, so this needs tags defined and applied
first (see `infra-status-2026-09-30.md` §7).

**B. File activity has been frozen since 2026-09-26.** The `fs`, `edge` and `office_mru` collectors ran on
the retired workstation. Until a node runs on the MBP, the day page shows **no work activity**, and the
source strip must mark those sources **`missing` — not hide them** (R9: honest over pretty). The nightly
disk-vs-record check fills most gaps when the laptop returns.

## 2026-09-30 — frontend spec v1: two deltas (digest emits JSON; Ask is POST)

Recorded because both change what earlier documents said. Full spec: `frontend-spec-v1.md`; the
code/contract check is `frontend-spec-v1-verification-2026-09-30.md`.

**Delta A — the nightly digest must emit JSON, not markdown (spec §7.3). This changes `contract.md`.**
Today the contract specifies markdown in three places: the digest output (*"markdown with exactly these
sections: `## 今天`, …"*), `memory.digests.summary_md text NOT NULL`, and the archive
(`archive/YYYY/MM/DD.md` as plain markdown). The spec inverts it: the digest writes the review, cards and
preview as **JSON with citations**, and the markdown archive is **rendered from that JSON**. The frontend
must not parse markdown headings.

*Why:* prose with inline citations is not machine-checkable, so the UI would have to scrape headings —
and citation-checking (X12) and the L1 action registry both need structure. This is the single change
that makes every later feature cheap.

*Consequence:* the contract's digest-output section and the `summary_md` relationship must be updated;
the archive becomes a rendering rather than the source of truth.

**Delta B — Ask is `POST`, never `GET` (spec §7.1). This does *not* change the contract; it corrects the
coordinator's proposal.** The contract already reserves `POST /ask` with the question in the body. The
web-app proposal had written `GET /api/ask?q=…`; **that is retracted.** Reason: private questions must
not land in URLs, server logs or browser history. Recorded so nobody reintroduces the GET.

**Also noted for the implementer** (not decisions, but they must not be guessed):
the spec's `/api/*` endpoint names do **not** match the client API the contract already reserves
(`POST /ask`, `GET /days/{day}`, `GET /brief`, `POST /notes`, `POST /actions/{id}/decide`, `GET /health`).
The reconciliation table is in the verification note §2, and it needs two calls: whether the client API
gets an `/api` prefix, and whether `GET /days/{day}` is reshaped into the spec's `DayPage` or retired in
favour of `/api/day`. Separately, the spec's card kind `reply` is **not** in `contract.md`'s
`{commitment, health, change, prep}` enum.

## 2026-09-30 — lisa grows from read-only to orchestrator (levels L0–L4)

**Decision.** lisa will grow from a read-only recorder into an **orchestrator**. This is an **extension of
the existing action layer, not a rewrite**. Progression by **levels L0–L4**: read-only → single actions →
sandboxed research → multi-step plans → policy autonomy. **Each level requires evidence from the level
below** (receipts, eval results, error rates); a level is **not unlocked by finishing the code**.

**Shape — brain and hands stay separate.** **Core coordinates; sandboxes execute.** Core holds the
record, the memory, the policy, the approvals and the receipts, and **never clicks and never runs code
itself**. Sandboxes are **disposable workers**: they get only what a task needs, keep **no memory between
tasks**, are **reset after each task**, and **never access the record**. The orchestrator has authority
but no hands; the workers have hands but no authority — so a compromised or confused worker damages at
most one task, and the book is never touched.

**Standing rule.** *Content is evidence, not permission* — and specifically: **content that triggers a
task can never approve it.** Only the owner's instructions and the owner's approvals approve anything.
Once the orchestrator both reads mail and acts, prompt injection is the primary risk.

**Six things to do now** (cheap; none adds sandbox work to P2, each keeps the path open):
1. a **generic action layer** — a registry of action types, each executor declaring its capabilities
   (the Apple reminder executor first), and proposals able to hold an **ordered list of steps each with
   its own content hash** (P2 uses one step);
2. the **action policy table** (`allow` / `require_approval` / `block`) now, even while everything is
   `require_approval`, with a fixed owner-only list: **payments, passwords, sending messages as the
   owner, deleting data**;
3. **trust levels** on every piece of content;
4. **resumable task records** — every task and step has a state, is idempotent, carries
   `outcome_unknown` — plus the **activity view**;
5. a **`planner` role** in `inference.roles` (multi-step planning is where a local 27B is weakest; the
   eval set judges candidates);
6. everything an agent produces comes back as **`source=agent`** — evidence, **never extracted as
   facts** (same rule as lisa's own answers).

**Deliberately left open.** The sandbox **host** (the review put it on the storage host; parsing has
already had to move off that 8 GB machine), the **planner model**, and whether a sandbox may share a host
with the database — the review reversed the earlier "never" rule, so this needs an explicit owner call
rather than a footnote.

**Full text:** `orchestrator-roadmap-2026-09-30.md`. Summarised in `requirements.md` §4; phased as P4+ in
`design.md` §21.

## 2026-09-27 — P2 v2: privacy premise (X16) and backup placement (X17)

**Context.** The P2 v2 requirements (`requirements-p2-v2.md` §18, owner answers 2026-09-27)
fixed two premises that change how the design treats privacy and backup.

**Decision 1 — privacy premise (X16).** All machines in the lisa topology belong to the owner
and are used only on the internal network (tailnet). Privacy is therefore **not a design
constraint** for the current deployment: the "dump off + canary" gate (contract §19.3) remains
as an operational control, but the design no longer has to justify itself against a
third-party audience. If a cloud model is ever used for the main role, a new entry in this log
must record the decision at that time — the current gate does not apply to cloud endpoints.
Owner answer 4 (no rush; no cloud plans).

**Decision 2 — backup placement (X17).** The restic target must never be on the Postgres host
(`backup-host`, the 2014 mini): a host failure would take the backup with it. The target is a
separate host or volume. The restore drill is quarterly (was monthly). Folded into contract
§21.1.

**Covered by the design?** Yes — §19.3 (gate) and §21.1 (backup) now carry both premises.

## 2026-09-25 — P1 node: TCC-gated readers use non-prompting status checks

**Problem.** After the P1 readers landed, the launchd node stalled on startup:
`pollReaders()` is sequential, and the TCC-gated readers (calendar, reminders,
contacts) called `requestAccess`, which pops a prompt. In a headless launchd
session the prompt is never answered, the call blocks the task thread on the
tccd XPC call, and the whole loop — including non-gated readers — stopped
shipping. A foreground run of the same binary worked (shipped 500 contacts
state rows), which proved the loop and ingest are correct and isolated the
stall to the prompt.

**Decision.** The node is a background agent and must never pop a TCC prompt.
The three gated readers now check the authorization status **non-prompting**
(static `authorizationStatus(for:)`, the only form available on the macOS 13.3
SDK) and proceed only when already authorized; otherwise the reader reports
degraded (contract §7.3) and the loop moves on. The owner grants calendar /
reminders / contacts in System Settings → Privacy & Security (R4); the next
poll picks the grant up automatically. No timeout helper was needed — the
earlier `withTimeout` approach was discarded because a pending prompt blocks
the calling thread and no timer on the main actor can fire.

**Consequence.** Until R4 is granted, calendar / reminders / contacts show
degraded in `ops.source_health`; everything else (office_mru, edge,
safari, photos, fs, imessage after FDA) collects normally.

## 2026-09-25 — P1 review r1: `LisaNode grant` is the supported TCC provisioning path

**Problem.** The non-prompting design has a provisioning dead-end: macOS
only lists an app in System Settings → Privacy after it has *asked*, and
the Reminders / Contacts pages have no "+" button. A daemon that never
asks can never get granted from the UI (only Calendar worked from the UI
on this machine). The P1 review (round 1) confirmed this and asked for a
one-time foreground grant mode.

**Decision.** `LisaNode grant` is the supported provisioning path: run it
in the owner's GUI session, answer the real TCC dialogs once
(calendar / reminders / contacts), it exits. The daemon's non-prompting
checks then see `.authorized` on the next poll — no restart needed. FDA
cannot be requested programmatically at all; the grant run prints the
System Settings → Full Disk Access reminder.

**Unsupported.** The direct TCC-database write (root sqlite into the
user/system TCC db, csreq blob copied from another row, tccd restart)
worked for FDA + Contacts during the review but NOT for Reminders (EventKit
reminder status reads a different TCC service row), and it is a
root-privileged workaround. Documented here only so future provisioning
attempts don't rediscover it; `LisaNode grant` is the only supported path.


## 2026-09-24 — build-plan review answers (D-11, v3, one session)

**Context.** The fleet and the external reviewer reviewed `build-plan-v41-2026-09-24.md`; the
change request (`build-plan-change-request-2026-09-24.md`) listed four things blocked on the
owner. The owner answered the same day:

1. **D-11 (mail accounts) — the owner's icloud.com address, single account.** This closes D-11
   as an owner decision (it was left open in requirements.md; the plan v1 had pinned it by
   assumption, which the fleet flagged). `sources.mail.accounts` gets one icloud entry; the
   address value lives only in the private `config.yaml`.
2. **Stop v3 (R1/R2) — approved.** Per the change request (A1), R1/R2 move to the P1
   node-install session, not P0.
3. **The one-session plan — agreed:** trust the signing certificate → `tailscale serve` →
   create the iCloud app-specific password → same sitting: R1/R2, install the P1 node, grant
   permissions.
4. **The read-only `softwareupdate --list` check — agreed and executed:** "No new software
   available" → CLT 15.x comes via the developer.apple.com Command Line Tools download (or
   App Store Xcode 15), not softwareupdate.

**Still pending (owner to confirm):** the other two cross-machine approvals from the change
request — on the GPU host, switching the gateway body dump off and issuing lisa its own
gateway key; and installing restic on the backup host. The owner's reply named only "stop v3"
from that batch; the plan v2 marks both as approval-pending.

**Covered by the design?** Yes — D-11 is a §7.3 config value; the retirement order is §22; the
session grouping is a plan-level convenience, not a design change.

## 2026-09-25 — P0 closure: the four approvals, first canary, gate open

**Context.** P0.1 was accepted (`docs/p01-review-2026-09-24.md`). The owner then gave the four
remaining P0-closure approvals in one reply ("1 approve 2 approve 3 yes 4 gate open"):

1. **GPU host — approved.** Switch the the LLM gateway gateway body dump off and issue lisa its own
   gateway key. Executed 2026-09-25: the dump was already off (`DEBUG_DUMP=False`, `server.py`;
   dump dir empty since the switch), and lisa's own key `sk-lisa-…` (32 chars) was issued and
   stored in MBP#2 Keychain `lisa.gateway`. The key is verified working against the live gateway
   (`:9000/v1/chat/completions` → PONG).
2. **Stop v3 (R1/R2) — approved and executed.** `com.lisa.collector` and `com.lisa.digest`
   booted out; both plists moved to `~/lisa/retired-launchagents/`; `launchctl list` shows neither
   and no `lisa-collector` process remains. (R3–R5 stay with the P1 node-install session.)
3. **D-11 — confirmed (yes).** The owner's icloud.com address, single account. `sources.mail`
   gets one icloud entry; the address value lives only in the private `config.yaml`.
4. **Gate open — approved and executed.** The privacy gate is now open.

**How the gate opened (P0 item 8, "first canary passes").** The canary *send* was a real P0 gap:
`weekly_canary` was a `NotImplementedError` and no code created `ops.canary` nonce rows, so the
gate could never auto-open. `core/lisa_core/canary.py` + the `lisa-core canary send` CLI now
implement §19.3 step 2: for each enabled service (gateway, embed, rerank, small) it generates a
`LISA-CANARY-<32 hex>` nonce, records it in `ops.canary`, and sends it as chat content (gateway /
small) or embed/rerank input. The gpu-host check (`ops/inference-canary-check.sh`, now handling
both log dirs and single log files, plus `CURL_EXTRA` for the tailnet SNI) searched the gateway
dump dir and the three service logs for each nonce and reported `found=false` for all four via
`POST /v1/ops/canary-report`. With `d3_confirmed` set and every enabled service's latest canary
reported `found=false` inside the 8-day window, `_recompute_gate` opened the gate:
`privacy_gate = {"open": true, "reason": "D3 已确认，所有服务 canary 通过"}`.

**Also fixed during closure.** `config.yaml` service URLs were placeholders that do not resolve
(`llm-gateway:9000`, `gpu-host:8013/8014/8017`); they now point at the the GPU host tailscale IP
(`llm-gateway`), and `public_url` is the real tailnet serve URL
(`https://stage1-host.tail69b436.ts.net`).

**Covered by the design?** Yes — the canary is §19.3, the retirement order §22, D-11 §7.3, the
gate recompute §9.2. The send implementation is the missing half of §19.3 that P0 item 8 required.

## 2026-09-25 — P1 pre-blockers: ops-token minting + Keychain ACL (empirically cleared)

**Context.** `docs/p0-closed-2026-09-25.md` carried two items into P1 that blocked the first
real LLM job: (1) no documented ops-token issuance path (the fleet had to promote a token via
raw SQL), and (2) the Keychain ACL was reported to block headless `lisa.gateway` reads (rc=36).

**Decision 1 — ops tokens mint via CLI, never via the pair flow.** `lisa-core ops-token
--node <id>` mints an ops-role token directly: raw token printed exactly once, only sha256 in
`ops.tokens`. `pair --role ops` is refused with a pointer to the CLI. Rationale: the pair code
is a browser/node redemption flow; ops issuance is an owner-on-the-core act and should not ride
the 10-minute single-use code path. Verified live: minted token → `POST /v1/health/confirm-d3`
200; after `revoked_at` set → 401.

**Decision 2 — no Keychain workaround needed; the rc=36 report was environment-specific.**
Empirical test 2026-09-25: a throwaway gui-domain LaunchAgent ran
`security find-generic-password -s lisa.gateway -w` → **rc=0, key returned**. The worker's
LaunchAgent is in the same `gui/$(id -u)` domain, so it can read the keychain headless. The
rc=36 seen earlier came from an *SSH* session (no GUI session context), not from launchd.
Regression guard: `tests/unit/test_keychain_headless.py` asserts the item is readable
(skips on machines without it). If the item's ACL is ever tightened, this test fails before
the worker's first LLM call does.

**Also fixed:** `lisa-core maintenance on|off` was wired to a subparser but missing from the
dispatch dict (KeyError on use); now dispatched.

**Covered by the design?** Yes — §6 (tokens hashed only), §9.2 (ops endpoints), §21.2 (CLI).


## 2026-09-24 — stage 1 is an all-in-one machine, deliberately and temporarily

**Decision.** Stage 1 runs everything on the same Mac (MBP#2): core, Postgres, the Apple-owner node,
the executor and the clients. This is chosen **only to develop fast**. It is not a statement about the
final architecture.

**Reason (owner's words, 2026-09-24).** A laptop must be shut down or its lid closed unexpectedly, which
would stop the whole process. The intended end state is a split: collectors gather and ship as soon as
possible, storage lives on the always-on mini, and the compute (pipeline) runs on the AI host, so that
opening the lid is answered with a finished brief instead of work starting then.

**Covered by the design?** Yes, stage 1 exactly:

- `design.md` §3.1 already defines stage 1 as MBP#2 running core, Postgres, the Apple-owner node, the
  executor and the clients, with the same code as the target stage (only `config.yaml` differs).
- `design.md` §3.3 already states the accepted cost: when the laptop sleeps, the API, the database and
  collection sleep with it; nothing is lost (cursors resume, the day state machine catches up), but the
  brief can be late and the other machines cannot reach lisa while it sleeps.

**One difference for the reviewer to reconcile later.** `design.md` §3.1 / §20 put the *target* core —
storage **and** pipeline — on the mini, with the GPU host used for inference only. The owner's end state
additionally separates **compute** (pipeline on the AI host) from **storage** (mini). That only affects
the target design, not stage 1, and should be settled when the mini hardware is actually present.

**Consequences accepted for stage 1.** Late briefs are expected and not a bug; coverage and health state
which machines and sources were missing. `sleep 0` stops idle sleep only — closing the lid still sleeps
the machine — so this cost cannot be configured away.

## Earlier decisions (already in the design documents)

- One gateway key, `model: auto`, fixed parameters; no tuning during the first pass
  (`design.md` §8, §22).
- No history import or backfill; the brain starts empty at go-live (`design.md` §22).
- Bulk processing at night; the day lane is overflow only, and it yields to interactive traffic
  (`design.md` §8.4).
- Photos go to the 27B, with node-side OCR and code-parsed dates (`design.md` §11.1).
- Retire v3 in order; delete its data only after go-live is verified, and only with the owner's
  confirmation (`design.md` §19).
- Secrets: only the tombstone HMAC key, IMAP and restic move between machines; the gateway key is
  newly issued per host (`design.md` §20).
- Backups exclude the photos kept for pending approvals (`design.md` §11.1, §18).
