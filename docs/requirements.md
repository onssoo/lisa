# lisa — requirements

**Source:** owner, 2026-09-23. This document states *what lisa must be* — the requirements.
It is **not** a design: how these are met is architecture, and the architecture is out for
external review.

---

## 1. What lisa is for

A personal life system that records the owner's digital life, explains it, remembers it, and
helps act on it. Five capabilities:

1. **Record** — continuously capture his digital life.
2. **Summarize** — a daily digest: what happened, what matters, what he decided or committed to.
3. **Remember** — a memory that compounds and can be queried by topic (people, projects, health,
   places, work…).
4. **Act** — propose and, with his approval, perform actions on his behalf.
5. **The acceptance case:** the doctor's email about a body check must work end to end —
   it is read, understood, surfaced as a proposal, approved, and becomes a real reminder.

## 2. Who uses it, from where — required

- **Multi-machine.** MBP#1, MBP#2 and Win11 must all be able to use lisa. One owner; one shared
  view of his life from any of them.
- **Centralised, not per-machine silos.** Owner's words: *"我会买一台新的 M1 Mac mini 作为服务器
  代替目前的老 mac mini 2014"* — the new **M1 Mac mini is the server**; the two MacBooks and
  Win11 are clients. (It replaces the 2014 mini.)
- **lisa is "a server + client app that act together"** (owner's words): the server runs
  continuously; the clients are what he actually uses.
- **Collection is per machine**, because the sources are machine-local: iMessage, Mail, Notes,
  Reminders, Calendar and browsing live on the Macs; Windows has its own set.
- **Sequencing:** **macOS first**; **Windows gets its own collector later** — until then Win11 is
  client-only.
- The laptops are allowed to sleep and be offline; that must not lose data or stall the system.

## 3. What must be usable from each machine

- From **any** machine: ask lisa questions about his life; read the daily digest; review and
  approve or deny proposed actions; see the receipt of what was done.
- On **macOS**: full recording of the sources listed in §2.
- On **Windows** (now): the client functions above. **Later:** its own collector.

## 4. Actions

- lisa **proposes**; the owner **approves**; only then does it act; and it **leaves a receipt**.
  Default is to ask, every time.
- Actions that touch Apple apps are performed **where the target lives** — the server
  orchestrates, the machine that owns the app does the write ("server + client acting together").
- Nothing happens silently: he can always see *why* lisa did something, traced to the source.
- If the machine that must perform an action is offline, the action waits for it rather than
  silently failing.
- **Out of scope for v1:** sending messages or mail on his behalf.

## 5. Constraints (these are the real ones)

- **No public / cloud service.** All model calls go to the owner's own inference gateway; no
  third-party SaaS. (This is also why lisa's data cannot live in an iCloud-synced vault.)
- **Captured personal data never enters a git repository.**
- **Explicitly *not* a constraint:** there is **no** requirement that the data stays on one
  machine. A line in an earlier draft invented one ("data never leaves MBP#2"); the owner has
  corrected it and it has been removed.
- **Forgettable.** He must be able to make an item go away — and have it actually gone, not just
  hidden.
- The data includes **third parties** (messages from other people): private by default.

## 6. Quality expectations

- **Unattended operation.** The server is always on; the machines he uses can come and go; a
  gap is caught up later, not lost.
- **Bounded cost.** No spending on external tokens; LLM use is bounded (nightly plus on demand).
- **Honest state.** If a source stops working (permission revoked, an app changes), he is told —
  he does not silently end up with holes in his own history.
- **One-person operable.** He is a scientist, not a software engineer: plain-language failures,
  no maintenance rituals.

## 7. Non-goals (v1)

- No phone / iPad capture; no ambient audio or video recording.
- No sharing with, or acting toward, other people.
- Not a product for others: one owner.

## 8. Requirement-level questions still open

1. Which machines must **record** (versus only use)? Assumption: both Macs record; Win11 later.
2. Windows collector — still wanted at all, or client-only forever?
3. Does he want **proactive** delivery (a morning push) or pull-only?
4. On first run, how far back should history be filled in?
5. **Forget** granularity: per item, per person, per time range?
