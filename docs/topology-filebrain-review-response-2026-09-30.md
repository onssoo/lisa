# 对外部评审的回应：fleet 拓扑 + 文件/filebrain 集成

> 2026-09-30 · 协调人（DSH）。评审对象：(a) `lisa fleet — topology and roles`；(b) 工作文件采集 + filebrain 集成设计。
> 方法：把评审的每条事实主张与 fleet 真相源（`~/router/FLEET.md`、`machines/*.md`、`docs/m2-llama-cpp-migration.md`、`lessons/03-fibr-project.md`）+ 本机实测对照。

---

## 0. 结论（一句话）

**架构判断是对的，值得采纳；但文档里有 2 处与"已验收的现实"冲突，还有 1 个"重复造管道"的陷阱 —— 这三点解决之前，不能当规格用。**

具体说：分层（LISA 记事件 / filebrain 存内容 / 内容哈希相连）和文件采集的一系列工程细节（去抖、垃圾过滤、iCloud 占位文件、FSEvents 漏事件 + 夜间对账、沙箱解析、检索即证据）都是对的方向，应当采纳。但它把 embed/rerank 判在 GPU 机 —— **那是我们自己的 `contract.md` 过时造成的**，不是评审的错；而它给文件设计的一条新管道，会与 filebrain **已有的**管道重复。

---

## 1. 应当采纳的（这部分很扎实）

1. **职责分层**：LISA owns the record，filebrain owns the content，两者用内容哈希相连，**不建第二个 RAG 库**。这是整份评审里最重要的一条，与 `contract.md` 的既有原则一致。
2. **按源定 owner 机**：Apple ID 同源（Notes/Reminders/iMessage/Safari/iCloud Drive）只由一台（m2）采集，避免同一编辑被记两次。配置里写 `owner`，而不是事后去重。
3. **两路输出**：元数据事件（小，进 `raw.events`，可立即用）与内容 blob（按哈希查重后上传，"两台机同一文件只存一份"，每次保存即一个版本）。
4. **工程细节（这些是真经验，逐条采纳）**：
   - **去抖 30–60s** —— Office 是写临时文件再改名，立即哈希会采到半成品。
   - **垃圾过滤**：`~$*.docx`、`.~lock*`、`.DS_Store`、`node_modules`、`.git`、`~/Library`、超大文件。
   - **跳过 iCloud 占位文件** —— 打开会**触发下载**（这是很多人踩的坑）。
   - **FSEvents 会合并/丢事件（尤其睡眠前后）** → 保留 `sinceWhen` 重放 **+ 夜间全盘对账**；删除/改名靠对账才可靠，改名用 file id + hash 判定。
   - **收窄 node token**：只允许上传 blob + 发事件，**不得读 filebrain**。
   - **解析放在沙箱里**：文档是不可信输入（恶意 PDF/Office 真实存在）——这条是安全洞见，必须保留。
   - **检索结果 = 证据，不是指令**（防提示注入）——与既有契约一致。
   - **两种删除要分开**：磁盘上删了（记事件、留历史）vs 要求 LISA 忘记（清 blob/md/chunk/vector + 墓碑；并说明旧 restic 快照到期前仍留有副本）。
   - 容量先算、三副本两介质一异地、restic append-only、季度恢复演练、VPS 看门狗（dead-man's switch）。
5. **不做的取舍**写得清楚（不建第二个 RAG、不做原生 app、旧硬件只做可弃工作）。

---

## 2. 与已验证现实冲突（必须先解决）

### 2.1 embed/rerank 不在 GPU 机，已在 m2（且在用）

评审 §2 写 `m2` **"Runs no models"**、§4 把 `embed → the GPU host:8013`、`rerank → the GPU host:8014`，§11.1 还要求"移除 m2 上的 llama.cpp LaunchAgents"。

**现实（2026-09-27 执行并验收）**：embed + rerank 已从 the GPU host **迁到 M2**，走 llama.cpp(Metal)，launchd 常驻，**the GPU host 侧已退役**。

| 验收项 | 结果 |
|---|---|
| embed 维度 | 1024 ✓ |
| 同文本/异文本余弦 | 1.0000 / 0.3735 ✓ |
| **与 the GPU host 同输入向量一致度** | **0.99969**（Metal vs CUDA 浮点差异）✓ |
| rerank 排序 | 与 the GPU host **完全一致** ✓ |
| 权重 | 与 the GPU host **逐字节一致**（md5 已核）✓ |

并且 **2026-09-29 起 M2 的 `:8013/:8014` 已从 `127.0.0.1` 改绑 `0.0.0.0`** —— 即它们**已在被别的机器使用**，不是试验状态。

> **根因在我们自己**：`docs/contract.md` 第 213–214 行仍写着 `embed_url: http://gpu-host:8013` / `rerank_url: http://gpu-host:8014`。评审是照契约读的，于是把过时事实传播了下去。
> **动作**：先修 `contract.md`，再让评审基于修正后的事实重出拓扑。

**这同时暴露一个真实的设计张力，需要 owner 拍板**：
- 评审的原则「所有推理集中一地」（可运维、可扩、一处容量池）——有力。
- 现实「M2 用 Metal 跑 embed/rerank」（2026-09-27 owner 要求：复用 the GPU host 现有权重、零转换零重下、参数原样）——也已验收且在用。
- 两者只能留一个。**在 owner 决定前，不要让 §11.1 的迁移清单执行。**

### 2.2 MBP#2 已离线，但 router 仍把它写成"常开 coding 主力机"

实测：`ping stage1-host` → **100% 丢包**；tailnet 上不存在（历史条目 `leimacmacbook-pro-1..13` 均已 offline 30–104 天）。

评审的拓扑**没有列 MBP#2 —— 这是对的**。过时的是我们的 `~/router/FLEET.md`（仍称其"常开"）。**动作**：更新 router 的 MBP#2 条目（并把 `mac-mbp2.md` 标为历史）。

> 影响：文件采集的 `mbp1` 侧要看 MBP#1 是笔记本（常睡、常离线）。所以"按源定 owner"要顺带回答：**MBP#1 的 fs 采集断线期间怎么补**（现在 `fs/edge/office_mru` 自 09-26/27 起就是冻住的 —— 因为那台采集器原本在 MBP#2 上）。评审的"夜间对账"正是这条的正解。

### 2.3 小遗漏

- 拓扑未提 **Win11（`dl-pc`）**：它是 filebrain 的**开发工作区**（`Desktop\OMP`）、owner 的 office 机（只读、禁远程触碰）。拓扑可以不含它，但文件集成那节应当有一行说明"filebrain 的代码从哪来"。
- 拓扑未记 **lisa 自身的 P2 计划**（另一份文档的事），但文件工作必须与 P2.0–P2.4 排期对齐。

---

## 3. 最大的问题：重复造管道（会白干）

评审正确地说"LISA 不该建第二个 RAG 库"，但随后**提议了一条新的 parse → md → chunk → embed 管道**（落盘 `/srv/filebrain/md/<sha256>.<parser_ver>.md`，自建 chunk/vector 表）。

**filebrain 已经是这条管道了**，而且不变量不同：

| 评审的假设 | filebrain 的实际不变量（`lessons/03-fibr-project.md`） |
|---|---|
| 按 **sha256** 键 | 按 **`(path, content_hash)` 复合键**（`ON CONFLICT (path, content_hash) DO NOTHING`）——同内容两个路径是两行 |
| md **落盘** `/srv/filebrain/md/…` | md **在 DB 列里**（`files.text` / `fv_chunks.text`）；**把 .md 写进语料树会被 indexer 吞掉** |
| 新建解析器（MarkItDown/pandoc/pymupdf4llm/openpyxl） | **已有解析层**（docreader + parser registry + WeKnora 栈），含 `status` 生命周期、`fv_chunks`、版本表 |
| 自建 chunk/embed 表 | **已有** chunk + embedding 列（1024 维，走 `:9000` model `auto`） |

> **结论**：文件工作不应该并排长出一条 ingest 管道，而应该**驱动 filebrain 已有的 ingest**，LISA 只负责"记录 + 生命周期 + 引用"。
> 落点应当是：filebrain 暴露"以 path/hash 摄入这个 blob"（或 LISA 把 blob 放到 filebrain 扫描根、由它入库），LISA 侧只维护 `file_ref` 与进度（seen→uploaded→parsed→embedded），并在页脚暴露积压/失败。

**这也回答了评审最后那个阻塞问题**（"filebrain 在不在 repo？schema/API 能给吗？按 hash 还是按 path？"）：

- **在**：the fleet git host `filebrain`；运行在 2014 mini（Linux Mint）上，与 Postgres 同机；开发工作区在 Win11 `Desktop\OMP`（`filevault_frontend` / `filevault_indexer_v4.py` / `filevault_*_report.md`）。
- **键**：`(path, content_hash)` 复合 —— 所以"只用 sha256 连接"需要一个映射层（或直接沿用复合键）。
- **md**：DB 列，不是文件。
- **鉴权坑**：`.env` 的 TOKEN 未设时，端点**鉴权空转**（全返回 200）—— 别把"401 冒烟返 200"当代码 bug。
- 另有硬事实：`fibr_deploy` 是冻结的 rsync 副本，**只有 orchestrator 主树是权威**；冒烟禁止拿生产真实 chunk/file id 当靶（曾有毁数据事故）。

---

## 4. 其它风险 / 空白

1. **2014 mini 的容量被低估。** `FLEET.md`：8 GB（可用 4.1），并明确"别在这台跑模型"。而评审在这台上同时放：**Postgres + filebrain + 主 restic 副本 + blob 库 + 沙箱（2 GB/任务）+ 全部文档解析**。这是超额认购；评审自己也承认解析在"8 GB Intel 机器"上很重。
   → 建议：解析要么留在 filebrain 现有的 docreader 位置，要么明确迁到别处；沙箱与 DB 同机的取舍要 owner 明确点头（评审是"推翻旧规则"，但只写在括号里）。
2. **异地副本容量是政策，不是细节。** 512 GB SSD 同时装 Postgres + blob + 主 restic；VPS 只有 40 GB。所以"异地只留最近 N 版"必须**写成明确政策**（N 是多少、由谁决定）。
3. **沙箱与 DB 同机**（§7）是对既有规则的**反转**（"sandbox never shares a host with the database"）。这是一次安全姿态变更，值得一条独立 `decisions.md`，而不是脚注。
4. **提示注入**已考虑（检索=证据），但还应补：**工作文档里的内容不得触发动作**（与 sandbox 的"content is evidence, not permission to act"一致）——评审 §7 有，文件那节应显式引用。

---

## 5. 建议的动作顺序

1. **先修事实，再谈设计**：更新 `contract.md` 的 `inference` 段（embed/rerank 指向真实宿主）+ 更新 router 的 MBP#2 条目。
2. **owner 拍一个板**：embed/rerank 留 M2（现实、在用）还是收归 the GPU host（评审原则）。**这一条是唯一真正阻塞的冲突。**
3. **采纳文件设计的原则与工程细节**，但把**管道改成"驱动 filebrain 的 ingest"**（一份集成说明，而不是一条新管道）。
4. **文件工作按 F0–F4 排期之前**，先出一道**2014 mini 容量题**的答案：解析与沙箱到底放哪。
5. **让评审拿到 filebrain 的真实 schema/API**（或由我们先出一页 schema 摘要）再定集成细节 —— 否则"按 hash 还是 path"这类问题会反复。

---

## 6. 与 owner 原话的对照

Owner 自述："工作文件（docx/xlsx/pptx/pdf/md/html）的元数据（增删改）与内容（解析 ingest，借助 embed/rerank 做 RAG）要由 lisa 协调；存到 mini2014 备份，解析出的 md 也存；2014 的 pgsql 存向量（我自己的 filebrain 已在 2014 上跑，per-machine 采集器还只是计划）。"

→ **完全同意**，且评审的分层方案与之一致。唯一要改的是：**"解析出的 md 也存"这条要按 filebrain 的现状落地（md 在 DB，不在磁盘）**，否则会与 indexer 打架。
