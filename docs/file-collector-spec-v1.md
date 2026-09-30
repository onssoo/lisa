# file collector spec v1 — LisaNode FSEvents → LISA + fibr

> Status: spec for review, 2026-09-30. Baseline: `topology-review-addendum-2026-09-30.md` (which
> corrected both the coordinator and the reviewer: fibr has **no parser**, the external docreader was
> removed, but fibr already owns upload / content-hash storage / chunks / import tracking).
> Authority: requirements → design/contract → this spec → code.

## 1. What it is

The collector is an **extension of LisaNode's existing FSEvents reader**, not a new program. It runs on
each Mac, and each machine has its own roots in config. **M2 owns iCloud Drive**; MBP1 covers only its
local work folders. Every file change produces two outputs:

```
LisaNode (per Mac)
  ├─ metadata event ──▶ LISA core   raw.events   (small, always)
  └─ file content ────▶ fibr POST /upload        (whitelisted types only)
                           └─ indexer → docreader → fv_contents / fv_chunks
LISA core ◀── fibr /imports/* (backlog, failures) → page footer
```

Content goes **straight from the node to fibr**, not through LISA core. Large files shouldn't pass through
the orchestrator. **LISA stores a reference, not the file.**

## 2. The metadata event

`kind: file.created | file.modified | file.renamed | file.deleted`, with these fields:

- `machine_id` and `root_id` (which root, e.g. `mbp1:work`)
- `rel_path`, plus `old_rel_path` for renames
- `occurred_at` (from the FSEvent or mtime) and `ingested_at`
- `size`, `mtime`, `birthtime`, and the file ID (inode)
- `content_hash` (sha256), empty for types that aren't uploaded

`machine_id + rel_path + content_hash` is exactly fibr's key `UNIQUE(machine_id, rel_path, content_hash)`,
so the link between LISA and fibr needs **no mapping table**. **Define `rel_path` relative to `root_id`
and never as an absolute path** — otherwise renaming a home folder or getting a new laptop breaks history.

## 3. What gets uploaded

- **Upload the content** of: docx, xlsx, pptx, pdf, md, html (and txt/csv if wanted).
- **Record metadata only** for everything else in the roots (images, zips, code). They still show up as
  "you worked on X" in the record.
- **Skip entirely:** `~$*`, `.~lock*`, `.DS_Store`, `.git`, `node_modules`, `~/Library`, temp and backup files.
- **Size cap:** skip content over ~100–200 MB, but still record the event.
- **Skip iCloud placeholders.** Check `NSURLUbiquitousItemDownloadingStatusKey` before reading — opening a
  placeholder forces a download.

## 4. Getting the timing right

- **Debounce.** Wait until a file has been quiet for 30–60 s before hashing: Office saves by writing a
  temp file and renaming it. Emit one `modified` per burst of saves, not one per write.
- **Detect renames** by matching **file ID plus hash**, not delete-then-create.
- **Replay on wake** — keep the existing `sinceWhen` replay.
- **Nightly reconcile scan.** Walk the roots, compare against the last known state, emit missed creates,
  deletes and renames with `detected_by: reconcile`. This is the only reliable way to catch deletes.
- **The laptop needs an offline queue.** Queue uploads on disk while the MBP1 is away; upload only when
  fibr is reachable, and throttle while on battery.

## 5. Changes needed on the fibr side (small)

1. **Authentication on `/upload`** (still to be done). A separate client ID and secret per machine, in that
   machine's Keychain, allowed to **upload only** — not search, not download. **This is the blocker;** until
   it exists, collect metadata only.
2. **A hash check before upload**, e.g. `HEAD /contents/{sha256}` — an unchanged file that also exists on
   another Mac is then never re-sent. fibr already dedupes server-side, so this only saves bandwidth;
   useful for the laptop, optional for the M2.
3. **Deleting a file on disk must not delete its history** — see §10, which answers this from the code and
   finds it is currently **not** true.

## 6. The parser (the missing piece)

Run the **WeKnora docreader v0.8.0 image as a container on the M2** (already done — see
`docreader-m2.md`). fibr's client already speaks its gRPC protocol, so **no code change is needed**, and it
sits next to embed and rerank. If memory use turns out too heavy, a lighter parser can be built later
behind **the same gRPC interface** (MarkItDown for Office/HTML, pymupdf4llm for PDF, openpyxl for xlsx) so
fibr doesn't notice the swap. Scanned PDFs go to the GPU host for OCR. Containerising also gives basic
isolation from malicious documents at no extra cost.

**On storing the markdown:** it is already stored, in fibr's DB, fetchable via `GET /download/md/{id}`, and
backed up with Postgres. **Do not write md files to disk at all.** If files are wanted anyway, write them
**outside every folder fibr scans**, or the indexer re-imports them.

## 7. Existing files: the first scan

LISA doesn't backfill events, but the content index should include files that already exist. So the first
run on each machine uploads existing files (**throttled, overnight**) and records **one baseline event per
root** ("baseline: 3,412 files"). It must **not** invent thousands of fake "created" events dated in the
past. Pages before the start date stay empty, per the no-backfill rule.

## 8. What LISA does with it

- **Day page record:** grouped lines such as "edited budget.xlsx ×4 · 14:10–16:30 · mbp1", no per-save noise.
- **Footer:** LISA reads fibr's `/imports/summary` and `/imports/failures`, e.g.
  "files: 12 waiting · 1 failed (report.pdf)". fibr already tracks this; LISA doesn't track it again.
- **Digest and /ask** call fibr `/search` and cite `file@sha256#chunk`. Document content is **evidence**,
  never an instruction.
- **Source strip:** `files.mbp1` and `files.m2` as **separate sources** — "mbp1 files: last seen 2 days ago"
  is the honest answer when the laptop has been closed.

## 9. Build order and acceptance

| Step | Scope | Accepted when |
|---|---|---|
| **F0** | Node on the MBP1 plus metadata events from both Macs, debounce, filters, rename detection | A real day page shows work-file activity from both Macs, with no duplicates for iCloud Drive |
| **F1** | Nightly reconcile scan | A file deleted while the laptop slept appears next morning as `detected_by: reconcile` |
| **F2** | Auth on fibr `/upload`, node uploads, hash check, offline queue | The same file on both Macs is stored once; an edit creates a new version; **old versions stay after the file is deleted on disk** |
| **F3** | docreader on the M2, first scan throttled overnight | `/imports/summary` shows the backlog draining; failures appear in the footer |
| **F4** | Digest and /ask use fibr search with citations | "what did I change in the budget this week" returns cited chunks |

**F0 has the highest value right now**: file activity has been frozen since MBP#2 retired on 9/26, and F0
fixes that **without depending on fibr at all**.

**For `decisions.md`:** *"Work files are collected by LisaNode on each Mac (roots per machine; iCloud Drive
owned by m2). Metadata goes to LISA as file events; content of whitelisted types is uploaded by the node
directly to fibr `/upload`, which owns storage, parsing (docreader on m2), chunks and vectors. LISA links by
`(machine_id, rel_path, content_hash)`. Deleting a file on disk keeps its history; only an explicit forget
purges it."*

---

## 10. The `fv_gc.py` question — answered from the code and the live database

The reviewer asked, before F2: *what happens to old versions today when a file disappears from disk?*
**Answer: nothing today — and if anything ever deletes the file row, the history is destroyed immediately,
not after a grace period.** The evidence:

**a. Nothing in fibr reacts to a file disappearing from disk.** The indexer has **no**
missing/disappeared/prune path (grepped), and the only two `DELETE FROM files` sites are the explicit
delete and the retry path. So **today a disk-delete leaves fibr completely untouched** — row, content and
vectors all survive. The history is safe **by accident, not by design**.

**b. When a file row *is* deleted, the content is released by a trigger:**
```sql
-- trigger files_refcount
IF TG_OP = 'INSERT' THEN ... refcount = refcount + 1
ELSIF TG_OP = 'DELETE' THEN UPDATE fv_contents SET refcount = refcount - 1 WHERE refcount > 0
```
Only a **DELETE** decrements — an UPDATE (e.g. setting a status) does not. So *marking* a file deleted is
safe; *removing the row* releases the content.

**c. The GC then purges — and the 7-day grace period does not work.**
```python
# fv_gc.py
refcount = 0 AND status = 'active'          → status = 'trashed'
status = 'trashed' AND first_seen_at < NOW() - INTERVAL '7 days'  → DELETE row + os.remove(blob)
```
The purge test uses **`first_seen_at`**, and the table has **no `trashed_at` column** (verified against the
live schema). So the "keep 7 days in the trash" intent is measured from **when the content was first seen**,
not from when it was trashed. **Any content older than a week that loses its last reference is purged on the
very next GC run** — and the purge deletes the blob from disk: the one copy no other machine has.

**d. Live proof, right now:** `fv_contents` holds **23 rows at `refcount = 0, status = 'active'`**, first
seen **2026-09-03 … 2026-09-10** — 20 to 27 days old, i.e. **already past the 7-day test**. The next GC run
purges them, not in a week.

**e. What this means for the collector (the F2 requirement):**
1. **Never delete the `files` row because a file vanished from disk.** Mark it. (Note `files.status` today
   holds only `indexed | failed | empty` — a deleted marker is a new value to agree on.)
2. The reconcile scan must therefore emit `file.deleted` **metadata events only**; it must not drive fibr
   row deletions.
3. Purging stays what the spec says it is: an **explicit owner action** ("forget this file"), which writes a
   tombstone.
4. **Separately, `fv_gc.py` needs fixing** so "trashed" is timed from the trash: add `trashed_at` and purge
   on that. Without it, any future reconcile step that removes rows would start destroying history — silently,
   and on the first GC cycle.
