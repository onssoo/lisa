# Plan review, round 2 — the reviewer's response, and the fleet's verification

**Date:** 2026-09-24. The external reviewer read `build-plan-v41-2026-09-24.md` (commit `a19b0de`),
`build-plan-review-2026-09-24.md` and `decisions.md`, and answered. His response is reproduced verbatim
below, followed by the fleet's verification of every checkable claim and the resulting change list.

---

## The reviewer's response (verbatim)

I read the new files: `build-plan-v41-2026-09-24.md` (about 540 lines, commit `a19b0de`),
`build-plan-review-2026-09-24.md`, and `decisions.md`, plus the three new comments on issue #1. The raw
file cut off in the last few lines of the plan's "Assumptions & contingencies" section, so I read
everything except its last line or two.

## Where things stand

The plan follows contract v2.1 phase by phase: P0 foundation, P1 collection, P2 the doctor's-email test,
P3 memory/forget/photos, P4 completion, P5 the mini migration runbook. The fleet review says it's cleared
to start. The plan also settles three unverified items along the way: the reranker's API format is
confirmed, building the Mac app for both Intel and Apple chips works on MBP#2, and there's a fallback for
MBP#2 having Swift 5.8.1 when the contract asks for 5.9+.

Nothing is waiting on the coder. It's waiting on you:

- **D-11 (mail accounts):** still open.
- **Three approvals for work on other machines:**
  - stopping the two v3 background agents on MBP#2;
  - on the GPU host, turning the gateway's request logging off and issuing lisa its own key;
  - installing restic (the backup tool) on the backup host.

`decisions.md` records that the all-on-MBP#2 setup is temporary, for fast development only. It also
records your end state: storage on the mini, and the processing pipeline on the AI host.

## I agree with the fleet review, with one correction

Moving the v3 shutdown to the day v4 collection starts is right. But moving that step alone won't close
the gap. The plan installs the v4 Mac app in P0.4, and it refuses to run while v3 is active. Yet in P0
the app has no data readers; those only arrive in P1. So "v3 agents gone" also has to move out of the P0
acceptance list and into P1. The P0 app can safely run next to v3, because it collects nothing.

The first canary check (which proves the GPU host isn't logging lisa's data) needs `tailscale serve` set
up first. So it's a hard prerequisite for P0.5, as the review says.

## Problems I found that neither document mentions

1. **Start mail collection earlier.** Collecting data never touches the models, so it doesn't need the
   privacy gate. The mail reader runs inside core and needs only the app password. Starting it as soon as
   core runs means real mail piles up while the summary pipeline is built. That makes the P2 test with
   your real mailbox much easier.

2. **The weekly canary conflicts with a sleeping laptop.** It runs Mondays at 05:30, and the gate closes
   if no check passed in the last 8 days. With MBP#2's lid closed on a Monday morning, the report can't
   get through. The gate then closes and night processing stops without any error. Fix: the GPU host
   retries every hour until the report lands, or core asks for a check when it wakes. Backup, cleanup and
   canary should all share one "catch up on wake" rule, not just backup.

3. **`~` in the Postgres config won't work.** Postgres doesn't expand `~`. So
   `unix_socket_directories='~/lisa-core/run'` needs a full path, written in at deploy time.

4. **The app's Info.plist is missing the permission text.** The plan lists only the bundle ID and minimum
   macOS. On macOS, an app that touches reminders, calendars, contacts or photos without the matching
   description text is terminated instead of shown a permission prompt. It needs
   `NSRemindersUsageDescription`, `NSCalendarsUsageDescription`, `NSContactsUsageDescription` and
   `NSPhotoLibraryUsageDescription`. For macOS 14 or later (likely on the mini) it also needs the two
   `…FullAccessUsageDescription` keys.

5. **Upgrade Swift now rather than waiting for an error.** The build tools on MBP#2 (SDK 13.3) don't know
   the macOS 14 permission call. So the app can't include the `if #available(macOS 14, *)` branch it will
   need on the mini. Updating the Command Line Tools to the 15.x series (Swift 5.9+) before P0.4 avoids a
   rebuild in the middle of the migration. As far as I know, Ventura 13.5 or later supports them, but
   check with `softwareupdate --list`.

6. **Browser login on MBP#2 itself.** The security check compares the browser's `Origin` with
   `public_url`. Opening `http://127.0.0.1:8443` on MBP#2 will fail every approve or decide click. The
   docs should say to use the tailnet URL on every machine, including MBP#2. Also, test B step 4 needs
   the iPhone on the tailnet, or the reminder link won't open.

7. **pg_bigm and Postgres.app.** `make install` writes into the app bundle, which Postgres.app updates
   will wipe out. That's already covered by turning off auto-update. Two additions: never let the
   Postgres.app window start its own server alongside the LaunchAgent one, and note that pg_bigm must be
   rebuilt for Apple chips at migration step M3.

8. **The restic password protects everything.** The backup of secrets is encrypted with a key derived
   from it. Lose it, and the backups and the tombstone key are gone. You should keep an offline copy, and
   it should be on your P0 action list.

## About the "processing on the AI host" end state

I'd push back a little. Your reason was that the brief should be ready when you open the lid. The
always-on mini already gives you that, because the heavy model work already runs on the GPU host through
the gateway. Moving the pipeline itself there would mean opening Postgres to the network, which is
currently local-only by design. It would also put a second machine in charge of personal data.

A cheap step now keeps the option open without that cost. Split the pipeline into its own
`lisa-core worker` process, separate from `lisa-core serve`, even though both run on MBP#2. Where it runs
can then be decided when the mini arrives, as `decisions.md` suggests.

## What to answer now

Answer D-11 and give the three approvals; that's the only blocker. I'd also group everything that needs
you physically at a machine into one session:

- trust the signing certificate;
- set up `tailscale serve`;
- create the iCloud app password;
- on the same day: stop v3, install the P1 app, and grant its permissions.

Doing it in one session also shrinks the collection gap as far as it can go. Issue #1 could then be
closed or retitled, since its five decisions have been answered.

---

## Fleet verification of every checkable claim

Every claim below was checked against the documents in this repository. Line numbers are from
`docs/contract.md` / `docs/design.md` / `docs/build-plan-v41-2026-09-24.md` as of commit `2e877b4`.

| # | Claim | Evidence | Verdict |
|---|---|---|---|
| — | "v3 agents gone" must move from P0 acceptance to P1 | plan:212 lists it under P0 acceptance; P0.4 (plan:145–172) defines no readers; the readers are P1.1 (plan:219+) | **Correct**, and the reason is right: the P0 node collects nothing, so it can run beside v3 |
| 1 | Mail can start before the gate | contract:512 puts mail in core as a live-lane IMAP reader; contract:1115 scopes the gate to "no real event content to any inference service" | **Correct** — collection is not inference, so the gate does not apply. Needs only the app password in the Keychain |
| 2 | The canary conflicts with a sleeping laptop | contract:1127 (Mondays 05:30) vs contract:1120 (must be reported within 8 days); catch-up-on-wake exists only for the brief (contract:695) and backup (contract:1145); design:482 states the general rule but the canary is not named | **Correct, and the arithmetic is worse than it looks**: a 7-day cadence with an 8-day window leaves exactly one day of slack, so a single missed Monday closes the gate two weeks later — silently, since nothing raises a health code for an overdue canary |
| 3 | `~` in `postgresql.conf` | contract:135 already reads `unix_socket_directories = '<abs>/lisa-core/run'`; the `~` appears in plan:96 | **Correct in substance, wrong about where**: this is a plan typo, not a contract gap |
| 4 | Info.plist permission strings | `grep -r UsageDescription docs/` → no hits anywhere in the design, contract or plan | **Correct** — the plan only specifies bundle id and minimum macOS (plan:168). Missing usage strings kill the process at the moment of the request instead of prompting |
| 5 | Upgrade the Command Line Tools before P0.4 | plan:48 and plan:525–527 deliberately wait for a compile error | **Direction is right.** The machine is Ventura 13.7.8, so the 15.x tools should be installable (Xcode 15 requires 13.5+) — `softwareupdate --list` on that machine settles it |
| 6 | Origin check vs the loopback URL | contract:622 requires `Origin == public_url` on browser POSTs; contract:168/581 give the node `core_url` = `http://127.0.0.1:8443`, which a browser on the same machine will otherwise be tempted to use | **Correct** — every browser uses the tailnet `public_url`, on all machines. Test B step 4 also assumes the iPhone is on the tailnet |
| 7 | pg_bigm vs Postgres.app | contract:129–141 covers the compile, the `pg_config` path and disabling auto-update; nothing says the GUI server must not start its own cluster, and nothing says to rebuild for arm64 at M3 | **Correct on both additions** |
| 8 | The restic password is a single point of failure | contract:1150 derives the bundle key from the restic password via scrypt, and the bundle holds `lisa.hmac` (the tombstone key) | **Correct** — losing it loses the backups *and* the tombstone-key copy. It is not on any owner action list |

## Change list

**contract.md**

- §4: add that `~` is expanded by the config loader, never by Postgres — the socket path is written
  absolute at deploy time.
- §9.1/§9.3 + §5: every browser uses `public_url` (the tailnet URL), including on the stage-1 host; the
  loopback address is for the node and CLI only.
- §7.3 (mail row): note that mail collection may start as soon as core runs and does not depend on the
  gate.
- §3/§5 + §24-2: the node's Info.plist must carry `NSRemindersUsageDescription`,
  `NSCalendarsUsageDescription`, `NSContactsUsageDescription`, `NSPhotoLibraryUsageDescription`, and for
  macOS 14+ the two `…FullAccessUsageDescription` keys.
- §19.3: the canary retries hourly until the report lands and also runs on wake; an overdue canary raises
  a health code (new `CANARY_OVERDUE`) instead of closing the gate silently.
- §20 M0/M3: `pg_bigm` is rebuilt against the arm64 Postgres.app on the mini; the Postgres.app GUI must
  never start a second server.
- §21.1: the restic password gets an offline copy — an owner action in P0.
- §3/§4: add the worker process (`lisa-core worker`) to the layout and the LaunchAgents, per the
  reviewer's hedge below.

**design.md**

- §10.3: canary and maintenance share the backup's catch-up-on-wake rule.
- §3.4: the one-URL rule (tailnet everywhere, including the stage-1 machine); note that test B step 4
  needs the iPhone on the tailnet.
- §21 (Phases): P0 also delivers the split between `serve` and `worker`, both on the same host, so the
  pipeline's placement is decided at migration time rather than now.

**build-plan (coder)**

- Move R1/R2 out of P0.1 to the P1 node-install day, and drop "v3 agents gone" from P0 acceptance.
- P0.2: absolute socket path; never launch the Postgres.app GUI.
- P0.3/P1: start the IMAP reader as soon as core runs (needs the app password; independent of the gate).
- P0.4: add the Info.plist usage strings; update the Command Line Tools to 15.x before building the node,
  so the `if #available(macOS 14, *)` branch compiles.
- P0.5: canary retry-until-reported + run-on-wake + `CANARY_OVERDUE`.
- P0.6: restic password offline copy as an owner action.
- Add the four contract §24 verification items to the phase acceptance lists: `attributedBody`
  cross-check `< 1%` (P1), grants surviving rebuild and re-sign (P1), PhotoKit change tokens (P3),
  EventKit external ID stability and marker round-trip (P2).

## The reviewer's hedge on "processing on the AI host"

Accepted, and recorded as an open design item rather than a decision: split the pipeline into
`lisa-core worker`, separate from `lisa-core serve`, both running on the same machine for now. It costs
one entry point and one LaunchAgent, keeps the option open, and avoids the two costs of moving core to
the GPU host early — opening Postgres to the network, and putting personal data on a second machine.
Where the worker runs gets decided when the mini arrives, together with the storage question.

## Blocked on the owner

1. **D-11** — which mail accounts lisa collects.
2. **Three approvals for work on other machines:** stopping the two v3 agents; on the GPU host, switching
   the gateway's request logging off and issuing lisa its own key; installing restic on the backup host.
3. **One session at a machine**, in this order: trust the signing certificate → `tailscale serve` →
   create the iCloud app password → same day, stop v3, install the P1 node, grant its permissions. One
   session also shrinks the collection gap as far as it can go.
4. **One read-only check:** `softwareupdate --list` on the stage-1 machine, to settle item 5.
