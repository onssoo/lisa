# Owner decisions — log

Append-only. Records decisions the owner has made, the reason, and whether the design documents
already cover them. The reviewer folds these into the next revision of `design.md` / `contract.md`.

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
