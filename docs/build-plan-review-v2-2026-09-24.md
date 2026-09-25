# Build-plan v2 — fleet review

**Applies to:** `build-plan-v2-2026-09-24.md` (commit `efc8547`), line references from that revision.
**Verdict:** every change-request item A–G is applied and verified in the text. Five new findings, one
recommendation that removes an owner download from the critical path, and two nitpicks.

---

## What v2 got right (verified, not assumed)

| Change-request item | Where in v2 | Check |
|---|---|---|
| A1 R1/R2 → the P1 install session | P1.0 (341–356); P0 acceptance says v3 keeps running and why (331–335) | Applied, and the reasoning matches §7.4 baselines |
| A2 §22 order inside the session | P1.0 R1→R4 (343–348) | Applied |
| A3 P0 acceptance replacement | P0 verification fake-label `V3_ACTIVE` test (325–327); full guard test stays in P2 (499) | Applied |
| B1 absolute socket path | P0.1 (162–163) with the "Postgres never sees `~`" note | Applied |
| B2 Info.plist usage strings | P0.3 (265–270), incl. the macOS 14 pair | Applied |
| B3 CLT before the node build | P0.3 prerequisite (235–241), decision log (81), contingency (673–676) | Applied, but see F3 |
| B4 GUI never launched; arm64 rebuild | P0.1 (154–156, 168–170); P5 M3 (642–643) | Applied |
| B5 restic password offline copy | P0.5 (304–307), owner list (707) | Applied |
| B6 one-URL rule | header rule (20–24); P0 verification (321–322); acceptance B (518–520) | Applied |
| B7 aliases in prose | alias table (26–29), throughout | Applied, one stray (S2) |
| C1 mail from P0.2 | P0.2 `sources/imap.py` (218–228) | Applied; one citation slip (S1) |
| C2 canary retry / run-on-wake / `CANARY_OVERDUE` | P0.4 (289–293), health codes (201–203) | Applied |
| C3 `tailscale serve` hard prerequisite | P0.4 (278–279, 284–285) | Applied |
| C4 worker/serve split | P0.2 (189–195, 211–217) | Applied; contract §4 needs the fourth plist (F5) |
| C5 catch-up rule per job | P0.2 scheduler (214–217) | Applied |
| D1–D5 §24 verification items | P1 (413–416), P2 (502–503), P3 (601–602), P0 (168–170) | Applied |
| E1–E3 not pinned by assumption | decision log (68–73, 86–87) | Applied |
| fleet small items | Makefile venv (136–138); mirror (59, 153); §24 rows | Applied |

No item was declined, and nothing was quietly dropped.

---

## New findings

### F1. `pytest` is not in the contract's dependency whitelist (compliance)

Contract §1 lists the core dependencies and ends: *"Anything else requires amending this contract."*
The plan's `make test` runs `~/lisa-core/venv/bin/pytest` (v2:143) and the test layout needs a pytest
fixture (v2:665–666). Either way is fine, but it has to be explicit:

- **Preferred:** a separate `core/requirements-dev.lock` (pytest and any test-only dependency) installed
  into the same venv by a `make test-venv` target, and one line in contract §1 saying the runtime
  whitelist is unchanged and dev dependencies live in their own lock.
- Or amend contract §1 to name pytest.

Without this, the first `make test` in P0 either fails or installs something the contract does not
allow.

### F2. Postgres.app: pin the PostgreSQL 16 build, not "latest"

The plan says "Download Postgres.app 16" (v2:59, 153) but the main download of the current release
installs **PostgreSQL 17**; PostgreSQL 16 is a separate asset, and there is also a 533 MB
"all versions" build. Contract §4 hardcodes `…/Versions/16/bin/…`, so a `latest` install would break
the cluster paths and require `pg_bigm` rebuilt for 17.

Verified today at `postgresapp.com/downloads.html`: the current release line (v2.9.6) offers separate
downloads — "with PostgreSQL 16" (113 MB, universal, PG 16.15) — plus the all-versions build.

Action: name the exact asset in the plan ("Postgres.app v2.9.6 with PostgreSQL 16, universal"), record
the installed version in health (D5 already asks for a pinned version — make it a concrete string), and
keep it identical on both stages so the migration stays a dump/restore.

### F3. Do not let the Command Line Tools block P0 (recommendation)

Two things are worth separating:

1. `softwareupdate --list` returning "No new software available" does not prove no newer CLT exists for
   this machine. Before an Apple-ID download, try `xcode-select --install` — it is one command, needs no
   Apple ID, and on some machines it triggers Apple's own CLT install/update path.
2. **The macOS 14 branch is only needed when the node runs on the mini.** Stage 1 is macOS 13.7.8, which
   uses `requestAccess(to:)`; that compiles fine with the SDK already installed. The
   `requestFullAccessToReminders` branch becomes necessary at M-time, and M0 installs a current
   toolchain on the mini anyway.

So the plan's "if the download is blocked, stop" (v2:675–676) is stricter than it needs to be. The
queue-friendly sequence is: try `xcode-select --install`; if that yields nothing, either do the Apple-ID
download now, or build P0.3 with the current toolchain and add the macOS 14 branch during M0/M1 on the
mini. Choosing the second route means recording a one-line deviation from contract §1 (build tools
5.8.1/SDK 13.3, macOS 13 path only) in the plan and in the contract's next revision — a documented
deviation, not a silent one, which is exactly what contract rule 1 asks for.

Also worth recording once measured: the CLT version, its SDK version, and whether `swiftc` still
cross-compiles after the upgrade.

### F4. The canary check's `ops` token has no provisioning step

`ops/inference-canary-check.sh` runs on the GPU host and reports with a `POST /v1/ops/canary-report`
that requires an `ops`-role token. v2 describes the script, its schedule and its retry, but never says
where that token comes from or how it lands on the GPU host. Add: mint it from core
(`lisa-core pair --node gpu-host --role ops`), store it as a 0600 file (or env) on the GPU host, and
include it in the GPU-host approval batch — a token that cannot be revoked cleanly is the one thing
worth being fussy about.

### F5. Add the "doc changes owed" table

v2 marks "(reviewer doc pass)" only twice (v2:270 for the Info.plist, and E1). The change request listed
the full set; without a list in the plan, the contract and design keep drifting from what the builder is
told to do. Suggested rows: contract §1 (F1), §4 (worker plist, absolute socket path), §7.3 (mail may
start before the gate), §9.1/§9.3 (one-URL rule), §3/§5/§24-2 (Info.plist keys), §18 (`CANARY_OVERDUE`),
§19.3 (canary retry + run-on-wake), §20 M0/M3 (arm64 `pg_bigm`, GUI never launched), §21.1 (restic
password offline copy), §10.3 (per-job catch-up); design §3.4 (one URL), §10.3 (catch-up), §21 (P0
delivers `serve` + `worker`).

---

## Small

- **S1.** v2:225 says "contract §24 scopes the gate to inference services". The gate is defined in
  contract §19.3 (and design §8.3); §24 is the verification checklist.
- **S2.** v2:79 still writes `sftp:owner@backup-host:…`. The export sanitises it, but B7 asked for aliases
  in prose — `owner@backup-host` in the doc, the real account name only in `config.yaml`/`ops/`.
- **S3.** `make deploy` uses `git archive <tag>` (v2:144) without saying who tags. Either name a tag
  convention (for example `v4.0.0-p0`) or deploy by commit sha.

---

## Owner-side status

- **D-11** is recorded as the owner's answer (icloud.com, single account) rather than a coder default —
  correct shape. One line of confirmation from the owner closes it for good.
- **Still pending, and correctly marked as pending in v2:** the GPU-host batch (switch the gateway body
  dump off, issue lisa its own gateway key) and installing restic on the backup host.

## Definition of done for v3

1. F1–F5 applied (or declined with a reason in the plan).
2. The doc-change table exists and matches the change request's list.
3. The Command Line Tools question is resolved one way or the other, with the measured version recorded
   — and P0.3 is not blocked by it.
4. Nothing else changes: the applied items stay as they are, and the plan keeps its phase structure.
