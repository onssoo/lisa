**Fleet-side review of the build plan** — `docs/build-plan-v41-2026-09-24.md` (540 lines, commit `a19b0de`).

Read against `design.md` v4.1 and `contract.md` v2.1, section by section. Verdict: **the plan is
contract-faithful and can start.** Phase/acceptance structure tracks the contract, the pins for
contract-open values are declared in a decision log (that is the right shape), and the `[K]` discipline
is kept (`requestFullAccessToReminders` on macOS 13, `attributedBody`, PhotoKit tokens are all left as
verify-then-write-back).

## What the plan closes

| Item | Result |
|---|---|
| `:8014` rerank API shape (contract §24-15, half) | `POST /v1/rerank {model, query, documents}` → `{results:[{index, relevance_score}]}` — verified |
| Universal binary on the Intel stage-1 machine | `swiftc -target arm64-apple-macos13.0` cross-compile verified → the risk is gone |
| Swift version | machine has 5.8.1, contract asks 5.9+; plan proceeds and escalates CommandLineTools only if a compile error demands it |
| Backup host | restic absent, `/` has 301 GB free; SSH works |

It also carries the v3 `FSEvents` lesson (bridge `eventPaths` as `char**`, not `CFArray`) into the new
reader, which is the kind of institutional memory that is worth keeping.

## Three things the owner must decide, not the coder

1. **D-11 is pinned by assumption.** The plan states mail is the owner's iCloud account only, on the
   grounds that D-11 was left empty and the default is the primary account. D-11 is an open decision,
   not a default: the owner has other mailboxes in active use, so "iCloud only" may simply be wrong.
   Please treat it as open until the owner answers. It is a one-line config change either way.
2. **Three cross-machine operations are written as routine steps but need explicit owner approval**
   (fleet rule: no starting/stopping services or writing to other machines without it):
   - stopping the two v3 LaunchAgents on the stage-1 machine (R1);
   - on the GPU host: switching the gateway body dump off, and issuing lisa its own gateway key;
   - installing restic on the backup host.
3. **Confirm the plan's read of D3.** The plan has the fleet switch the dump off first and the owner
   confirm afterwards. That is the right order; both steps need the owner's explicit go-ahead, and the
   the GPU host side will show the diff before touching the running service.

## One sequencing change I recommend

**The collection gap.** By design nothing is backfilled, and every baseline is "wherever the source
stands when the reader first runs" (contract §7.4). So everything that happens between R1 (v3 retired)
and the first v4 node run is never collected — not buffered, never read. With a multi-day P0, that is a
real hole in mail, messages, files and photos.

Recommendation: move R1/R2 out of P0 into the day the P1 node is actually installed and started, so the
window collapses into a single session. Nothing in contract §22 requires R1 to be in P0 — it only
requires the order R1 → R2 → R3.

Related ordering point: **P0.5's canary cannot report until `tailscale serve` exists** (§24-4), because
core binds loopback and the canary check runs on the GPU host. §24-4 is therefore a hard predecessor of
the first canary, and of every acceptance test that needs the gate open.

## Small items for the plan

- Postgres.app is downloaded from a GitHub release URL. Direct GitHub downloads are unreliable on this
  network; the fleet mirror should be used (or expect retries).
- `uv pip sync core/requirements.lock` must point at the venv explicitly (`--python
  ~/lisa-core/venv/bin/python`, or export `VIRTUAL_ENV`), otherwise uv resolves against the system
  interpreter.
- Contract §24 verification items that are not yet in the plan's acceptance lists: the `attributedBody`
  decode cross-check `< 1%` (P1), grants surviving a rebuild and re-sign (P1), PhotoKit change tokens
  (P3), EventKit external ID stability plus the marker round-trip (P2).
- Public copy only: one sanitised line reads ``MBP#2 = `stage1-host` (stage1-host)`` because the
  hostname and its address were both replaced. Cosmetic.

## Status

The plan is published here for review: `docs/build-plan-v41-2026-09-24.md`. The private source of truth
holds the real hostnames and addresses; this copy uses aliases only (design.md, "Public copy").

If anything should change before P0 starts, now is the cheapest moment. Otherwise the fleet waits on the
owner's answers for D-11 and the three approvals above.
