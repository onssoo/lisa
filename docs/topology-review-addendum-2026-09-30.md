# 追加：owner 澄清 + fibr 实证 —— 对拓扑/文件集成的修正

> 2026-09-30 · 协调人。接 `topology-filebrain-review-response-2026-09-30.md`。
> 本文修正上一篇里**被 owner 纠正**与**被我查错**的部分，并给出修正后的建议。

---

## 0. owner 给出的三条约束（推翻/修正了我的判断）

1. **embed/rerank 在 M2 是刻意的** —— the GPU host 被 owner 当**本地 LLM 试验田**，把常驻小服务挪走是为了**避免 OOM 干扰实验**；M2 是 Apple Silicon、内存富余，适合承载这种"极小但常驻"的服务。
2. **docreader 已从 the GPU host 移除**，fibr **缺文档解析工作流**；owner 有意把 **docreader 随 embed/rerank 一起放到 M2**。
3. **VPS 不作备份站点**（至少现在）：它只跑公开 Web 应用。**沙箱以开发速度优先**，当前阶段安全性不是第一优先。

→ 这三点我接受。下面按它们重排。

---

## 1. 回答 owner 的直接问题：the GPU host 上的东西还在吗？

**在，两样都在。** 实测（`ssh the GPU host`）：

| 项 | 实测 |
|---|---|
| **llama.cpp 构建** | ✅ **4 套**：`~/llama-cuda/build/bin/llama-server`、`~/llama-qwen35-src/…`、`~/llama-k2/…`、`~/llama-mtp/…`；另有 `~/.cache/llama.cpp` |
| **embed 权重** | ✅ `~/models/Qwen3-Embedding-0.6B-Q8_0.gguf`（另有 `~/.cache/qmd/models/` 一份） |
| **rerank 权重** | ✅ `~/models/qwen3-reranker-0.6b-q8_0.gguf`（另有 cache 一份）；`~/models/qwen3-reranker-4b-q4_k_m.gguf`、`~/models/qwen3-embed-4b-nvfp4` 也在 |
| **docreader 镜像** | ✅ `wechatopenai/weknora-docreader` **v0.7.2 + v0.8.0**（各 3.81 GB）仍在本地 |
| **当前监听** | `:8019` llama-server 在跑（4B 小模型）；`:8013/:8014` **无人监听**（已迁 M2） |

**结论：搬回 the GPU host 是"改配置/改 launchd"级别的事，不需要重下权重、不需要重建引擎。**

---

## 2. 修正：embed/rerank 该放哪 —— 原则要改，不是二选一

评审的原则「**所有推理集中一地**」在 owner 的真实约束下**站不住**：the GPU host 是**试验田**，把常驻小服务放上去恰恰会引入 OOM 风险 —— 这正是当初迁走的理由。

**建议改成**：
> **重推理集中在 GPU 机；极小且常驻的公共工具（embed/rerank/docreader）放在服务宿主（M2），必要时可一行配置迁回。**

这样两边的诉求都满足，且 M2 的容量实测支持：**16 GB、内存空闲 73%、磁盘空闲 154 GB**，当前只跑 core/worker/node/embed/rerank。

**待办（我这边）**：`docs/contract.md` 第 213–214 行仍写 `embed_url: http://gpu-host:8013` —— 与已验收的现实（M2）不符，**契约必须改成 m2**，并把 the GPU host 记为回退位。

---

## 3. 修正：我对 fibr 的判断错了 —— owner 是对的

我上一篇说"fibr 已有解析管道，评审在重复造"。**读完 fibr 仓库后，这个判断要改。**

### 3.1 fibr 确实**没有**解析器

```
docreader/            → 只有 __init__.py + proto/     （gRPC 客户端桩）
client/               → 只有 docreader_pb2*.py        （同上）
```
**解析服务是外部的**（WeKnora docreader 容器），已从 the GPU host 撤走 → 所以评审说的 "parse" 阶段**确实是缺的，不是重复**。owner 的判断正确。

### 3.2 但 fibr **已有的东西比评审假设的多** —— 这部分仍然是"别重造"

`ARCHITECTURE.md` + `main.py` + `indexer.py` 实证：

| 能力 | 现状 |
|---|---|
| **上传入口** | ✅ `POST /upload`（multipart，多客户端）**已做**，待补鉴权 |
| **文件表键** | ✅ `files`：`(machine_id, rel_path)` 一行 → 引 `content_hash`；**`UNIQUE(machine_id, rel_path, content_hash)`**（即"路径+内容"复合，且天然带版本） |
| **内容寻址** | ✅ `fv_contents`：**按 `content_hash` 存 + `refcount` + GC**（`fv_gc.py`）—— 正是评审想要的"按哈希去重" |
| **分块/向量** | ✅ `fv_chunks`（含 embedding 列，1024 维） |
| **概念卡** | ✅ `fv_concepts` |
| **导入生命周期** | ✅ `/imports/summary`、`/imports/failures`、`/imports/{id}/retry`、`/imports/retry-batch` —— **评审想让 LISA 追踪的"seen→uploaded→parsed→embedded"，fibr 自己就有** |
| **md 供给** | ✅ `GET /download/md/{file_id}` —— **解析出的 md 在 DB 列里，已可下载** |
| 搜索/筛选/思维域 | ✅ `/search`（四腿混合 + RRF）、`/facet`、`/timeline`、`/stats`、`/filters`、`/preview` |

### 3.3 所以正确的形状是

```
LISA 节点 ──POST /upload(带 client_id/secret)──▶ fibr
                                                 │ indexer（已有）
                                                 ▼
                                        docreader（★唯一缺件：把服务重新跑起来）
                                                 ▼
                                  fv_contents / fv_chunks / fv_concepts（已有）
LISA ◀──读 /imports/* 拿积压与失败，写进页脚────────────────┘
```

**LISA 只做三件事**：①记文件事件并按 hash 关联；②读 fibr 的 `/imports/*` 展示进度/积压；③digest 与 /ask 调 fibr 的搜索并以 `file@sha256#chunk` 引用。
**不要**再写一套 chunk/embed/concept 管道 —— 那才是重复。

### 3.4 md 落盘这件事

owner 要"解析出的 md 也存"。fibr 的铁律是：**md 在 DB 列里，写进语料树会被 indexer 吞掉**（`lessons/03-fibr-project.md`）。安全做法只有三条，选其一即可：
- 写到**扫描根之外**（如 `/srv/filebrain/md/…`）—— 评审的方案其实落在这条上，**可以接受**；
- 或给扫描加排除白名单；
- 或干脆不落盘 —— 因为 `GET /download/md/{id}` 已经能取。

---

## 4. 按 owner 方向修正的其余结论

| 我上一篇写的 | 修正为 |
|---|---|
| 三副本第三份放 VPS | **VPS 不参与**（只跑公开应用）。→ 当前实际是**两份**（mini SSD 主 restic + router HDD），**"一场建筑事故"风险未覆盖**，记为**已知接受的缺口**，待 owner 日后选异地（或明确接受）。 |
| 沙箱必须与 DB 分机 | **放行**：沙箱按需给资源（含网络），**开发速度优先**；安全隔离留到后续阶段收紧。 |
| 2014 mini 超额认购（含解析） | **解析搬到 M2**（与 embed/rerank 同机），谜题解开：mini 只留 **Postgres + fibr DB/向量 + 主 restic**。 |
| fibr 在 mini 上跑 | owner 提议放 M2 —— **可行**：M2 磁盘 154 GB 空闲、内存 73% 空闲。但按 owner 原设计**库与向量仍在 2014 mini**（"2014 pgsql 存 embedded vector"），即 **worker 在 M2、record 在 mini**。 |

---

## 5. 待 owner 拍板 / 待办

1. **worker 分工**：fibr 的 **indexer+docreader+embed/rerank 放 M2**，**Postgres/向量留 2014 mini** —— 确认这个"计算在 M2、记录在 mini"的分法？
2. **docreader 复跑形态**：用 the GPU host 上现成的 WeKnora 镜像（v0.8.0）在 M2 起容器？还是换成轻量解析器（MarkItDown/pymupdf）走同一条 `/upload → indexer` 路？（前者零改动、复用 proto；后者省内存、但偏离 fibr 现有链路。）
3. **契约同步**（我做）：`contract.md` 的 `inference` 段改成 m2 + the GPU host 记为回退。
4. **router 同步**（我做）：MBP#2 标为已退役；M2 角色补 docreader/fibr worker。
5. **文件工作排期**：评审的 F0–F4 保留，但第 2 步（blob 上传）改为**直接 POST fibr 的 `/upload`**（不要另建 blob 库）—— 除非 fibr 的 `/upload` 鉴权会成为阻塞。
