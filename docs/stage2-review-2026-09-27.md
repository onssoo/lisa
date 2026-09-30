# stage-2 迁移真机复验（M2 主力服务机）— 2026-09-27

> 部署：backup-host → stage2-host（stage2-host 为主力服务机 / macOS 26.7 arm64）。
> 评审人：Hermes（MBP#1）。运行环境实机验证，非纸面结论。
> 同批完成：PG 迁 2014（owner 拍板 dev 期复用现有 PG16）、omp 18.2.7 落 M2。

## 一、迁移验收台账

| # | 项 | 结果 | 证据 |
|---|---|---|---|
| 1 | 数据抢救（chat.db 118M / PG dump / node.db / 证书） | ✅ | `~/lisa-rescue/data/`（M2 本地） |
| 2 | PG 356 事件 + 闸门 + 审计全量恢复 | ✅ | 2014 `lisa` 库，REASSIGN 后 30 表 owner=lisa |
| 3 | M2→2014 PG 连通（pg_hba 授权 tailnet） | ✅ | psycopg 实连 `SELECT version()` |
| 4 | core 常驻（LaunchAgent，8443） | ✅ | `/health` 401（鉴权语义正确） |
| 5 | worker 常驻，tick 零错误 | ✅ | 日志 `tick: {...'errors': []}`（schema 权限修复后） |
| 6 | node 编译（绕 SDK 27.0/编译器 6.3.3 不匹配，用 15.4 SDK） | ✅ | 0 error，selftest 结构项全 PASS |
| 7 | node 配对 + 心跳 | ✅ | `node loop started (node=m2-node)`，6 sources |
| 8 | 采集状态流 | ✅ | 2014 `ops.source_health`：calendar/reminders/contacts=ok（m2-node） |
| 9 | tailscale serve | ✅ | `https://dls-mac-mini-m2.tail69b436.ts.net` → /pair 200 |
| 10 | 浏览器配对登录（client 会话） | ✅（修复后） | `login /health: 200` + 健康页 HTML |

## 二、新缺陷（B6–B9，均为 v4.1.0 源码自带）

**B6（页面路由 KeyError → 500）**：`web/routes.py` 的 `_shell`（挂 `/`、`/today` 等）与 `health_page` 读 `request.state.principal`，但鉴权中间件（`app.py:93`）只对 `/v1/*` 与 `/health` 注入 —— 匿名访问任何页面路由直接 500。MBP#2 未暴露纯因无人点过首页。

**B7（CODES / health_message / message 三连名错 → 登录态 /health 500）**：`web/routes.py:54` 用 `CODES` 与 `health_message`，两者均未 import；`health.py` 中真名为 `message()`。**登录成功后跳 /health 必 500** —— 即 P0 验收过的"健康页"在带会话访问时从未真正渲染过。这是 P0 台账的实质性漏洞：55+ 测试全绿但没有任何一个测试带会话 GET 过 /health。

**B8（配对全局锁过严）**：`api/pair.py` 失败计数全局共享（不分端点、不分来源），5 次即锁 15 分钟。单用户 tailnet 场景下锁的是唯一合法用户；一次客户端 bug（见 B9 期间的孤儿进程重试）即可造成自我拒绝服务。

**B9（pair_raw 明文 token 落 ops.meta）**：`api/pair.py` node 兑换流程将原始 token 以 `pair_raw:<code_sha>` 键存 ops.meta（兑换后 DELETE，但异常路径可残留）。`ops.meta` 同时存有多枚明文 `lisa_*` 令牌（本次排查实测读出）。建议：meta 表只存哈希。

## 三、本次热修（M2 部署目录，仓库未合入 —— 待 coder 正式修）

| 文件 | 改动 |
|---|---|
| `lisa_core/web/routes.py` | 两处 `request.state.principal` → `getattr(..., None)`；补 `from ..health import CODES` + `from ..health import message as health_message` |
| `lisa_core/api/pair.py` | `LOCKOUT_AFTER = 5` → `100`（B8 临时缓解） |

⚠️ 以上补丁只在 `~/lisa-core/app`（M2 部署副本），`~/lisa-v4` 工作树与仓库 main 未动 —— coder 修复后部署目录需同步。

## 四、部署侧修复（非代码，记录备查）

- plist 硬编码旧部署机的用户主目录 → 新机用户名不同，sed 全量替换；node plist 使用 `~` 波浪号路径，launchd 不展开 → 换绝对路径。
- `Info.plist` 误放 `Contents/Resources/` → macOS 不识别 app（Dock 跳两下不启动）→ 挪 `Contents/`。
- SSH 无头会话写 Keychain 报 -25308（errSecInteractionNotAllowed）→ node `--pair` 需经 `launchctl submit`（gui 域）执行。
- TCC：日历/提醒/通讯录经 `LisaNode grant` 实测已授权；imessage 仍 `degraded`（FDA 对 LaunchAgent 实例未生效，待 owner 在 GUI toggle 一次完全磁盘访问）。

## 五、给 coder 的修复请求

1. B6/B7：正式修复页面路由 principal + 健康页 import（以 M2 热修为参考），并**补一个带会话 GET /health 的集成测试**（B7 之所以漏过，就是测试从未以登录态渲染过健康页）。
2. B8：锁计数按端点/来源分桶，或提供 ops 侧解锁命令（本次解锁走直改 ops.meta，属旁路）。
3. B9：token 哈希化 + 清理现存明文行。
4. 顺带：B5（imessage 苹果纪元）已在 main `65b1785` 修复，M2 尚未同步该版本 —— 下次部署一并拉。
