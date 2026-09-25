# Build-plan change request — for plan v2

**Audience:** the coder. **Source of truth:** this repository (the fleet git host). **Status:** actionable now.
**Inputs folded in:** the fleet's round-1 review (`build-plan-review-2026-09-24.md`), the external
reviewer's round-2 response (`build-plan-review-round2-2026-09-24.md`), the owner decisions
(`decisions.md`), and the fleet's verification of every claim (same documents, with `file:line`
evidence). Nothing here overrides `design.md` v4.1 or `contract.md` v2.1; where an item changes either
document, it says so and the change is listed for the reviewer to fold in.

**How to use this:** produce `build-plan-v2`, apply every item below or decline it with a reason in the
plan, and update the decision log. Section F lists what is explicitly accepted and must not churn.

---

## A. Ordering — must change before P0 starts

**A1. Move R1/R2 (retire v3) out of P0.1 into the P1 node-install day.**
Remove "v3 agents gone" from P0 acceptance (plan:212). Reasons, both verified:

- Baselines are "wherever the source stands when the reader first runs" and nothing is backfilled
  (contract §7.4). Every source that uses a cursor — mail, iMessage, fs, Edge, Office MRU, photos —
  therefore has a hole covering the whole interval between the v3 shutdown and the first v4 read.
- The P0 node has no readers (they arrive in P1.1), so it collects nothing and can run safely beside
  v3. The v3 lock becomes meaningful only at P1, which is where the switchover belongs.

**A2. Keep the contract §22 order inside that session:** R1 → R2 → R3 → R4. R1/R2 become the first
steps of the P1 install, in the same sitting.

**A3. P0 acceptance, replacement wording:** the guard exists and behaves — with a fake v3 label loaded,
the node reports `V3_ACTIVE` and refuses to collect (contract §22 R3). The full v3-guard integration
test stays in P2 (plan:361).

---

## B. Corrections to the plan's own text and values

**B1. `unix_socket_directories` must be an absolute path** (plan:96 writes `~/lisa-core/run`).
Postgres does not expand `~`; the contract already specifies `'<abs>/lisa-core/run'` (contract:135).
Write the expanded path into `postgresql.conf` at deploy time.

**B2. The node's Info.plist needs the permission strings** (plan:168 lists bundle id and minimum macOS
only; `grep -r UsageDescription docs/` returns nothing anywhere):

- `NSRemindersUsageDescription`, `NSCalendarsUsageDescription`, `NSContactsUsageDescription`,
  `NSPhotoLibraryUsageDescription`;
- for macOS 14+: `NSRemindersFullAccessUsageDescription`, `NSCalendarsFullAccessUsageDescription`.

Without them macOS terminates the process at the moment it asks for access, instead of prompting. This
also belongs in contract §3/§5 and in §24-2.

**B3. Update the Command Line Tools to 15.x (Swift 5.9+, SDK 14+) before building the node in P0.4.**
The current SDK 13.3 cannot compile the `if #available(macOS 14, *)` branch the mini will need, so
waiting for a compile error means rebuilding mid-migration. Confirm availability with
`softwareupdate --list` on the stage-1 machine (owner authorises the read-only check), then replace the
`swift` pin in the decision log with the verified toolchain and SDK versions.

**B4. Postgres.app: never launch the GUI, and plan the arm64 rebuild.**
The app starts a server of its own on its own cluster; the LaunchAgent must own the only server, so the
app is a source of binaries and nothing else. Add to the plan: pg_bigm must be rebuilt against the arm64
Postgres.app at migration step M3 (and after any Postgres.app update).

**B5. The restic password is a single point of failure → owner action.**
The secrets bundle is Fernet-encrypted with a key derived via scrypt from the restic password
(contract:1150) and that bundle holds `lisa.hmac`, the tombstone key. Losing the password loses the
backups and that key copy. Add "keep an offline copy of the restic password" to the P0 owner list and to
contract §21.1.

**B6. Browsers use the tailnet URL everywhere, including on the stage-1 host.**
The CSRF check requires `Origin == public_url` (contract:622), while the node's `core_url` is loopback
(contract:168, 581). Opening the loopback URL in a browser on the machine that hosts core makes every
approve/decide POST fail. State the one-URL rule in contract §9.1/§9.3 and design §3.4, and note that
acceptance B step 4 needs the iPhone on the tailnet for the reminder link to open.

**B7. Keep hostnames out of the plan's prose.**
The public copy substitutes aliases for real hostnames and addresses (the fleet's export does this);
writing aliases in the plan directly removes a class of leak and makes the two copies diff-clean.

---

## C. Behaviour and sequencing changes

**C1. Start mail collection as soon as core runs.** Collection never sends content to a model, so it
does not depend on the privacy gate (contract:1115 scopes the gate to inference services); the mail
reader lives in core and needs only the iCloud app password in the Keychain. Collecting real mail while
the pipeline is still being built makes acceptance B (real mailbox, next-morning notification) much
easier to run. Contract §7.3's mail row should say so.

**C2. The canary must survive a sleeping machine, and must not fail silently.**
Verified arithmetic: it runs Mondays 05:30 (contract:1127) and is valid for 8 days (contract:1120) — a
7-day cadence with one day of slack, so a single missed Monday closes the gate two weeks later, with no
health code to say so. Required:

1. the check retries hourly until a report lands;
2. the scheduler runs it on wake when the slot was missed (same rule as the brief and the backup);
3. a new `CANARY_OVERDUE` health code fires when the last passing canary is older than the window;
4. the gate never closes without a health message naming the reason.

**C3. `tailscale serve` is a hard prerequisite of P0.5**, not a parallel task: the canary check runs on
the GPU host and reports over the tailnet, and core binds loopback. Contract §24-4 must land first, and
before any acceptance test that needs the gate open.

**C4. Split `lisa-core worker` from `lisa-core serve`** (reviewer's hedge, accepted). Same codebase, same
host for now, its own entry point and LaunchAgent. It costs almost nothing today and keeps the pipeline's
placement open for the mini migration. Recorded as an open design item, not a decision.

**C5. One catch-up rule for maintenance, backup and canary.** design §18 states it generally; make it
explicit per job so no scheduled task can be skipped by a closed lid without being picked up on wake.

---

## D. Verification items to add (contract §24 items the plan does not yet carry)

| # | Item | Phase | Where it goes |
|---|---|---|---|
| D1 | `attributedBody` decode cross-check against `imessage-exporter` on the real `chat.db`, counts only, difference `< 1%` (contract §24-11) | P1 | P1 verification |
| D2 | After a rebuild and re-sign, `LisaNode --selftest` still passes — i.e. the grants survived; if not, the owner re-grants (contract §24-3) | P1 | P1 verification |
| D3 | PhotoKit persistent change tokens and `PHCloudIdentifier` on macOS 13 (contract §24-13) | P3 | P3 verification |
| D4 | EventKit external IDs are stable for reminders, and the marker URL plus notes round-trip (contract §24-12) | P2 | P2 verification |
| D5 | Postgres.app: version pinned, auto-update off, GUI never launched, `pgvector` bundled, `pg_bigm` built against its `pg_config` (contract §24-9 + B4) | P0 | P0.2 / P0 verification |

---

## E. Open items the plan must not pin by assumption

| # | Item | Why |
|---|---|---|
| E1 | **D-11 mail accounts.** The plan states "iCloud only" as a decision on the grounds that D-11 was left empty (plan:45, plan:522). It is an open requirement, not a default, and the owner uses more than one mailbox. Leave `sources.mail.accounts: []` with an explicit TODO until answered. | Owner decision |
| E2 | **Where `lisa-core worker` runs.** Same host as `serve` for now (C4); placement is decided with the storage question when the mini arrives. | Reviewer hedge / owner end state |
| E3 | **Gateway busy probe.** `busy_probe_url` stays null; the day lane yields on observed p50 latency (`backoff_latency_s`), per contract. Revisit only if the fleet adds a probe. | Contract fallback |

---

## F. Explicitly accepted — do not change

- The plan is cleared to start; its phase structure and acceptance tests track contract v2.1.
- Three `[K]` items it closed stay closed: the `:8014` rerank API shape, arm64 cross-compilation from the
  Intel stage-1 machine, the backup-host facts (restic absent, disk free, SSH works).
- The v3 `FSEvents` lesson (`eventPaths` as `char**`, never `CFArray`) stays in the reader.
- `convert_url: null` with `documents.mail_attachments: false` and the attachment path inert — accepted.
- Backup target = the old mini, restic to be installed there (owner approval pending, see §G).
- v3 code archived under `docs/archive/` — accepted; that path is excluded from the public copy.
- Test hygiene: synthetic fixtures only, temp Postgres for tests, `core/tests/fixtures/` gitignored.
- The all-on-one-machine stage 1 is deliberate and temporary (`decisions.md`); late briefs and the
  sleeping-host costs are accepted, not bugs.

---

## G. Definition of done for plan v2

1. Every item in A–E is either applied or explicitly declined with a reason.
2. The decision log distinguishes owner decisions from coder defaults; D-11 is not assumed; Swift/CLT is
   resolved with measured versions.
3. Items needing owner approval are marked as needing it — retiring v3, the GPU-host changes (gateway
   logging off, lisa's own gateway key), and installing restic on the backup host — rather than written
   as routine steps.
4. No real hostnames, addresses or account names outside `config.yaml`.
5. The owner's action list is one session, in this order: trust the signing certificate →
   `tailscale serve` → create the iCloud app password → same sitting: R1/R2, install the P1 node, grant
   permissions. Plus one read-only `softwareupdate --list` on the stage-1 machine (B3).

Nothing in this list is waiting on the coder. The blockers are the owner's D-11 answer, the three
cross-machine approvals, and that one session at a machine.
