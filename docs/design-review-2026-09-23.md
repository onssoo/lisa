# Design review — lisa v3

**Date:** 2026-09-23 · **Reviewed:** `docs/design.md` (682 lines, read in full)
**Reviewer:** fleet agent (Hermes, local 27B via the fleet LLM gateway), at the owner's request.
**Companion:** an external human/model review was requested; this is a second pair of eyes, not a gate.

**Legend:** `[V]` verified this session with real output · `[K]` known platform behavior,
must be re-verified during the build · `[A]` judgment/assumption, not measurement.

---

## Verdict

**Buildable as designed.** The layer chain is sound (each layer regenerable from the one
below), grounding is a hard rule (`src` on every fact/action), and the action lifecycle is
copied correctly from openmuse (idempotency → hash → expiry → CAS → receipt →
`outcome_unknown`). §13 states the OS limits honestly instead of wishing them away.

Six change requests below. Two of them (A1, A3) are cheap and remove the most likely
failure modes; the rest are design-level and best done before phases 4–5.

---

## A. Fix before / while building phase 1 (these bite immediately)

### A1 — Make the nightly chain strictly serial, locked, and define "day" once `[V]`
§4 says extractors run nightly ("default 04:00"); §6 says the digest runs at 04:00 and
consumes the snapshot, which is built from the extractors' output. Same clock, dependent
steps → race.

* One entry point (one job) running `extract → snapshot → digest → brain → dream` under a
  lock file; each step writes a status marker; a failed step stops the chain and leaves the
  day `pending`. **Never update the brain without a fresh digest for that day.**
* Define the day boundary exactly once (e.g. 04:00 cut: a 02:00 message belongs to the
  previous day) and use that rule in snapshot, digest *and* backfill — otherwise the three
  disagree about what "2026-09-23" contains.
* 04:00 is also when the GPU host may be serving a different model stack → retry with
  backoff and a `digest_pending` state rather than dropping the day.

### A2 — Collector pitfalls that silently lose data `[K]`
* **iMessage body:** on recent macOS the text often lives in `attributedBody` (binary
  typedstream) and `message.text` is NULL. Undecoded → messages arrive empty, with no
  error. Verify by cross-checking against `imessage-exporter` output on the same `chat.db`.
* **Reading `chat.db`:** read from a snapshot/copy (`immutable`), because Messages.app
  writes continuously; direct reads intermittently fail with `database is locked`.
* **Office MRU is a rolling window:** measured 85 entries for Word. The design's 1-minute
  poll is what makes this work — a single 04:00 read would miss most of the day.
* **FDA is bound to the binary's cdhash:** an ad-hoc signed collector may need re-granting
  after every rebuild. Add a self-check to the build (try to read `chat.db`; on
  `Operation not permitted`, tell the owner to re-grant) so "suddenly nothing is readable"
  is never a mystery.

### A3 — Pull the delivery surface forward to phase 1.5 `[A]`
Phases 1–3 only record; the owner sees nothing until phase 6/7. Systems like this tend to
rot on disk. Add the smallest end-to-end slice in week 1: `extract → store → snapshot →
one LLM call → one message pushed to the owner`, with approve/deny by reply. It validates
the whole pipeline immediately, makes every later phase observable, and matches the
design's own thesis — the action layer is the product.

---

## B. Design-level changes (before phases 4–5)

### B1 — The brain should be a git repository `[A]`
"Compiled truth" is *rewritten* in place; a bad rewrite loses the previous synthesis (only
the timeline survives). `git init` in `~/lisa/data/brain` + a nightly commit after
digest/dream gives diff, rollback and provenance for every rewrite, with zero new
dependencies. Given the brain is LLM-written, this is the cheapest insurance in the design.

### B2 — Provenance class on proposed actions `[A]`
External text (mail bodies, web pages) is untrusted input flowing digest → facts → brain
pages → actions. v1 (every action approved by hand) is safe. Before any per-verb
auto-approve is enabled, gate it by **provenance**: auto-approve only actions derived from
the owner's own data (calendar, reminders, local files); never auto-approve anything
derived from external text. Same discipline for the brain: mark external text clearly in
timeline entries, and never let a page's compiled truth rest on a single unsourced
external mail.

### B3 — Add "what may never be written" to `RESOLVER.md` (borrowed from compass) `[V]`
compass's `wiki/routing-map.md` declares, per content type, where an agent may write, what
it must **link instead of copy**, and which categories are **never ingested** (journal,
planning, habit, task content — "personal operating data"). lisa's `RESOLVER.md` answers
"where does a page go" but not "what may never be written". Add: (a) human-authored text is
referenced, never rewritten or duplicated; (b) the digest may append timeline entries and
facts, but may not rewrite sections it does not own; (c) nothing unsourced enters a page.

### B4 — One config file `[A]`
Policies currently live in prose/prompt/schema: the domain list, approval policy per verb,
digest focus, push time. compass's `Meta/Compass Config.md` is a single YAML that every
template and dashboard reads, with the rule "change things here; nothing else needs to
move." lisa should have the equivalent (`~/lisa/config.yaml`): domains, per-verb approval
policy, digest focus, push schedule, backfill limits.

### B5 — Backfill strategy + versioning `[A]`
"Digests build day-by-day at 1/day" leaves an unreachable backlog. **Recency-first:** full
narrative for the last ~90 days; older history gets the cheap pass only (entity/fact
extraction feeding the brain, no per-day narrative). Add `digest_version` /
`prompt_version` to `digests`, `facts` and `actions`, so a future model or prompt change
can be re-run and compared instead of silently mixing generations.

---

## C. Optional (cheap, high leverage)

### C1 — Phase 7's viewer can be near-free (with one privacy caveat) `[V]`
compass runs seven dashboards with **zero backend**, via DataviewJS over markdown
front-matter. The brain is already plain markdown → opening `~/lisa/data/brain` as an
Obsidian vault gives timeline, backlinks, search and dashboards for free.
**Caveat:** keep the brain out of a *sync-enabled* vault (iCloud): that would push
collected personal data to a third-party cloud service, which §10 forbids. Storing or copying
it on another **fleet** machine (GPU host, mini) is fine — there is no "must stay on one host"
rule.

### C2 — Optional human layer `[A]`
compass's daily ritual works because it is six 1–10 "Did I do my best to…" questions and
nothing else. If the owner wants it: end the nightly message with **one** question; store
the reply as human-authored content in the brain. One question, not six.

### C3 — Mail source `[K]`
JXA against Mail.app is slow and fragile (Automation permission, Mail must be running,
hangs). Consider IMAP + an app-specific password in the Keychain: full history, UID-based
deltas, no GUI. A nightly JXA delta is acceptable if the mailbox is small. Decision needed.

### C4 — Nightly chain resilience `[A]`
The chain depends on the fleet LLM gateway being up at 04:00 with a capable model. Track
`digest_pending` and retry the same day; never let a day go silently undigested.

---

## D. What I would not change

* The regenerable chain (store → snapshot → digest → brain) and **"record is dumb, digest
  is smart"**.
* Grounding: every fact/action cites `src`; the dream cycle's citation audit.
* The action lifecycle in full — idempotency key → content hash → 30-min expiry → CAS →
  receipts → `outcome_unknown`.
* §13's honesty, especially "macOS has no file-read event stream".

---

## E. Open questions (for the owner, and for the external reviewer)

1. Is "day" cut at 04:00 or at midnight?
2. Brain inside or outside the synced vault? (C1)
3. Auto-approve: everything manual for v1, or `create_reminder` only?
4. Mail: JXA or IMAP?
5. Backfill: how many days of narrative; how far back for facts only?

---

## F. Verification ledger (what was actually run, vs assumed)

* `[V]` Read all 682 lines of `docs/design.md` (this review's basis).
* `[V]` The fleet LLM gateway answers **both** `/v1/chat/completions` and `/v1/responses`
  (HTTP 200) — relevant because some agent frameworks speak the Responses API shape.
* `[V]` The gateway writes **every request body in plaintext** to its debug directory on
  the shared GPU host (measured: 2,000 files / 331 MB; a sampled file 408 KB with 311 full
  messages). The owner has accepted this for his own infrastructure, but §10's wording
  "data never leaves MBP#2" is inaccurate — it should read "…except LLM calls to the
  owner's own gateway".
* `[V]` compass inventory used by B3/C1/C2: `Meta/Compass Config.md` (single YAML config),
  `wiki/routing-map.md` (write rules), 7 DataviewJS dashboards, 16 prompts, 16 templates.
* `[A]` Everything not marked `[V]` above is judgment based on reading the design; the
  collector items marked `[K]` need to be confirmed on MBP#2 while phase 1 is built.
