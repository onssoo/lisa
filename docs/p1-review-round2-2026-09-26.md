# P1 review round 2 — real data is flowing (2026-09-26)

**Applies to:** commit `f805049` (coder: B1–B4 fixes + pipeline unblocks).
**Method:** re-verified live on stage1-host — tests re-run, node rebuilt + re-signed + restarted,
and the collection pipeline observed actually shipping data.

**Verdict: the pipeline is REAL now. raw.events went 0 → 246 (200 imessage / 30 edge / 16 fs),
idempotent, de-duplicated. B1, B3, B4 are fixed and verified. One new defect found (imessage
dates), one claim still open (B2 grant flow untested — needs one GUI run).**

---

## A. Verified fixed (live evidence)

| Item | Evidence |
|---|---|
| B1 iMessage WAL | selftest `reader imessage` PASS (was FAIL); deployed node ships imessage events — the copy fix (skip -shm, open RW, errmsg added) works |
| B3 FSEvents | selftest `reader fs` PASS; fs events land (16 rows) — flag removal + dispatch-queue delivery fixed it |
| B4 SSH test | `71 passed, 1 skipped` — the keychain test now skips under SSH_CONNECTION |
| batch_id collision (found by coder while verifying) | batches now `batch:<sha256>` — old `batch:1` idempotency collision gone; new batches accepted |
| tz NOT NULL (coder find) | events carry tz; core no longer rejects |
| fs roots | fs now watches configured roots, missing roots skipped — no more whole-home scan |

## B. New defect (the only blocker left)

### B5. imessage dates are all 2001-01-01 — Apple absolute epoch not converted

Every imessage row: `day = 2001-01-01`, `occurred_at = 2001-01-01 08:00:00+08`. chat.db stores
`message.date` as **seconds since 2001-01-01** (Apple epoch); the reader is treating the raw
number as a unix timestamp or storing it unconverted. Content is intact (real SMS texts visible:
AlipayHK, roaming-data messages). Fix: `occurred_at = 2001-01-01 00:00:00 UTC + message.date`
(milliseconds variant: `date*1000` first if the values look scaled). Existing 200 rows are
salvageable via one UPDATE or a re-poll with parser_version bump (upsert already keeps the newest
body — make it also refresh occurred_at/day).

Also check: several imessage rows have empty preview bodies — confirm empty-text messages
(delivered receipts, stickers) are intentionally kept or filtered per contract §6.

## C. B2 (grant flow) — built but not yet exercised

coder shipped `LisaNode grant` (GUI-session EK/CN request calls). It has NOT been run in the
owner's GUI session yet, so calendar/reminders/contacts events are still absent from raw.events
(readers report 未获得权限 from the launchd context — note selftest SKIPs from SSH are expected,
that's not a defect). **One run of `LisaNode grant` from a terminal on stage1-host will settle
it.** The fleet can't run it for the owner (GUI dialogs), so it waits for the next owner visit.
(TCC state today: calendar/contacts/reminders rows present via the direct-write workaround +
GUI calendar grant; the launchd reader still sees reminders denied — the grant command is the
supported path and should supersede the workaround.)

## D. Steady state now

- node: loop healthy, ships on change, idempotent batches, 5-min poll cadence.
- worker: scheduler ticks clean, `closed: 0` — days are only closed by the night window; with
  the 2001-01-01 bug the imessage events all land on a bogus day, so nothing reaches the night
  lane yet. Fix B5 and the digest pipeline finally has dated material.

## E. Acceptance for this round

1. B5 fixed → imessage days span the real message history (years, not 2001).
2. Owner runs `LisaNode grant` once → calendar/reminders/contacts events appear.
3. Then the first night window closes a real day and `memory.units`/`digests` have content —
   P1's collection milestone is complete with evidence.
