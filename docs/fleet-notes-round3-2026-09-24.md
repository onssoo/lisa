# Fleet notes — round 3: measured throughput, environment facts, questions before v4.1 / v2.1

**Date:** 2026-09-24. **Audience:** the external reviewer, alongside the contract and the two earlier
fleet notes. Everything here is measured, not assumed; measurements used synthetic content only — no
personal data was involved. Tags: `[V]` measured · `[K]` to verify on the machine · `[A]` assumed.

---

## 0. Owner decisions recorded this round

- **Postgres placement — decided.** For stage 1, the simplest path: the database runs locally on the
  coding laptop (the stage-1 machine), installed without Homebrew. Reason, in the owner's words:
  everything moves to the new machine eventually, so stage 1 should stay simple and the migration is a
  planned cutover, not a reason to build a distributed setup now.
- **Simplification is acceptable in general.** Where a choice exists between a simpler stage-1 setup and
  one that anticipates the new machine, prefer the simple one.

---

## 1. The number requested: decode throughput of the main model

`[V]` Three runs of a synthetic map-shaped call through the gateway, warm and otherwise idle:

```
run 1   prompt 353   completion 1174   wall 28.1 s   41.7 tok/s
run 2   prompt 353   completion 1085   wall 23.6 s   45.9 tok/s
run 3   prompt 353   completion 1082   wall 23.8 s   45.5 tok/s
```

`[V]` Seven days of real traffic, from the gateway's own telemetry (2,871 requests with output > 50
tokens and latency > 1 s):

```
end-to-end output rate   24.1 tok/s      (average output 979 tokens, average latency 41.3 s)
thinking share           ~27% of output tokens
per caller               the coding agent 23.8 tok/s (2,708 requests)
                         unknown prefix   37.7 tok/s
                         others           16 – 41 tok/s
peak concurrency         48 requests inside one minute
```

So: **≈44 tok/s single-stream when idle, ≈24 tok/s end-to-end under real load.** The reviewer's assumed
30–50 tok/s `[A]` holds at the top of the range when idle, and about half that in practice.

## 2. Consequence: keep the bulk nightly, use continuous map as overflow only

Arithmetic against the measured rate: 40–60 map units × ~1,000 output tokens = 40k–60k tokens →
**20–30 minutes at the idle rate, 28–42 minutes loaded**; ten photos add roughly 5–8 minutes. The
05:00–06:30 window (90 minutes) fits with margin.

The night is also the *quietest* period, measured — requests per hour over seven days:

```
02:00 → 1    03:00 → 5    04:00 → 0    05:00 → 0    06:00 → 0
peaks: 19:00 → 351   00:00 → 312   20:00 → 295   (daytime ≈ 100–190/hour)
```

So moving the bulk into the day would place lisa's ~30 minutes of generation inside the hours when the
owner's interactive agents are busiest — the 44 → 24 tok/s difference above is what that costs, on both
sides. **Recommendation: bulk stays nightly; the continuous, quiet-based lane becomes the overflow path**
(units left over from the night, and events that arrive during the day), not the primary path.

**Question for the fleet side, asked here so it can be specified:** the gateway exposes `/stats`. Can it
add a small signal — "an interactive request is in flight / recent p50 latency" — that lisa polls, so
lisa yields rather than relying only on observed latency (`llm.backoff_latency_s`)?

## 3. Environment facts on the stage-1 machine

`[V]` measured on the coding laptop:

```
Homebrew           absent      (and the owner declines it on that machine)
Docker             absent
Postgres           absent      (no server, no client)
Python 3.12        3.12.13     present via uv  (~/.local/bin/uv) — visible in a login shell
ports 5432, 8443   free        (in use: 22, 88, 445, 5900, 7890, 9090, 18488 …)
power              sleep 0, disksleep 0  (currently additionally held awake by screen sharing)
```

Two corrections to the earlier stage-1 advice:

1. **Do not install the python.org build** — 3.12 is already there (uv-managed). The venv can be built
   from it directly.
2. **Postgres has to come from Postgres.app or a prebuilt package**, because there is no Homebrew and no
   Docker on that machine; `pg_bigm` must be compiled against that install's `pg_config`. Compiler
   toolchain is present. `[V]`

## 4. Handover from the existing installation (needs to be in the contract)

The stage-1 machine **already runs an earlier installation of lisa** today: its database and binaries
under the user's home directory, plus two LaunchAgents (a resident collector, and a 04:00 digest job).
The new core and node will live on the same machine, so the contract needs:

1. **An explicit retirement order** — the old agents are unloaded before the new node starts collecting.
   Otherwise two collectors read the same sources, write two stores, and can re-trigger each other.
2. **A decision on the working directory** — reusing the old paths risks mixing old cursors and state
   with the new store. Recommendation: a new directory now, with "delete the old one only after go-live
   is verified" written down.
3. **A note that the privacy permission does not carry over.** The OS grants disk access per executable
   identity, so the new binary must be granted access again; the old grant is useless to it. This is an
   owner-at-the-machine step on day one.

## 5. Postgres placement (decided — see §0)

Documented consequences of the simple choice:

- the database sleeps whenever the laptop sleeps, and with it the API — the design already reports this
  honestly through coverage and health (`NODE_OFFLINE`, `partial` coverage);
- `pg_bigm` is compiled locally against the Postgres.app `pg_config`;
- the move to the new machine is a `pg_dump` / `pg_restore` plus config, already covered by the cutover
  runbook (M0–M6).

## 6. Three small gaps worth folding into v4.1

1. **Photo blobs reach the backups.** Keeping an image while its action is pending is right, but the
   database backup will then contain that image — i.e. it survives *outside* forget's reach until the
   backup retention expires. Either exclude that table from backups, or state it in the forget receipt
   alongside the other retention wording.
2. **Do not migrate the gateway key.** lisa has its own entry at the gateway, so the new machine should
   be issued **its own key**; migrate only the tombstone HMAC key (plus IMAP and backup credentials).
   Otherwise a gateway credential travels between machines as a file.
3. **`stream: false` reduces exposure but is not a guarantee.** "Non-streaming was not captured" is a
   single observation, and the capture condition was not fully pinned down. The real guarantee remains
   switching the body-dump off; so the ordering should be "dump off first, streaming enabled only after
   the canary has passed", not the reverse.

## 7. Division of labour for P0

Fleet-side agent, over SSH on the stage-1 machine (no physical access needed):

- create the venv and install dependencies; compile and run the Swift node; install the user-level
  LaunchAgents; run the pipeline, the tests and the acceptance runs; write the docs.

Owner, physically at the machine (cannot be delegated):

- granting the privacy permission to the new binary; the EventKit and Photos prompts; the three iCloud
  checks; creating the signing certificate in the keychain and trusting it.

**Request to the reviewer:** mark, in the day-one `[K]` list, which items need the owner physically
present. P0 will then be scheduled around that split rather than discovering it mid-build.
