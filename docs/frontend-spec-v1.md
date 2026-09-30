# lisa web app: frontend spec v1

> Status: spec for review. It builds on `web-app-design-proposal-2026-09-30.md`. Order of authority: requirements → design/contract → this spec → code. Where this spec and the proposal disagree, this spec wins. Changes to the proposal are marked **[changed]**.

## 0. What we are building

lisa's web app is **a book of days, not a dashboard**. Each day is one page. The main device is an iPhone running Safari as an installed PWA, reached over the tailnet only. It is a single-user app for the owner alone.

**The KPI:** in the morning, on the phone, the owner can read yesterday's review and today's cards in 30 seconds, without scrolling. Any design choice that competes with this loses.

## 1. Rules the UI must never break

| # | Rule | What it means for the frontend |
|---|---|---|
| R1 | Now is the reference point (T1) | Relative times ("in 2 days", "3 h ago", "past") are worked out when the page is opened and updated live. They are never stored. |
| R2 | Models never write relative time (T2) | The API only returns absolute ISO-8601 times with offsets. The client does all relative formatting. |
| R3 | Every section says what time it is as of (T3/X6) | Each section shows its `as_of`. Missing or stale sources are shown plainly, not hidden. |
| R4 | Two timestamps (T4) | Events carry both `occurred_at` and `ingested_at`. Late data appears in an "added later" block. |
| R5 | Closed pages are append-only (T5) | A closed page's body can't be edited. Additions are visually different. |
| R6 | Citations can be checked (X12) | Every factual sentence links to its source. Tapping a citation opens the source drawer. |
| R7 | No LLM calls at render time | Pages come from stored data only. The only live model call is `/ask`. |
| R8 | Content is evidence, not permission | Email, web and file content is shown as untrusted text. It never produces a button that acts on its own. |
| R9 | Honest over pretty | Empty or stale states say what is true ("iMessage not updated for 3 h"). No decorative placeholders. |
| R10 | lisa drafts, the owner sends | No UI path sends a message or email. Actions that go outside the house end with the owner pressing the final button. |

## 2. Day model the UI relies on

A **day** runs from 04:00 local time to 04:00 the next day. The server assigns each event to a day using `local_date(occurred_at − 4h)`. The client never works out which day an event belongs to.

Each page has a `state`:

| state | meaning | UI |
|---|---|---|
| `open` | today, still being written | live record and cards, input box active |
| `ready` | day ended, nightly job not done yet | "closing…" badge, record frozen |
| `digested` | nightly digest written | review, cards and preview shown |
| `closed` | past page | read-only, "added later" block if there are late events |

## 3. Routes

| Route | Screen | Phase |
|---|---|---|
| `/` | redirects to `/today` | F1 |
| `/today` | today's page (same view as `/day/{today}`) | F1 |
| `/day/YYYY-MM-DD` | any day | F1 |
| `/days` | the book index: one line per day, grouped by month | F1 |
| `/week/YYYY-Www` | week page | F4 |
| `/pending` | approvals desk: every open card or proposal across days | F3 |
| `/memory`, `/memory/{slug}` | brain pages: people, projects, places | F5 |
| `/health` | ops view (already exists, keep it) | exists |

**[changed]** There is no standalone `/ask` screen as a main tab. Chat lives in the input box on every day page (§6). A plain `/ask` URL can open the sheet on today's page.

Phone navigation is a bottom tab bar with three tabs: **Today · Book · Pending**. Memory and Health go in a small menu. On the day page, swipe left or right or use ‹ › to move between days.

## 4. The day page

### 4.1 Sections

| Section | Content |
|---|---|
| Header | Large date, weekday, state badge, `as_of`, ‹ › buttons |
| Review | 2–3 sentences about yesterday, only what carries over into today |
| Cards | 3–5 cards needing action, plus a collapsed "N more" |
| Record | Today's timeline of events and owner notes |
| Preview | Tomorrow and the lead items (1/3/7 days ahead) |
| Footer | Source strip, "last night lisa…" line, page `as_of` |
| Input box | Pinned at the bottom (§6) |

### 4.2 Order depends on time of day

Same URL, different reading order:

- **04:00–12:00:** Review → Cards → Preview → Record
- **12:00–18:00:** Cards → Record → Preview → Review
- **18:00–04:00:** Record → Cards → Preview → Review

The server picks the order using the local time at the moment of the request. A past page (`closed`) always uses Review → Cards → Record → Preview.

### 4.3 Above the fold (morning, iPhone viewport 390×844)

Without scrolling, the owner must see:

1. date and state
2. the review sentences
3. up to 3 cards (collapsed, showing title and due chip)
4. a single warning line, but only if a source is `stale`, `missing` or `blocked`

If all sources are healthy, nothing about sources appears above the fold. The source strip stays in the footer.

### 4.4 Sparse and empty states

LISA doesn't backfill, so early pages will be thin. Design these on purpose:

- 0 cards: "Nothing needs you today." Keep it calm and small.
- Few events: "Quiet day · 3 things recorded."
- Before the first digest: "This page will be written tonight around 04:40."
- Before LISA started: "lisa started on 2026-09-2x. Nothing before that."

## 5. Components

**DayHeader.** Shows the date (serif, large), the state badge, `as_of` ("as of 04:40"), ‹ › buttons, and a day/week switch.

**CardList / Card.** A collapsed card shows a kind icon (`commitment | prep | change | health` — the contract's four kinds; a reply one owes is a `commitment` whose next step is `draft_reply`, [owner edit 2026-09-30]), a one-line title, a due chip (relative, from `due_at`), and a count of citations. Expanded, it adds the sources list, the prepared next step (a draft or a proposed action, with its text shown in full), and the mark buttons: **Done · Ignore · Drop · Do it**.

- Mark buttons act on a single tap and show a 5-second **Undo** toast. There are no confirm dialogs. `Ignore` is sent to the server as a signal, because it's used for learning.
- **Do it** is only shown for actions the action registry allows. At L0, "Do it" for a reply means **copy the draft and open the original thread in Mail** (a `message://` link). Nothing gets sent (R10).
- Each card has an **Ask about this** link, which opens the input sheet scoped to that card's sources (§6.4).

**Timeline (Record).** One row per event: local time, source icon, who, a one-line summary. Tapping a row opens the source drawer. Owner notes appear here with a distinct "you" style. Late events go in an **Added later** block under the timeline, each showing "added at HH:MM" (from `ingested_at`).

**PreviewCard.** Shows tomorrow's calendar items, plus lead items tagged `1d / 3d / 7d` with the reason ("reply needed before Fri").

**CitationChip.** Inline `[1]`, `[2]`… in prose, numbered per section. Tapping one opens the SourceDrawer.

**SourceDrawer.** A bottom sheet on phone, the right column on desktop. It shows:

- source type and account
- who
- `occurred_at` and `ingested_at`
- the raw text, escaped, with remote images blocked
- an **Open in app** link where possible: `message://<Message-ID>` for mail, and machine plus path for files, with a chunk anchor for `file@sha256#chunk`

**SourceStrip.** One pill per source: `complete | partial | stale | missing | blocked`, with "updated X ago". Tapping a pill goes to that row on `/health`.

**NightLine.** One line in the footer, for example: "last night lisa closed 9/29 at 04:40 · 212 events · 4 cards · preview check 3/4".

**CommitmentBadge.** Shows the commitment's state (`open | done | dropped | snoozed`) and days open. From 7 days it adds "asked once".

**BookIndex (`/days`).** Lists days by month. Each row has the date, state, a one-sentence summary and counts. Tapping a row opens that day.

## 6. The input box: note, ask, request

A single input pinned at the bottom of every day page. On the phone it's collapsed to one line and expands into a sheet when tapped. On desktop it sits in the right column.

### 6.1 Three modes

| Mode | Example | Result |
|---|---|---|
| **Note** | "Picked up the kids late, meeting ran over." | Saved as an event with `source=owner` on the current day. It's the most trusted input for the digest. |
| **Ask** | "What did the school email say about Friday?" | A streamed answer with citations and `as_of` |
| **Request** | "Reply to my doctor: I'll see her next Monday noon." | Creates a card with a draft and/or proposed actions. Nothing is executed. |

The server classifies the text and returns the mode. The UI shows a chip such as "saved as note · switch to ask", and switching takes one tap. **If the server isn't confident, it defaults to Note**, because saving something as a note loses nothing.

### 6.2 Ask answers

- Every factual claim has citations. If nothing supports an answer, the reply is "I don't have that in the record." No guessing.
- The answer header shows freshness, for example "as of 21:40 · mail synced 10 min ago".
- Times in answers are absolute and get the same client-side relative rendering as the rest of the page.
- A **Pin to page** button adds the Q&A to the Record as "you asked… lisa found…".
- A conversation belongs to the day it happened on. There is no global chat history sidebar. Past conversations are reached by going to their day.
- A **"remember this"** message produces a *proposed* memory edit that the owner must confirm. The model never writes memory silently.

### 6.3 Requests that involve dates

Relative dates must be resolved and shown explicitly. "Next Monday" on Wed 9/30 is ambiguous: it could be 10/5 or 10/12. The card shows the resolved date in absolute form, and if the server marks it ambiguous, it shows both dates to choose from. Draft text must contain the absolute date ("Monday, October 5 at 12:00"), never "next Monday". If there's a calendar conflict, the card shows it.

### 6.4 Scoped ask

"Ask about this" on a card sends `scope: {card_id}`, so retrieval is limited to that card's sources. The sheet header shows the scope, and the owner can remove it.

### 6.5 Degraded mode

If the model host is down, Ask and Request are disabled with the message "lisa can't think right now (model offline since 21:12)". **Notes always work.** When the phone is offline, notes are queued in IndexedDB and sent later, each with its original typed time as `occurred_at`.

## 7. API contract (for `contract.md` §12.6)

All responses are JSON. All times are ISO-8601 with an offset. Every section has `as_of`.

### 7.1 Endpoints

**[owner edit 2026-09-30] Routes stay flat — no `/api` prefix — matching what the contract already
reserves. `GET /days/{day}` is reshaped into the full DayPage and carries a `shape_version` field.**

```
GET  /days/{day}                     → DayPage   (reshaped; carries shape_version)
GET  /days?from=&to=                 → [{date, state, summary, counts}]
GET  /source/{ref}                   → SourceItem        (ref = event:<id> | mail:<msgid> | file@<sha256>#<chunk>)
POST /cards/{id}/mark                → {action: done|ignore|drop|execute, undo_token}
POST /cards/{id}/undo                → {undo_token}
POST /input                          → {mode, note_id? | ask_stream_url? | card_id?}   (classifies, then delegates to /notes or /ask)
POST /ask                            → text/event-stream (answer tokens + citations + as_of)
POST /memory/proposals/{id}          → {decision: accept|reject}
GET  /health/summary                 → sources[] for SourceStrip
```

`POST /cards/{id}/mark` stays **separate** from `POST /actions/{id}/decide`: a card is a note to the
owner, an action is something the executor does. `POST /input` only classifies what was typed and passes
it to the existing `/notes` or `/ask`.

**[changed]** Ask uses `POST` with the question in the body, not `GET ?q=`, so private questions don't end up in URLs, server logs or browser history.

### 7.2 DayPage shape

```json
{
  "shape_version": 1,
  "date": "2026-09-30",
  "state": "open",
  "tz": "Asia/Shanghai",
  "as_of": "2026-09-30T21:40:12+08:00",
  "order": ["record", "cards", "preview", "review"],
  "review": {
    "as_of": "2026-09-30T04:40:00+08:00",
    "sentences": [
      {"text": "The school asked for the trip form by Friday.", "cites": ["mail:<abc@school>"]}
    ]
  },
  "cards": {
    "as_of": "2026-09-30T04:40:00+08:00",
    "items": [
      {
        "id": "c_812", "kind": "commitment", "title": "Reply to Dr. Chen about the visit",
        "due_at": "2026-10-02T18:00:00+08:00", "status": "open",
        "cites": ["mail:<xyz@clinic>"],
        "next_step": {
          "type": "draft_reply",
          "thread_ref": "mail:<xyz@clinic>", "account": "icloud",
          "resolved_dates": [{"phrase": "next Monday noon", "value": "2026-10-05T12:00:00+08:00", "ambiguous": true,
                              "alternatives": ["2026-10-12T12:00:00+08:00"]}],
          "body": "Dear Dr. Chen, … Monday, October 5 at 12:00 …",
          "allowed": ["copy", "open_thread"]
        }
      }
    ],
    "more_count": 2
  },
  "record": {
    "as_of": "2026-09-30T21:38:02+08:00",
    "events": [{"ref": "event:9912", "occurred_at": "…", "ingested_at": "…", "source": "imessage", "who": "Mia", "summary": "…"}],
    "late": []
  },
  "preview": {"as_of": "…", "items": [{"at": "…", "lead": "3d", "title": "…", "cites": ["…"]}]},
  "coverage": [{"source": "mail.icloud", "state": "complete", "as_of": "…", "watermark": "…"}],
  "night": {"closed_day": "2026-09-29", "at": "2026-09-30T04:40:00+08:00", "events": 212, "cards": 4, "preview_hits": "3/4"}
}
```

`allowed` comes from the server's action policy. The client never decides by itself which actions are available.

### 7.3 The digest must produce JSON **[changed]**

The nightly job writes the review, cards and preview as **JSON matching the shapes above**, citations included. The markdown archive (`archive/YYYY/MM/DD.md`) is rendered *from* that JSON. The frontend must **not** parse markdown headings. If the P2.1 tables (`0003` migration) aren't there yet, `/days/{day}` fills `review` and `cards` from the digest JSON and fills `record` from `raw.events`, so the frontend doesn't change when the tables arrive.

## 8. Time rendering

The server renders every time as an absolute value:

```html
<time datetime="2026-10-05T12:00:00+08:00" data-rel="due">Mon 10/5 12:00</time>
```

A small script (`reltime.js`, about 50 lines, no dependencies) turns these into relative text ("in 5 days", "past", "3 h ago"). It re-runs every 60 seconds and whenever the page becomes visible again (`visibilitychange`). The absolute value is shown in `title` and on long-press. Without JS, the absolute time is still correct. Use `Intl.RelativeTimeFormat` and `Intl.DateTimeFormat` with the page's `tz`.

## 9. Visual direction

Use the **book look for the frame and the tool look for the cards**:

- **Frame** (header, review, record, preview): a serif display face for the date and prose, generous line height, visible page edges, warm neutral colours, and a page-turn feel between days.
- **Cards and source strip:** a sans-serif face, compact layout, high-contrast due chips, and one obvious tap target per action.
- **Dark mode first.** Mornings and nights are the main times it's used. Light mode follows `prefers-color-scheme`.
- **Fonts are self-hosted.** No Google Fonts and no CDNs (§11).
- **Design tokens as CSS custom properties:** `--ink`, `--paper`, `--accent`, `--warn`, `--stale`, `--you` (owner notes), `--late` (added-later), spacing on a 4px scale.
- **Status colours:** complete (neutral), partial (amber), stale (amber, outline), missing (grey, dashed), blocked (red).
- **Motion:** page turns and sheet slide only, and honour `prefers-reduced-motion`.
- **Tap targets:** at least 44×44 px. Respect the safe-area insets for the bottom bar and input box.

## 10. Tech stack

- **Server-rendered HTML** from the existing Python web layer and Jinja templates. The first paint must be readable without JS.
- **Light JS only:** htmx or plain `fetch`. No SPA framework, no build step beyond optional CSS minification. JS handles only relative time, card marks and undo, the drawer and sheet, input submission and streaming, swipe navigation, and the offline note queue.
- **Streaming** for `/ask`: SSE over `fetch` with a `ReadableStream`, since EventSource can't send POST.
- **PWA:** `manifest.json` (standalone, dark theme colour, icons) and a service worker. The service worker caches the app shell, uses network-first for `/days/{today}`, and serves the last 14 `closed` pages cache-first so they're readable offline. It does not cache `/ask` or `/source` responses for mail bodies older than the cached pages.
- **HTTPS origin:** always the `tailscale serve` ts.net URL. Service workers and cookies are tied to that one origin, and IP or port URLs should never be used.
- **Push (later, optional):** iOS web push only works for the installed PWA, and messages pass through Apple's servers. Payloads carry **no content**, only something like "3 cards ready". The page loads the details over the tailnet.

## 11. Security and privacy

- Use the existing session cookie auth (B6 fix): `Secure`, `HttpOnly`, `SameSite=Strict`. All POSTs require a CSRF token. The node/client tokens from B9 are never exposed to page JS.
- **Strict CSP:** `default-src 'self'`, no inline scripts (use nonces if needed), `img-src 'self' data:`, `connect-src 'self'`. No third-party origins at all.
- **All source content is untrusted.** Escape it by default. Render email HTML as sanitised text only, with remote images, forms and scripts stripped. Show links with their full URL, and use `rel="noopener noreferrer"`. Never auto-open a link.
- **Text that looks like instructions** inside sources gets no special treatment. It is displayed, never acted on (R8).
- **Nothing private in URLs** (search terms, questions). Use `Cache-Control: no-store` for source bodies.
- **Logs:** request logs record only route and status, never request bodies.

## 12. Performance and accessibility

- `/today` server response under 150 ms on the M2 with a warm DB. Page ready under 1 s on the phone over the tailnet. JS under 30 KB gzipped. CSS under 30 KB.
- No layout shift when relative times swap in: reserve space for them or use tabular numerals.
- Semantic HTML: `<article>` per page, `<section>` per part, a real `<button>` for each action. The drawer and sheet trap focus and close on Esc or swipe-down. WCAG AA contrast in both themes. Test with VoiceOver on iPhone.

## 13. Testing and acceptance

- **Fixtures:** export 5 real days from the DB (anonymised if they're shared outside the fleet), covering a normal day, a sparse day, a day with late events, a day with a stale source, and a day whose digest failed.
- **Playwright** at 390×844 (iPhone) and 1440×900 (MBP): screenshot tests for each fixture in the morning, afternoon and night orderings. The clock is frozen with `page.clock`.
- **Unit tests** for `reltime.js`, including the 04:00 boundary, DST-free `tz` handling, and "next Monday" display.
- **Acceptance** for each phase requires a screenshot of the *real* morning page on the owner's phone, not fixtures only.

## 14. Build order

| Phase | Scope | Accepted when |
|---|---|---|
| **F0** | Digest JSON output (§7.3) + `GET /days/{day}` + `GET /source/{ref}` | `/days/{real date}` returns valid JSON with citations that resolve |
| **F1** | Day page (header, review, cards read-only, record, preview, footer), ‹ › and swipe, `/days`, citation drawer, `reltime.js`, dark theme | Real morning page passes the 30-second, no-scroll check on the phone. Every citation opens its source |
| **F2** | Input box: **Note** mode + offline queue + PWA install | A note typed offline appears in the Record after reconnecting, with its original time |
| **F3** | Card marks + undo + `/pending` + Ignore signal + "Do it" = copy draft / open thread | Marking, then undo within 5 s, restores the card. Server logs show the ignore signal |
| **F4** | **Ask** mode with streaming, citations, `as_of`, pin to page, scoped ask. Week page | 10 test questions from the eval set: every claim is cited, and "not in the record" is returned when appropriate |
| **F5** | `/memory` view, memory-change proposals, **Request** mode creating cards | A "remember…" message produces a proposal, and nothing changes until it's accepted |
| later | Push, L1 actions (drafts saved to the mail Drafts folder, calendar holds) through the action registry | Separate spec |

F1 depends only on F0. It does not wait for the P2.1 tables.

## 15. Non-goals

These are out of scope: a native app, multi-user support, public internet access, user-defined schedules, and plugin or skill marketplaces. The UI never sends messages or emails, never runs a browser or computer-use agent, logs in to no sites, and never calls the LLM while rendering pages. Health metrics are P3.

## 16. Open questions for the owner

1. Confirm the look: book frame with tool-style cards, dark mode first.
2. How long closed pages stay readable offline on the phone: 14 days proposed.
3. Should "next Monday" default to the nearest Monday (10/5) or the one in the following week (10/12)? When unsure, the card will still show both.
4. Should pinned Q&A appear in the nightly digest's input, or only on the page?
