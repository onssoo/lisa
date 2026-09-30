# frontend spec v1 — verification against code and contract

> 2026-09-30 · coordinator. Checks the four assumptions `frontend-spec-v1.md` makes about the existing
> system, so the coder starts from verified facts. **Spec authority is unchanged**: requirements →
> design/contract → spec → code. Where this note finds the spec disagreeing with the *contract*, the
> contract must be updated deliberately — it is not silently overridden.

## 1. Assumption: the web layer is Python + Jinja ✅

Confirmed. `core/lisa_core/web/routes.py` defines the routes and renders Jinja templates
(`base.html`, `today.html`, `health.html`, `pair.html`, `shell.html`); the contract's tree already
lists `web/{ingest,days,snapshot,search,ask,actions,forget}.py`. Today's routes are only
`/health`, `/pair`, `/`, `/today`, `/pending`, `/ask` — and `/pending` and `/ask` are still shell
placeholders. **The client API in the spec is therefore almost entirely new work.**

## 2. Assumption: endpoint names are proposals, to be matched to the contract ⚠️ **conflict**

The contract **already reserves a client API**, and it does **not** use the spec's `/api/*` namespace
(`contract.md` §"client endpoints"). The reconciliation the coder needs:

| Spec proposes | Contract already reserves | Verdict |
|---|---|---|
| `POST /api/ask` | **`POST /ask` → SSE** (`{question, session_id?}`) | ✅ **Already agreed** — same semantics; drop the `/api` prefix |
| `GET /api/health/summary` | `GET /health` | ✅ Reuse, no new route |
| `GET /api/day/{date}` | `GET /days/{day}` → `{state, coverage, digest_md, facts[], snapshot}` **and** `GET /brief?day=` | ⚠️ **Path exists, shape differs.** The spec's `DayPage` (§7.2) is far richer than `digest_md`; decide whether `/days/{day}` is reshaped or `/api/day` supersedes it |
| `POST /api/cards/{id}/mark` | `memory.card_feedback` is a **table**, not an endpoint; `POST /actions/{id}/decide` is the *action* pipeline (a different thing) | ⚠️ **New route needed** — and keep it distinct from `/actions/{id}/decide` |
| `POST /api/input` (the classifier) | maps onto existing `POST /notes` (note) and `POST /ask` (ask) | ⚠️ New classifier route that fans out to the two existing ones |
| `GET /api/days?from=&to=` | — | 🆕 New |
| `GET /api/source/{ref}` | — (the contract embeds `sources[]` inline on action items) | 🆕 New |
| `POST /api/cards/{id}/undo` | — | 🆕 New |
| `POST /api/memory/proposals/{id}` | — (the contract has `PUT /entities/{id}/pinned`) | 🆕 New |

**Two decisions for the owner/coder:** (a) does the client API get an `/api` prefix (spec) or stay flat
(contract)? (b) is `GET /days/{day}` reshaped into `DayPage`, or retired in favour of `/api/day`?

## 3. Delta A: the digest must emit **JSON** — this **changes the contract** ✅ real

The contract currently specifies markdown, in three places:
- §digest output: *"Output: markdown with exactly these sections: `## 今天`, `## 重要的事`, …"*
- `memory.digests.summary_md text NOT NULL` (§ data model)
- §13/§archive: the nightly job writes `archive/YYYY/MM/DD.md` as plain markdown.

The spec §7.3 inverts this: the digest writes **JSON** (review / cards / preview, with citations), and
the markdown archive is **rendered from that JSON**. The frontend must not parse markdown headings.

**Consequence:** `contract.md` must change (the digest output contract + how `summary_md` relates to
the JSON), and the archive becomes a *rendering*, not the source of truth. Recorded in `decisions.md`.

**Why it matters:** today the only structured output is prose, so the UI would have to parse headings —
which the spec forbids, correctly, because headings are cosmetic and citations inside prose are not
machine-checkable.

## 4. Delta B: Ask is **POST** ✅ — the spec matches the contract; my proposal was wrong

The contract already reserves `POST /ask` with the question in the body, and the reason is the same one
the spec gives: *private questions must not end up in URLs, logs or browser history*.
`web-app-design-proposal-2026-09-30.md` had written `GET /api/ask?q=…`; that is **retracted**.
Recorded in `decisions.md` so it is not reintroduced.

## 5. Implementation notes for F0/F1

- **F0 is the true dependency, not P2.1.** Today `memory.digests` holds markdown only, so `/api/day`
  cannot produce `review`/`cards` until the digest emits JSON. The spec's claim that F1 "does not wait
  for the P2.1 tables" holds **only if** F0's JSON path fills `review`/`cards` from the digest JSON and
  `record` from `raw.events` — which is exactly what §7.3 specifies. Good design; just don't skip F0.
- **Cards vs actions.** The spec's card kinds are `commitment | prep | reply | change | health`; the
  contract's `memory.cards.kind` is `{commitment, health, change, prep}` — **`reply` is new**. Decide
  whether `reply` is a fifth kind or a `prep` subtype before writing the enum.
- **`/pending` and `memory`** are currently shells, so F3/F5 start from placeholder pages.
- **The digest already reaches the DB** (`memory.digests.summary_md` is populated for real days), so F0
  has real data to convert — no fixture-only development needed.

## 6. What the spec gets right that earlier docs did not

Worth stating, because it should survive review:
- **Client does all relative time** (R1/R2) — matches T1/T2 and the contract's §12.3 V10.
- **The digest emits structure, not prose** (§7.3) — the single change that makes every later feature
  cheap, including citation-checking (X12) and the L1 action registry.
- **Honest empty states** (R9/§4.4) — a lisa that does not backfill will have thin pages, and pretending
  otherwise would make the UI lie on day one.
- **R10 "lisa drafts, the owner sends"** — the correct L0 posture, consistent with the orchestrator
  roadmap's approval boundary.
