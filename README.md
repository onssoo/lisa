# lisa

the owner's **local AI-powered life digest system** on MBP#2.

Five layers, one loop:

1. **Record** — a resident collector captures the digital life: iMessage,
   Mail, files (created/edited/deleted/opened), web history, Notes,
   Reminders, Calendar, Photos.
2. **Snapshot** — each day distills into one snapshot (markdown + JSON).
3. **Digest** — each night an LLM (the fleet LLM gateway) reads the day's
   snapshot: what happened, what matters, what was decided or committed.
4. **Remember** — a compounding **brain**: interlinked markdown pages
   (people, projects, health, places…) with compiled truth + append-only
   timelines, backed by an entity registry + facts + full-text search.
   Every day makes it smarter.
5. **Act** — lisa reads brain + snapshots and helps: answers questions
   about your life, and does things through a safe action lifecycle
   (propose → approve → execute → receipt) — e.g. doctor's email about a
   body check → Apple Reminder, with your approval.

Day by day, life is recorded, analyzed, and memorized.

Architecture borrows two proven open designs: **gbrain** (memory/brain
layer) and **openmuse** (action layer) — see `docs/design.md` §13.

- **Design & architecture:** [docs/design.md](docs/design.md)
- **Data:** lives only on MBP#2 at `~/lisa/data/` — **never in this repo** (code only).
- **Status:** design complete (2026-09-23), build starts at phase 1.
