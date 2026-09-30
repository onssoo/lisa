# lisa infra — current settings, and the coordinator's comments

> **For the external reviewer.** 2026-09-30. Self-contained: the reviewer cannot see the private fleet
> repo, so the measured state is reproduced here. **Sanitized** — machine names are role aliases, no
> addresses, no accounts.
>
> Companions: `topology-filebrain-review-response-2026-09-30.md` (the point-by-point reply),
> `topology-review-addendum-2026-09-30.md` (owner clarifications + the file-brain evidence),
> `agent-context-2026-09-30.md` §3 (the short topology), `orchestrator-roadmap-2026-09-30.md`.

---

## 1. What changed since the topology review

| The review said | Measured reality | Coordinator's comment |
|---|---|---|
| `m2` "runs no models"; `embed → gpu-host:8013`, `rerank → gpu-host:8014` | **embed/rerank run on `m2`** (migrated 2026-09-27, llama.cpp/Metal, launchd, verified equivalent to the GPU host — dim 1024, cosine 0.99969, identical rerank ordering, byte-identical weights). The GPU host no longer listens on those ports | **The review is not wrong — our `contract.md` was.** It still pointed at `gpu-host`, and the review read the contract. Fixed. The migration is deliberate: the GPU host is the owner's **LLM experiment bed**, and parking always-on services there adds OOM risk to experiments |
| "All inference in one place" (principle 2) | Two always-on utilities (embed/rerank) plus a third (**docreader**) now live on the serving host | **Principle amended, not abandoned:** *heavy inference on the GPU host; tiny always-on utilities on the serving host.* The review's version would have reverted a verified migration and broken live users |
| docreader not mentioned as running | **A docreader had been running on the GPU host for weeks** (`Up 2 weeks (healthy)`), exposed to the whole tailnet via a `socat` sidecar on `0.0.0.0:50051` | **The service the owner asked for already existed.** The coordinator built a second one before checking. That failure is recorded in the fleet's lessons log; the GPU-host instance is now **retired** (stopped, `--restart=no`, images kept as a fallback) and `m2:50051` is canonical |
| `mbp` + `m2` + `the GPU host` + `mini2014` + `router` + `vps` | Correct, plus: **the second workstation is retired** (a company machine, now offline — ping fails, absent from the tailnet) | The review's omission was **right**; our own fleet doc still called it the always-on coding machine |
| Sandbox host: `mini2014` | `mini2014` has 8 GB (≈4 GB free) and parsing **had to move off it** | The sandbox on the storage host is now **an open owner decision**, not a settled one — see §5 |
| `vps` as the offsite backup copy | Owner: **the VPS runs public web apps only and will not be a backup target** (for now) | Accepted. Consequence: the fleet currently has **two** copies (storage host + router HDD), so "one building incident" is **not** covered. Recorded as an open gap, not as solved |
| `router` (OpenWrt box, 1 TB HDD) as backup copy 2 | The router **cannot be reached at all right now**: it answers ARP (so it is powered and on the wire) but **every IP port is closed**, even from its own subnet | Backup copy 2 is therefore **aspirational**. A different always-on NAS (a Synology, on the tailnet, DSM answering) is the practical candidate |

## 2. Current settings

| Alias | Machine | OS / capacity | Role | What runs (ports) |
|---|---|---|---|---|
| `mbp` | laptop, 16 GB | macOS | workstation / pilot | the owner's agents + the dev loop; a node for sources that exist only there |
| **`m2`** | mini, Apple M2 **8-core**, 16 GB, 228 GB SSD (**143 GB free**, 66 % RAM free) | macOS 27.0.1 | **serving host** | lisa-core, worker, Apple-source node; **embed `:8013`**, **rerank `:8014`** (bound `0.0.0.0` → tailnet); **docreader `:50051`** (gRPC, tailnet + loopback only); container runtime |
| `gpu-host` | GB10, 128 GB unified | Ubuntu 24.04 aarch64 | **compute / experiment bed** | LLM gateway `:9000` (**the single entry for every agent's LLM traffic**), 27B main `:8888`, small model `:8019`. **No always-on utility services; no agents** |
| `backup-host` | mini, Intel, 8 GB (≈4 GB free), 457 GB SSD (298 GB free) | Linux Mint | **storage host** | the fleet git host `:3001`, **Postgres `:5432`**, file-brain API `:8900`, 3× hermes |
| `public-host` | 2 vCPU, 1.9 GiB (1.4 free), 40 GB (27 free) | Ubuntu 24.04 | public edge | reverse proxy `:80/:443` (live site, HTTP 200), one public app `:8080`, a daily DB-backup timer. **Not part of lisa's backup strategy** |
| the NAS | — | — | (undocumented until today) | a Synology on the tailnet, DSM answering, SSH open. **The natural backup target for the storage host** |
| the router | OpenWrt-class, 1 TB HDD | — | intended backup copy 2 | **unreachable** — see §1 |
| office PC | — | Windows | file-brain development workspace | **not remotely reachable by policy** |
| ~~second workstation~~ | laptop, 16 GB | macOS | **retired** | — |

**Standing rules that go with this**
- All agent LLM traffic goes through the gateway on the GPU host; engine ports are never addressed directly.
- The serving host's model endpoints are bound to the whole tailnet; the docreader is bound to the tailnet **and loopback only** (tighter than the instance it replaced, which was on `0.0.0.0`).
- The docreader runs **unauthenticated**, matching the instance it replaced and the fleet's other tailnet services. A token exists in the private credential store purely as a future switch.
- SSH to the public host uses the **`ubuntu`** account, not `root` (a `root@` attempt fails with `Permission denied (publickey)`, which looks like a bad key but is a wrong username).

## 3. The placement rule (the amendment to principle 2)

> **Heavy inference on the GPU host (the experiment bed). Tiny, always-on utilities on the serving host.
> The record and the backups on the storage host. One machine per job, so nothing is both a test bed and
> a production dependency.**

Rationale, measured: the serving host has room (66 % RAM free, 143 GB disk free) and the utilities are
tiny (§4); the GPU host's value is that experiments can OOM it **without** taking the fleet's always-on
services down with them.

## 4. Measured numbers the sizing decisions should use

**The shared document-parse service (serving host, container)**
| | Measured |
|---|---|
| container idle | **146 MiB** RAM, ~0.2 % CPU |
| one large parse (2000-paragraph document → 106,901 chars of markdown) | **220 MiB peak, 0.4 s** |
| **the container runtime's Linux VM (host side)** | **≈1.3 GB RAM, ≈4.9 GB disk** |
| total | **≈1.45 GB RAM, ≈8.8 GB disk** |

The runtime VM is ~90 % of the RAM bill and is **shared by every future container** (a sandbox would ride
it rather than pay again). Note the trap: the image's *content size* reports 3.86 GB while it occupies
≈7.9 GB on disk — do not size from the content figure.

**A browser inside a sandbox (relevant to L2/L3)**
| | Measured |
|---|---|
| one headless browser page, in a container | **478 MB** |
| a Chromium-family browser with ~10 renderers (the live web collector) | **899 MB** across 18 processes |
| a plain Linux CLI in a container | ~30–80 MB |

→ **Budget ≈1 GB RAM and 1–2 cores per browser task**, capped; 3–4 concurrent on the serving host.

**Isolation, demonstrated rather than asserted.** In a container on the serving host: `uname` reports the
Linux VM, `/Users` does not exist, the macOS root is not visible, and there is **no display, no GPU
(`/dev/dri`), no input devices (`/dev/input`)** — so a sandbox cannot touch the owner's screen or cursor.
On macOS there are **two** boundaries (container → Linux VM → macOS), which is stronger than Linux-native
containers where an escape reaches the host kernel. **That asymmetry matters for the sandbox host choice.**

## 5. What the coordinator asks the reviewer to weigh in on

1. **The sandbox host.** The review placed it on the storage host; parsing has already had to move off
   that 8 GB machine. The serving host is cheap (the VM is already paid for) but it is where the
   orchestrator and the record's write path live — a worker escape lands next to them. A separate machine
   makes isolation structural rather than configurational. **No spare machine exists today.**
2. **The Linux-VM asymmetry.** A container on macOS has a stronger boundary than one on Linux. If the
   sandbox ends up on a Linux host (the storage host), the review's isolation story needs re-examining.
3. **The file/file-brain integration.** The review's core call — *the system owns the record, the
   file-brain owns the content, linked by content hash* — is **adopted**. But two of its proposed
   plumbing details conflict with the file-brain as built: it keys on **`(path, content_hash)`**, not hash
   alone, and its parsed markdown lives **in DB columns, not on disk** (writing `.md` into the corpus tree
   is swallowed by its indexer). Its `docreader/` directory contains only the **gRPC client stub** — the
   parser really is missing, exactly as the owner said. See the addendum for the full evidence.
4. **Backup copy 3.** With the public host excluded and the router unreachable, the offsite copy is an
   open gap. The review's "three copies, two media, one offsite" is not currently true.

> **Resolved 2026-09-30 (reviewer's reply).** (1) The sandbox goes on the **serving host, as its own VM** —
> not the storage host, and not the VM the parse service uses — which restores the rule that *a sandbox
> never shares a machine with the database*. Budget ≈1 GB / 1–2 cores per browser task, 2 concurrent at
> first. (2) The Linux-VM asymmetry is accepted as a reason to prefer macOS for the sandbox. (3) The
> file-brain plumbing is settled: **drive its existing ingest**, keyed on `(path, content_hash)` taken from
> each file event, with the system keeping only `file_ref` and the progress state. (4) **Copy 2 = restic to
> the NAS**; the offsite copy is still open, and `decisions.md` now states plainly that until it exists a
> fire or burglary loses everything.

## 6. Open owner decisions carried forward

- **Sandbox host** (above), **the planner model** (judged by the eval set), and **whether a sandbox may
  share a host with the database** — the review reversed the earlier "never" rule, so it needs an explicit
  owner call rather than a footnote.
- The orchestrator roadmap (`orchestrator-roadmap-2026-09-30.md`) records the levels L0–L4 and the gate
  that **no sandbox work starts before the daily page is trusted**.

---

## 7. Two exposures found after the review (2026-09-30)

### 7.1 The model and parse ports are reachable from the public host

**Measured, not inferred:** from the internet-facing host, TCP to the serving host's `:8013`, `:8014`
**and `:50051` all open**. Those services have **no login**, and the model endpoints are bound to
`0.0.0.0`. On a private tailnet that is a defensible choice — but it means **the one machine exposed to
the internet sits inside the trust boundary of the model ports**, and a compromise there can reach them.

**Recommended fix — a Tailscale policy rule.** The tailnet currently has **no tags applied at all**
(verified: self and all peers report no tags), so a tag-based rule needs tags defined *and* applied first:

```jsonc
{
  // 1. define who may own the tags (the admin)
  "tagOwners": {
    "tag:client":  ["autogroup:admin"],
    "tag:server":  ["autogroup:admin"],
    "tag:storage": ["autogroup:admin"],
    "tag:gpu":     ["autogroup:admin"],
    "tag:public":  ["autogroup:admin"]
  },

  "acls": [
    // 2. the model + parse ports: only the three machines that actually use them
    { "action": "accept",
      "src": ["tag:server", "tag:storage", "tag:client"],
      "dst": ["tag:server:8013", "tag:server:8014", "tag:server:50051"] },

    // 3. existing shape (keep as-is): browser -> the serving host's HTTPS
    { "action": "accept", "src": ["tag:client"], "dst": ["tag:server:443"] },
    // 4. serving -> storage (Postgres) and the GPU host (inference)
    { "action": "accept", "src": ["tag:server"], "dst": ["tag:storage:5432", "tag:gpu:*"] },
    // 5. storage -> the LLM gateway only
    { "action": "accept", "src": ["tag:storage"], "dst": ["tag:gpu:9000"] },
    // 6. the public host may reach nothing at home, and home may push backups to it
    { "action": "accept", "src": ["tag:server", "tag:storage"], "dst": ["tag:public:22"] }
  ]
}
```

**Tailscale's default is deny**, so a port not granted above is blocked — the rule only has to *add* the
three permitted sources. **⚠️ Risk:** this is a *replacement* policy, and applying a wrong one can lock
the owner out. It must therefore be applied **from the admin console with the phone as the test client**,
not blindly.

**What the coordinator needs to finish this:** the **current policy file** (or a Tailscale API key), so the
change can be produced as a reviewed diff instead of a whole-file replacement. **Then** the tags must be
applied to the devices, which also requires the console or an auth key.

### 7.2 File activity has been frozen since 2026-09-26

The `fs`, `edge` and `office_mru` collectors ran on the now-retired workstation. Nothing has replaced
them, so **the day page currently shows no work-file activity at all**.

Two requirements follow:
1. The **source strip must show those sources as `missing`**, with the last-seen date — not hide them
   (rule R9, honest over pretty). A page that silently drops a whole class of activity tells the owner a
   quieter story than the truth.
2. When a node returns to the MBP, the **nightly disk-vs-record check** is what fills the gap — which is
   exactly why that check is in the file-collection design rather than an afterthought.
