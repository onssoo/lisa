# Comments on `contract.md` v2 — infra reality, verified facts, open decisions

**Author:** the owner's assistant (fleet-side). **Date:** 2026-09-24.
**Audience:** the external reviewer, alongside `contract.md` and `design.md`.

This note does three things:

1. records **what infrastructure actually exists** in this fleet today, because the contract's
   LLM section assumes things that do not hold (and misses things that do);
2. states what I **verified by measurement** this session versus what is still assumption;
3. lists the **decisions only the owner can make**, so the reviewer can treat them as inputs
   rather than as settled requirements.

Confidence tags: `[V]` verified this session · `[K]` must be verified on the target machine ·
`[A]` assumption carried from the contract.

---

## 1. Infrastructure actually available

### 1.1 LLM side

| Endpoint | What it is | Status / notes |
|---|---|---|
| `llm-gateway:9000/v1` | **the only permitted LLM entry for agents** — OpenAI-compatible chat proxy. Exposes **one** model id: `auto`. Records telemetry. | `[V]` `/v1/models` returns exactly `{"id":"auto"}`. **No embeddings route**: `POST /v1/embeddings` → 404. |
| `:8013` | **embedding** — Qwen3-Embedding-0.6B, **dim 1024**, 8,192 tokens/slot, resident, GPU. | `[V]` documented and used today by other local tools. `dim 1024` is exactly the `embed_dim` the contract specifies. |
| `:8014` | **reranker** — Qwen3-Reranker-0.6B, resident. | available; the contract's `search.py` (bigm + vector + RRF) does not use it. |
| `:8017` | **small agentic model** — a 2B model with strong tool-calling. | could serve cheap local steps (unit classification, JSON repair, entity-resolution assist) without the main model. |
| main model | **27B, multimodal**, locally served. | serves digest/ask today. Multimodal matters: it can read images/PDFs directly, which the contract does not exploit (it treats photos as metadata only). |

**Two structural facts the contract should absorb:**

- **Model routing is by client identity, not by model name.** The gateway exposes a single id
  (`auto`); per-agent behaviour (sampling, `max_tokens`, thinking/effort) is **patched
  server-side** from the gateway's own agent rules, keyed on the API-key prefix of the caller.
  Client-side parameters do not win. So the contract's design — three model aliases
  (`lisa-digest` / `lisa-ask` / `lisa-embed`) plus "never use `auto`" — is the wrong lever:
  the correct construction is `auto` plus the project's **own agent entry / key prefix**, with
  any per-role differences expressed there. `[V]`
- **Auxiliary services are outside that routing policy.** The fleet rule that "all agent LLM
  traffic goes through the gateway" explicitly exempts the embedding / reranker / small-model
  services, which are called directly. So pointing the contract's `lisa-embed` at the gateway is
  a bug in the contract, not a policy violation to design around. `[V]`

### 1.2 Server machine side (the mini)

The owner states the server machine will carry, in addition to lisa:

- **Postgres** (already the fleet's service-host pattern);
- **DuckDB**;
- the **fleet git host (the fleet git host)**;
- **SQLite** (can also be installed);
- a **file-ingest workflow**: many input formats converted to markdown.

Other stacks are **open** — the stack choices in the contract are not a requirement.

**Why this matters to the review:** the contract chose Postgres + `pg_bigm` + `pgvector` and then
spent effort designing around SQLite limits; the server is in fact a Postgres host, so the
Chinese-search problem is already solved there. DuckDB on the same box is a good fit for
analytical passes over the event store (daily aggregates for the brief, coverage audits) without
loading the transactional DB. And an existing "any format → markdown" pipeline is directly
relevant to lisa's file and document handling, which the contract reduces to metadata-only
`fs` events.

---

## 2. What I verified this session (evidence)

| # | Claim / question | Result |
|---|---|---|
| 1 | Contract's LLM hostname resolves | **No.** The name used in the contract does not resolve on the client machine; the real address is what it does not name. Needs either a resolvable address or a hosts entry — in the private copy; the sanitized alias is fine for the public copy. `[V]` |
| 2 | Gateway can serve embeddings | **No** — 404. Embeddings must target `:8013` (dim 1024). `[V]` |
| 3 | Gateway exposes the aliases the contract requires | **No** — only `auto`. `[V]` |
| 4 | Gateway logging policy can be set per client | **No such mechanism**, but see §3.4 — the global body-dump is documented and cheap to disable. `[V]` |
| 5 | `[K]`-1 — can a node read the iMessage DB? | **Yes.** From a plain shell session on the collector laptop: `35,853` messages read; the DB ships as three files (`db`, `-wal`, `-shm`) — the contract's "copy all three" is correct. `sqlite3`, `swiftc` and CommandLineTools are present, so building the Swift node there is feasible. `[V]` |
| 6 | `[K]`-5 — does the Edge history DB use `-wal`? | **No** — it uses a rollback `-journal` on the machine I checked. The contract's "copy `-wal` *and* `-journal`" is therefore right, for a different reason than assumed. `[V]` |
| 7 | Safari as a source | The Safari history DB is readable but **stale — last write months ago**; the real browser is Edge. The contract enables `safari` by default and asks for a `[K]` investigation of its `origin` column; that effort is likely wasted. `[V]` |

Remaining `[K]` items from §19 are hardware- or credential-gated and I did not attempt them:
they need the new server machine (`[K]`-4), the signing certificate (`[K]`-3), the mail app
password (`[K]`-9), or are P4-only (`[K]`-8).

**One caveat on #5:** a shell session reading the DB does **not** prove a launchd-launched app
bundle can. Per-process identity decides TCC. That check still has to happen inside the node app.

---

## 3. Substantive comments on the contract

### 3.1 The LLM section must be rewritten around the real gateway (see §1.1)

Three aliases and "never use `auto`" cannot work as written. Concretely: `lisa-embed` → `:8013`;
`lisa-digest` / `lisa-ask` → `auto` with the project's own agent identity, with any parameter
difference expressed in the gateway's agent rules. If the owner prefers distinct behaviours per
call type, that is a gateway policy entry, not a model alias.

### 3.2 Backfill has no build spec

`config.yaml` carries `full_digest_days`, `facts_since`, `mail_since`; propose-rule **P2** says
backfilled days never propose actions — but **no section specifies how backfill runs**: how many
days per night, in what order, when it is considered finished, and how it interleaves with the
nightly digest of recent days (§9.3 only covers "today and recent"). For a first run over 90 days
of history plus IMAP since 2024, this is the difference between a bounded job and an unbounded one.

### 3.3 Two smaller gaps

- **No pairing/login endpoint.** §8 says the browser enters a token once and then uses an
  HttpOnly cookie, but the endpoint table has no route that performs that exchange.
- **IMAP backfill has no paging/throttling spec.** iCloud IMAP throttles and limits concurrent
  connections; the contract should state batch size, pacing and resumption.

### 3.4 The gateway body-dump is documented, and closing it is cheap — but the contract should say so

The fleet's docs record that the gateway currently **dumps full prompt and response bodies** to
disk (rolling, ~2,000 files), marked in-code as temporary. Health code `GATEWAY_POLICY` and the
weekly canary in §16 are exactly the right instruments for this — they are not hypotheticals.
The correct remedy is to switch the dump off globally (a one-line flag plus a restart of the thin
proxy container — seconds, no backend reload), **not** to add per-client exemptions, which the
gateway does not support. The contract should state that remedy and the owner's decision on it.

### 3.5 A prerequisite that is not in the contract: the server machine

§1 requires the core to run on **macOS 14+, arm64**. That machine does not exist yet — the
current service host is an old Intel box running Linux (which is where the git host lives today).
Several `[K]` items and all of the core deployment depend on the new machine arriving. The
contract should list it as a prerequisite, together with what happens to the existing git host
later.

### 3.6 Node assignment for Apple-account sources

§4 assigns iMessage / Reminders / Calendar / Contacts / Notes / Photos to the **server** node.
That silently requires the new server to be signed into the owner's Apple account. This is a real
decision, not an implementation detail. The alternative — keep those sources on the laptop that
already holds the history (it has 35,853 messages) — costs nothing and needs no Apple account on
the server. Either way the contract should say it explicitly rather than bake it into a config
listing.

### 3.7 What v2 fixed (for the reviewer's context)

The earlier external review raised: proposals that expire before they can be read; an undefined
idempotency key; the system reading its own writes back in; a LaunchAgent that cannot legally
carry both `KeepAlive` and a calendar interval; "every layer is regenerable" being false for the
LLM stages; prompt injection; and no plan for Chinese text. v2 addresses essentially all of them,
and improves on some of them:

- proposal expiry split (72 h for digest proposals, bounded by the due date) — `[V]`;
- idempotency anchored on the **earliest source event**, not on model wording — `[V]`;
- self-read prevention via a table of objects lisa itself created, plus FSEvents exclusions — `[V]`;
- digest is now "every finished day that has no digest yet", which makes catch-up and backfill the
  same code path — `[V]`;
- facts must cite a **verbatim substring** of the source (normalization defined) — `[V]`;
- entities: the LLM may only *propose* merges — `[V]`;
- `pinned` is never read or written by the compiler — `[V]`;
- forget cascades through every layer, with tombstones stored as HMACs, and the receipt tells the
  owner how long the data survives in backups — `[V]`;
- the database is canonical, markdown is a rendered view — `[V]`;
- coverage is generated by code, not by the model — `[V]`.

The remaining structural question the reviewer may want to weigh in on is §3.1 (LLM lever),
§3.2 (backfill), and §3.6 (which machine owns the Apple sources) — the rest of this note is
infra bookkeeping.

---

## 4. Decisions the owner has to make

| # | Decision | Options |
|---|---|---|
| D1 | Embeddings | point at `:8013` (dim 1024, matches the contract's `embed_dim`) **or** drop vector search (`embed_model: null`) |
| D2 | Chat-model lever | `auto` + the project's own agent entry in the gateway (**recommended**) **or** define per-role behaviours as gateway policy; model aliases as written are not available |
| D3 | Gateway body-dump | switch it off globally so `GATEWAY_POLICY` is meaningful **or** keep it and drop that health code |
| D4 | Apple-account sources | server signed into the owner's Apple account **or** keep them on the laptop that already holds the history |
| D5 | Server machine | confirm the new arm64 macOS host as a prerequisite, and the fate of the existing git host |
