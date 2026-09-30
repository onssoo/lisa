# lisa — agent context & coordination state

> **Status:** living handoff, seeded 2026-09-30. Public/sanitized copy.
> **Purpose:** let any agent (or reviewer) pick up this project without re-deriving it.
> **Authority order:** `requirements.md` → `requirements-p2-v2.md` → `design.md` → `contract.md` → build plan. This document is *context*, not spec.

---

## 1. What lisa is

A single-user, local-first AI life-digest system. It records the owner's digital life
(messages, mail, files, calendar, reminders, contacts, web), summarises each day into one
page, keeps a compounding memory, and — only with approval — acts on the owner's behalf.

The unit is **the day**. Each day is a page with three states: *preview* (the night before) →
*today* (live) → *memory* (after closing). The book starts at go-live; there is no backfill.

## 2. Who does what (the working model)

| Role | Responsibility |
|---|---|
| **Owner** | The single user. Makes product decisions, authorises OS-level permissions, chooses priorities. |
| **Coder** | Writes code. One commit per logical fix. Reports evidence (machine, command, counts). |
| **Coordinator** | **Re-verifies on the real machine.** Posts a verdict on the issue: pass → close, fail → a rework checklist. Never accepts "done" without reproducing it. |

**The loop:** coder proposes → coordinator re-verifies → close, or reject with specifics.
Work items are issues on the fleet git host; each has one assignee; each state change is
recorded on the issue so the history is auditable.

**The coordinator's standing rules**
1. A claim is not evidence. Reproduce it (`git` diff, real DB rows, real logs).
2. Tests passing ≠ production working. Ask what the test *mocks*.
3. Distinguish *verified*, *assumed*, and *unknown* in every report.
4. Scope discipline: an unrequested change inside a commit is a finding.
5. Sanitize before anything goes public.

## 3. Sanitized topology — current as of 2026-09-30

Roles are split by a single rule: **heavy inference on the GPU host; tiny always-on utilities on
the serving host; the record and the backups on the storage host.** One machine per job, so no
machine is both a test bed and a production dependency.

| Alias | Role | Runs |
|---|---|---|
| `mbp` | workstation / pilot | the owner's agents and the dev loop; a node for sources that exist only there (browser history, local file activity, Office recents). A laptop: it sleeps, so late data is expected and lands as addenda |
| **`m2`** | **serving host** | lisa-core (API), the worker (scheduler), the Apple-source node, **embed `:8013` / rerank `:8014`**, and the fleet-wide **docreader `:50051`** (document parsing for every agent on the tailnet) |
| `gpu-host` | compute / **experiment bed** | the LLM gateway `:9000` (the single entry for all agent LLM traffic) plus the 27B main and the small model. Deliberately carries **no always-on utility services** — they would add OOM risk to experiments |
| `backup-host` | storage | the database, the object/vector store, and the primary backup repo. 8 GB, so it runs no models and no parsing |
| `public-host` | public edge | the owner's public web apps, verified live, with its own daily database backup. **Deliberately not part of lisa's backup strategy** — which is why lisa's offsite copy is still an open gap |
| client | UI | a **tailnet-only PWA**; no public exposure |

**Changes recorded 2026-09-30**
- **embed/rerank moved to `m2`** (from `gpu-host`, 2026-09-27; acceptance-verified: dim 1024,
  cosine 0.99969 against `gpu-host`, identical rerank ordering, byte-identical weights).
- **docreader moved to `m2`** (2026-09-30) and became a **shared fleet service**, not a
  lisa-only component: `m2:50051`, gRPC, reachable by every tailnet machine, no auth (matching
  the previous instance), bound to the tailnet + loopback only. The `gpu-host` instance is
  retired but its image is kept as a fallback. This is the first service lisa *shares* rather
  than owns.
- **One machine was retired.** The second workstation is gone (a company machine, now offline).
  Consequence: the `fs` / `edge` / `office_mru` sources froze, so local file-activity collection
  has to move to `mbp` — which is exactly the per-machine node problem the file-collection
  design has to answer.
- **Parsing left `backup-host`.** It was going to carry Postgres *and* blob storage *and* the
  primary backup *and* a sandbox *and* all parsing on 8 GB. Parsing now runs on `m2`, so
  `backup-host` keeps only the record.

Machine names in the private repo appear here as purpose aliases; the private copy holds the real
values.

## 4. Gate status — the P2 start-gate (all closed 2026-09-30)

| Gate | Meaning | State |
|---|---|---|
| G1 | The stage-2 reader fix merged **and deployed** | ✅ |
| G2 | The stage-2 API/security fixes merged **and deployed** | ✅ |
| G3 | The collector has Full Disk Access | ✅ |
| G4 | Mail collection actually running | ✅ |
| G5 | **At least one real day fully closed** (digest + units populated) | ✅ |

G5's evidence: a real day reached `digested` with grounded `digests` / `facts` / `units` /
`entities` / `pages`, produced by the real nightly pipeline.

## 5. Verified vs not

**Verified working** (reproduced on real data, not from tests)
- The nightly pipeline: snapshot → unitise → map → reduce → brain → dream.
- A **real day closed and produced a genuinely useful page** — it caught a vendor deprecation
  notice, derived the deadline and the migration action, and asked the owner one clarifying
  question. That is the product working, not just the plumbing.
- The day page renders (`/today`), pairing works, health reports real status.
- All four Apple sources report `ok`; mail collects.
- Stable code signing, so OS permission grants survive rebuilds.
- The **shared docreader** parses real Office files end-to-end (docx with a table, xlsx) and is
  reachable from every machine on the tailnet.

**Known open work**
- **Event-queue starvation.** A stream of object-state rows could starve real events (the queue
  drained at ~30 rows per POST because of a 5 MB body cap). A fix landed — events take priority,
  state rows are deduplicated persistently, and cursors are made robust — but it has **not yet
  been re-verified over a full unattended night**. Until then, "events arrive promptly" is
  assumed, not verified.
- **Message-body decoding is only partly fixed.** The modern `attributedBody` container is now
  decoded correctly, but coverage is still partial: the reference implementation decodes ~10 of
  12 messages in a sample day where the shipped reader recovered 3. Real content is still being
  lost.
- **File collection has not started.** Metadata events, hash-checked content upload, and the
  hand-off to the file-brain are designed but unbuilt; the per-machine node question (which
  machine owns which source, and what happens when a laptop sleeps) is unresolved.
- **The web app is early.** No day-browsing, no approval desk, no memory page, no chat — see
  `web-app-design-proposal-2026-09-30.md`.
- **One backup gap is open by decision.** Public-host backups are out of scope, so the
  "one building incident" case is not covered yet.

## 6. Hard-won operational lessons (the expensive ones)

These cost most of a day to learn. **Any agent working here should read this list first.**

1. **Never put a daemon secret in the login keychain.** It locks whenever there is no GUI
   session, and background agents then read nothing — silently. Every daemon secret belongs in
   a `600` file (mail password, LLM gateway key, node pairing token).
2. **OS permission grants are anchored to the code signature.** With ad-hoc signing the identity
   is the binary hash, so *every rebuild* revokes the grants. Sign with a stable certificate, or
   you will re-authorise forever.
3. **Permission requests are attributed to the *responsible process*.** Launching an app binary
   from a terminal attributes the request to the terminal, so the app never gets its prompt.
   Launch it through LaunchServices instead.
4. **A cursor must never run ahead of its data.** A cursor left over from an older database
   silently skips everything — including new records — with no error.
5. **The day row must be created on the first event.** If only *updates* exist, the day never
   closes and the nightly job has nothing to do — while every health check still looks green.
6. **A vivid UI is a data problem, not a rendering problem.** Styling a markdown blob still
   leaves prose. Render objects (cards, events, commitments, source freshness) — see the
   web-app proposal.
7. **Tests that mock the failing component cannot catch its failure.** In this project the
   production bugs were: a locked keychain, an OS API deprecated two versions ago, a bash-4
   construct on bash 3.2, and a stale cursor. Every one of them passed the test suite.
8. **"The image is present" is not "the service is running".** Before building a service, check
   whether one already runs somewhere in the fleet — `docker ps`, not `docker images`. We built
   a document-parsing service for the tailnet while another machine had been serving exactly
   that, healthy, for weeks. A running deployment is also the best specification: take its real
   config with `docker inspect` instead of rewriting it from the docs. And before pulling
   anything over the public network, look for a local copy first.

## 7. How to pick this up

1. Read, in order: `design.md` (architecture) → `contract.md` (the spec) →
   `requirements-p2-v2.md` (what P2 is) → the current build plan → this document.
2. Obey the authority order (§top). Do not code from a requirements doc directly.
3. Check the open issues on the fleet git host — that is the live work queue, not this file.
4. The repository holds **code and design only**. Collected personal data never enters it.
