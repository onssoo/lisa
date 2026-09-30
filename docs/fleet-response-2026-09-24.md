# Fleet-side response — round 2: measurements, and owner decisions

**Date:** 2026-09-24. **Companion to:** `docs/review-comments-2026-09-24.md`.
This note (a) answers the question addressed to the fleet side, (b) reports what was measured since,
and (c) records the owner's decisions, which the reviewer should treat as fixed inputs.

Confidence tags: `[V]` measured this session · `[K]` still to verify on the target machine · `[A]` assumed.

---

## 0. Owner decisions (fixed inputs)

- **No digest/ask key split.** One key. The main model runs **fixed parameters**; **no parameter tuning at
  this stage — a working end-to-end path comes first.** This supersedes the two-key suggestion in the
  D2 discussion. (See §1.2 for why the split would not have bought anything anyway.)
- **Embeddings** go directly to `:8013`; **the gateway body-dump** is switched off globally. (D1 / D3.)

---

## 1. Answer: can the gateway route any key prefix to a non-local backend?

**No.** `[V]` Measured on the live gateway:

- its upstream list contains **four upstreams, all on localhost** (ports `8888`, `8020` and `8001`);
- `active` is a **single global** selection, not per-agent;
- the agent registry retains **only routing labels** (`match.api_key_prefix`). Its own header comment states
  that per-agent policy and constraint fields were dead configuration and have been **removed**.

Three consequences:

1. **"Pin lisa to the local model" is automatically satisfied.** There is no remote upstream to reach; any
   prefix, known or unknown, lands on the currently active local model. The pin is a formality today.
2. **A key prefix buys telemetry attribution only — not different parameters.** Per-agent parameter
   overrides do not exist gateway-side. This is why the owner dropped the digest/ask split (§0): two keys
   would have produced two labels in telemetry and nothing else. `[V]`
3. **Suggested cheap safeguard (gateway-side):** assert at startup that the active upstream URL is
   loopback/private, and refuse requests carrying lisa's label otherwise — fail-closed. Then if a remote
   upstream is ever added, lisa cannot be routed off-host silently.

Client-side request parameters are likewise ignored: the gateway patches sampling / `max_tokens` /
thinking from its own policy. Record the `model` field the gateway returns on every unit, fact, digest and
page — that is the thing that is actually true.

---

## 2. Photo path — measured, not assumed

`[V]` A real image (900 px, 134 KB) sent to the gateway as an image content part:

```
HTTP 200 · model = qwen3.8-27b · finish = stop · image_tokens = 420
prompt_tokens = 465 · completion_tokens = 2207 (reasoning_tokens = 1248)
```

So the two premises of the photo plan hold: **the gateway passes image parts through unchanged, and the
main model reads them.** Three corrections to fold into the contract:

1. **Cost.** 2,207 completion tokens for a "transcribe" instruction, against the assumed ~300 output
   tokens — 3–7×. Still a rounding error at <10 photos/day against the nightly budget, but the budget
   should be written from measurement, not from an estimate. `[V]`
2. **Quality.** On a dense, text-heavy image the transcription came back with insertions and errors (it
   invented a title and concatenated unrelated strings). Sparse text — an appointment card, a receipt —
   will be considerably easier, but the risk belongs in the document. `[V]`
3. **The grounding rule loses its force on photos — the important one.** `[V][A]` The stored `body` *is*
   the model's transcription, so the verbatim-substring check validates consistency **with the
   transcription**, not with the image. A misread date still passes as a grounded fact. And photo/document
   trust is `third_party`, not `web`, so rule P6 does not stop it → **a wrong reminder can be created.**
   Two candidate patches:
   - extract the visible text with the host OS's built-in OCR (deterministic, on-device, no model) and let
     the model only describe the scene; or
   - require owner re-confirmation for date/time-bearing actions whose only sources are photos, or require
     two independent reads (OS OCR and model) to agree before a proposal is allowed.

---

## 3. Embedding service — API shape `[K]` resolved

`[V]`

- `POST /v1/embeddings`, OpenAI-shaped (`{model, object, usage, data}`), **dim 1024** — matches the schema.
  Note the path **without** `/v1` returns a bare array; call the `/v1` path.
- The query instruction prefix is **accepted**, and it **materially changes the vector**: the same sentence
  with and without the prefix has cosine 0.82.
- "Skipping the prefix noticeably hurts retrieval" should be marked **to be measured**, not asserted. On a
  single query–document pair here the *unprefixed* query scored higher (0.6835 vs 0.6406). One pair proves
  nothing in either direction — recommend 20–50 labelled queries on lisa's own corpus before the convention
  is frozen.

---

## 4. Extending the canary to the direct services — cheap, and worth doing

`[V]` The three direct services' logs contain **no request bodies** today (content matches: 0; what they
log is slot lifecycle and token counts). llama-server only echoes content when verbose logging is enabled —
which is exactly the reason a canary there has value: it detects the temporary switch being turned back on.

---

## 5. D3 evidence, with a measured nuance

`[V]` The gateway's debug dump is active and rolling (2,000 files / 319 MB). Its newest entries are always
**paired** `*_req.json` + `*_raw.sse` — i.e. **streaming requests are captured with full bodies**. A
non-streaming request sent for this note was **not** captured.

Practical reading: keep photo description non-streaming (the contract already does), and switch the dump off
globally anyway; the canary stays as the detector for it coming back. Note that lisa's `ask` is a streaming
endpoint — under the current setting its full prompt and response would be written to disk.

---

## 6. Fleet-side opinion on D4 / D5

- **D5 — agree with waiting for the new machine.** With no backfill, the "rolling sources lose data while we
  wait" argument disappears; it is one migration instead of two, and no extra load on the machine that also
  serves git.
- **D4 — with no backfill, lean toward the always-on machine owning the Apple sources.** "It can only see
  new messages" stops being a drawback, and an always-awake machine removes the need for an executor pool.
  The three iCloud checks cannot be run before that machine exists; run them on day one. Either path is a
  configuration change.
- **One addition:** if that machine signs into the owner's Apple account, record it in the docs as an **owner
  decision**, and state that the core process itself needs no Apple credentials — only the on-device node
  process needs the OS privacy permissions.

---

## 7. `[K]` status after this round

| Item | Status |
|---|---|
| §19-1 node can read the message DB | **Answered** — 35,853 rows; three-file DB (`db`/`-wal`/`-shm`); toolchain present |
| §19-2 Safari `origin` field | **Dropped with the source** — Safari is stale; default off |
| §19-5 Edge journal mode | **Answered** — rollback `-journal`, not `-wal` |
| §19-6 embedding API shape | **Answered** — `/v1/embeddings`, dim 1024 |
| new: gateway passes image parts; model reads them | **Answered** — HTTP 200, `image_tokens` 420 |
| new: do the direct services log bodies? | **Answered** — no content in their logs today |
| §19-7 gateway per-client logging | **Resolved differently** — no per-client mechanism exists; use the global switch (§5) |
| §19-3 signing / permission persistence | Not attempted — needs the app plus the certificate |
| §19-4 EventKit authorization via LaunchAgent | Not attempted — needs the macOS host |
| §19-8 NoteStore / SFL formats | P4 only |
| §19-9 IMAP app password | Needs the credential — owner action |

---

## 8. Small confirmations

- Accepted as revised: the gateway / embedding / reranker / small-model split; Safari default off; Edge
  "copy both" with the corrected reason; **hostnames only in config** (the private copy holds real
  addresses); the pairing and IMAP-pacing patches; IMAP live-lane-only (no backfill).
- **Reranker and the small model as a filter:** agreed. Both are local, and at this scale effectively free.
  Keep the small model to triage, JSON repair, and file-event grouping — never facts, entities, or actions.
- **Document conversion:** agreed, with the hard condition unchanged — the converter must retain nothing (no
  cache, no index, no copy), or forgetting stops being true.
- **Photos via the main model:** fine at <10/day; keep screenshots opt-in and location rounding as written.
- **DuckDB:** not in v1 — agreed; Postgres covers the brief's aggregates and coverage checks at this scale,
  and a second persistent store is one more place forgetting has to reach.
