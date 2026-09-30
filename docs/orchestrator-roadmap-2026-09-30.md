# lisa as orchestrator — roadmap (local sandbox)

> **Vocabulary.** lisa core is the **orchestrator** (it orchestrates lisa's services and the
> workers). The project's **coordinator** is a different role entirely — the agent that coordinates the
> *development* of lisa. Do not conflate the two.
>
> Status: **owner decision, 2026-09-30.** Recorded here in full; summarised in `requirements.md` §4 and
> `decisions.md`. This is an **extension of lisa, not a rewrite** — the action layer already exists in
> `design.md` §10.4 / §18; what changes is **scale**, from single actions to multi-step tasks that lisa
> plans, hands to workers, checks and reports on.
>
> **The page comes first.** No sandbox work starts before the daily page is something the owner trusts,
> because that page is the yardstick for every orchestrator task.

---

## 1. The architecture to aim for: brain and hands stay separate

- **lisa core is the orchestrator.** It holds the record, the memory, the policy, the approvals and the
  receipts. It plans and verifies, but **never clicks and never runs code itself**.
- **Sandboxes are the workers.** They are disposable, get only what a task needs, have **no access to
  the record**, keep **no memory between tasks**, and are **reset after each task**.

> The orchestrator has authority but no hands; the workers have hands but no authority.
> If a worker is compromised or confused, the damage stays inside one task and the book is never touched.

## 2. Build it in levels

| Level | lisa can | Gate to move up |
|---|---|---|
| **L0 Read-only** (now) | Collect, digest, remember | A trusted daily page |
| **L1 Single actions** (P2) | Create reminders and calendar events, with approval | Receipts are reliable and `outcome_unknown` is rare |
| **L2 Sandboxed research** | Run read-only tasks in a sandbox (look something up, check a site) and return the results as evidence | Results are cited correctly and nothing leaks out of the sandbox |
| **L3 Multi-step plans** | Plan tasks made of several steps; the owner approves the plan, and any step that changes something gets its own approval | Plans match what actually happened |
| **L4 Policy autonomy** | Action types on the owner's allow list run without asking | A record of results, and the owner's decision |

**Each level is earned by evidence from the level below** — receipts, eval results, error rates.
It is **not** unlocked by finishing the code.

## 3. Do these now, while they are cheap

None of these adds sandbox work to P2, but each keeps the path open.

1. **Make the action layer generic.** Keep a registry of action types; each executor declares its
   capabilities. The Apple reminder executor is the first one. Design proposals so a task can hold an
   **ordered list of steps, each with its own content hash** — P2 only ever uses one step.
2. **Add the action policy table now** (`allow` / `require_approval` / `block`), even while every action
   is `require_approval`. Include a fixed list only the owner can change: **payments, passwords, sending
   messages as the owner, deleting data**.
3. **Label every piece of content with its trust level.** Keep *"Content is evidence, not permission"* as
   a contract rule. Once a orchestrator reads mail and then acts, prompt injection is the main risk. The
   key rule: **content that triggers a task can never approve it** — only the owner's instructions and
   the owner's approvals do that.
4. **Keep the task records resumable.** Every task and step has a state, is idempotent, and includes
   `outcome_unknown`. Build the activity view now; it will show what the orchestrator did later.
5. **Add a `planner` role to `inference.roles`.** Planning multi-step tasks is where a local 27B is
   weakest, so the orchestrator may later need a stronger model. The eval set is how any candidate model
   gets judged there.
6. **Mark what agents produce.** Results from a sandbox come back as `source=agent`, count as evidence,
   and are never extracted as facts — the same rule as lisa's own answers.

## 4. When the sandbox is built

- **Choose a host.** The serving host's 16 GB is already mostly taken, so the sandbox probably wants its
  own machine or a larger one later. The design's executor pool already allows executors on other
  machines — this is a decision, not a blocker.
- **Isolation**
  - start with a **separate OS user**, then a **Linux container**, and a **VM only if the task needs GUI
    apps**;
  - a **fresh environment per task**, reset afterwards;
  - **lisa's data is never mounted inside**.
- **Network:** limit each task to the sites it needs, following OpenMuse's pattern of **no network by
  default**.
- **Credentials:** handled by a **broker**; the sandbox gets short-lived, task-limited access and never
  keeps passwords or tokens. **Banks stay out entirely** (already an owner decision).
- **Recording:** keep logs, screenshots and outputs of every step, attached to the receipt.
- **Take-over:** Screen Sharing into the sandbox whenever it hits a login, a CAPTCHA or a judgment call.

## 5. What is deliberately not decided here

- The sandbox **host** (see §4 — the review proposed the storage host; that host has 8 GB and parsing has
  already had to move off it).
- The **model** for the `planner` role (local 27B vs something stronger) — decided by the eval set.
- Whether the sandbox shares a host with the database. The earlier rule said never; the review reversed
  it. **This needs an explicit owner call, not a footnote.**

## 6. Related

- `design.md` §10.4 (propose rules), §18 (approval and execution), §21 (phases — this roadmap is P4+).
- `contract.md` — action lifecycle, leases, receipts, `outcome_unknown`, `inference.roles`.
- `web-app-design-proposal-2026-09-30.md` — the activity/approval surfaces this will need.
