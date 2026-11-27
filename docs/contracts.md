# OxDb 协议契约（方法与 Schema 明细）

> 状态：**骨架 v1** — 方法清单按 v0.1 冻结；字段级 Schema 逐条标 `TBD`，实现时补齐并**删掉 `TBD` 字样**（留着的 `TBD` 就是没做完）。
> 上游：`docs/architecture.md` §2（传输形态、帧、错误码语义、能力族）、§4.3（插件 manifest）、§6.0（intent 总线）。
> 分工：architecture.md 回答"**为什么这样定**"，本文件回答"**字段长什么样、谁能调、失败返回什么**"。
> **单一来源**：本文件的 Schema 落地为 `contracts/*.schema.json`；intent 的权威定义在 Rust 代码里，前端 TS 类型由它生成——**任何字段定义不允许手写两份**。

---

## 1. 帧与传输（不可协商）

| 规则 | 内容 |
|---|---|
| 编码 | 一行一条 JSON-RPC 2.0 对象，UTF-8 无 BOM，`\n` 结尾；**禁止**缩进换行与 `\r\n` |
| stdout | 协议独占。Java 启动即抢占原始 stdout 并 `System.setOut(System.err)`；任何库往 stdout 打日志都是 bug（§2.2） |
| 写出 | 单写线程串行化；写失败 = 进程立即退出，不做恢复尝试 |
| stdin | EOF 视为父进程已死 → 服务自杀（父死子亡，不留孤儿，§8） |
| 并发 | 请求可并发处理，响应按**完成顺序**返回，靠 `id` 匹配（不保证保序） |
| 通知 | 无 `id` 的消息：`$/cancel`（必做）、`$/progress`（可选，仅迁移任务用） |
| 单帧上限 | `TBD`（建议 8 MiB）。超限 → `-32600 INVALID_REQUEST`；大结果一律分页或走大对象句柄 |
| 背压 | 队列满时返回 `OVERLOADED`，**不排长队、不静默丢弃** |

---

## 2. 握手 `initialize`

**请求**

```jsonc
{
  "jsonrpc": "2.0", "id": 1, "method": "initialize",
  "params": {
    "clientInfo": { "name": "oxdb-host", "version": "0.1.0" },
    "apiVersion": "1.0",
    "locale": "zh-CN",
    "logLevel": "info",
    "capabilities": ["sql", "schema", "job"]   // 宿主"想要"的能力
  }
}
```

**响应**

```jsonc
{
  "jsonrpc": "2.0", "id": 1,
  "result": {
    "serverVersion": "0.1.0",
    "apiVersion": "1.0",
    "plugins": [
      { "pluginId": "com.oxdb.mysql", "version": "8.4.0", "state": "ready",
        "source": "bundled", "capabilities": ["sql", "schema", "job"] }
    ],
    "limits": { "maxRows": 1000, "maxFrameBytes": 8388608, "maxConcurrentQueries": 4 }
    // features: 预留位，按 apiVersion 主版本内"只增不改"扩展
  }
}
```

规则：

- 握手前**只允许** `initialize` 与 `ping`；其余方法 → `-32002 SERVER_NOT_INITIALIZED`（`TBD` 入错误码表，见 §3.6）。
- `capabilities` 是宿主**想要**的；`result.plugins[].capabilities` 是服务端**能给**的，不一致时宿主按自己的渲染降级，不报错。
- `initialize` **幂等**：重复调用返回同一快照，不重建池、不重载插件。
- 插件装载失败不影响握手成功——失败插件在 `plugins` 里标 `quarantined`（§4.5）。

---

## 3. 通用约定

### 3.1 命名与分域

`<domain>.<verb>[.<sub>]`，全小写、点分隔。域固定为：`app`（生命周期，仅 `initialize`/`ping`/`shutdown`）、`conn`、`session`、`db`、`schema`、`kv`、`doc`、`job`、`tools`、`plugins`。
新增域 = 协议变更，走 §7。

### 3.2 分页

- 请求：`maxRows`（int，1..1000，缺省 **200**）、`nextPageToken`（string，不透明）。
- 响应：`rows[]`、`nextPageToken?`（缺失 = 已到末页）、`rowCount`（可得的行数估计）、`truncated`（是否因上限截断）。
- `nextPageToken` **不透明**，绑定会话与查询快照；会话关闭或服务重启后失效 → `PAGE_TOKEN_EXPIRED`（`TBD`）。
- AI 侧默认 200 行（§2.4），不得自行放大；要更多必须显式申请下一页。

### 3.3 取消

`$/cancel` 通知携带被取消请求的 `id`。服务端**尽力而为**：`db.query` 映射到 `Statement.cancel()`（§2.4）。被取消的请求最终仍必须返回一条响应，结果码 `CANCELED`——**不允许一取消就没有下文**。

### 3.4 大对象句柄

二进制/大文本不进帧：`{ "path": "...", "size": 12345, "sha256": "hex-lower" }`。
`path` 必在宿主管理的临时目录内（Java 侧不得拼任意用户路径）；接收方校验 `sha256` 不符即拒收（§4.7 的同一条纪律）。

### 3.5 JSON 类型映射（最容易出 bug 的一节）

| 源类型 | JSON 表示 | 硬规则 |
|---|---|---|
| SQL `NULL` | `null` | **与空串 `""` 严格区分**，任何一层都不许把 NULL 归一成 `""`（§2.6） |
| 空串 / 空数组 | `""` / `[]` | 同上，从 `null` 转来必须显式 |
| `BIGINT` 超出 ±2^53 | **字符串** + `type: "int64"` | 防止前端 JS 精度丢失 |
| `DECIMAL/NUMERIC` | **字符串** + `type: "decimal"` | 一律不转 double |
| 日期时间 | ISO-8601 字符串；带时区语义的带偏移 | 无时区语义的按连接 profile 的 `timezone` 解释 |
| 二进制 | 大对象句柄（§3.4） | 不 base64 塞帧 |
| 布尔 | JSON `true` / `false` | 不接受 `0/1` |
| Redis nil | `null` | |
| Mongo `ObjectId` / `Decimal128` / `Date` | 包装 `{"$type": "objectId"\|"decimal128"\|"date", "value": "..."}`（`TBD` 定稿） | 不允许裸字符串（会与同名普通字符串撞车） |

### 3.6 错误码

沿用 JSON-RPC 保留码，业务码用 `data.code` 字符串（**前端与 AI 都按字符串判据，不看数字**）。

| 码 | 含义 | 谁产生 |
|---|---|---|
| `-32700` / `-32600` / `-32601` / `-32602` / `-32603` | 标准：解析失败 / 非法请求 / 方法不存在 / 参数非法 / 内部错误 | 服务端 |
| `-32002 SERVER_NOT_INITIALIZED` | 握手前调用其他方法（`TBD`） | 服务端 |
| `AUTH_FAILED` | 库认证失败（凭据错、账号无权限） | 插件/服务端 |
| `CONN_FAILED` | 连不上（网络、隧道、库未启动） | 插件/服务端 |
| `TIMEOUT` | 超出方法级超时 | 服务端 |
| `CANCELED` | 被 `$/cancel` 取消 | 服务端 |
| `READONLY_VIOLATION` | 只读锁/生产护栏拒绝（§2.6） | **服务端强制** |
| `APPROVAL_REQUIRED` | mutating 调用缺审批令牌（§6.3） | Rust 审批门 |
| `APPROVAL_INVALID` | 令牌校验失败：签名/过期/nonce 已用/**paramsHash 不匹配** | Rust 审批门 |
| `UNSUPPORTED_CAPABILITY` | 连接或插件未声明该能力（如对 MySQL 调 `kv.*`） | 服务端 |
| `PLUGIN_QUARANTINED` | 目标插件已被隔离（§4.5） | 服务端 |
| `OVERLOADED` | 背压拒绝（§1） | 服务端 |
| `PAGE_TOKEN_EXPIRED` | 分页游标失效（`TBD`） | 服务端 |

### 3.7 重试与幂等

- **只读**方法可自动重试；**`mutating` 一律不自动重试**（防重复写入），由上层显式重新提案。
- 幂等：`conn.open` / `conn.close` / `session.*` / `db.ddl.preview` / `tools.list` / `plugins.list`。
- 非幂等且必须可去重的：`job.run`（用 `clientRequestId` 去重，同 id 重复提交返回同一 `jobId`）。

---

## 4. v0.1 方法清单（冻结基线）

`mut` = `mutating`（AI 侧必须过审批门）；`期` = 首次交付版本。字段明细 `TBD` 的按上表约定推进。

| 域 | 方法 | 关键参数 | 关键返回 | mut | 期 |
|---|---|---|---|---|---|
| app | `ping` | — | `{ pong: true, uptimeMs }` | ✗ | v0.1 |
| app | `shutdown` | `timeoutMs?` | `{}`（随后进程退出） | ✗ | v0.1 |
| conn | `conn.test` | `profile` | `{ ok, latencyMs, error? }` | ✗ | v0.1 |
| conn | `conn.open` | `profileId`, `profile` | `{ connectionId, capabilities[] }` | ✗ | v0.1 |
| conn | `conn.close` | `connectionId` | `{}` | ✗ | v0.1 |
| conn | `conn.stats` | `connectionId` | 池状态（总/活跃/空闲/等待） | ✗ | v0.2 |
| session | `session.open` | `connectionId`, `mode` | `{ sessionId, capabilities[] }` | ✗ | v0.1 |
| session | `session.begin` | `sessionId`, `isolation?` | `{}` | ✗ | v0.2 |
| session | `session.commit` | `sessionId` | `{ affectedRows }` | **✓** | v0.2 |
| session | `session.rollback` | `sessionId` | `{}` | ✗ | v0.2 |
| session | `session.close` | `sessionId`, `onUncommitted`: `fail`\|`rollback` | `{}`（默认 `fail`，不许静默丢） | ✗ | v0.1 |
| db | `db.query` | `sessionId`, `sql`, `maxRows?`, `nextPageToken?` | `{ columns[], rows[], nextPageToken?, elapsedMs, truncated }` | ✗ | v0.1 |
| db | `db.explain` | `sessionId`, `sql`, `analyze?` | 计划文本/结构化计划 | ✗（`analyze:true` **会真执行**，须走同一套审批，§6.2） | v0.2 |
| db | `db.ddl.preview` | `sessionId`, `ddl` | `{ statements[], estimatedImpact{rows,objects}, warnings[], riskLevel }` | ✗ | v0.2 |
| schema | `schema.namespaces` | `sessionId` | 库/命名空间列表 | ✗ | v0.1 |
| schema | `schema.tables` | `sessionId`, `namespace`, `maxRows?`, `nextPageToken?` | 表/集合列表（分页） | ✗ | v0.1 |
| schema | `schema.table` | `sessionId`, `namespace`, `table` | 列、索引、约束、DDL 一并返回 | ✗ | v0.1 |
| kv | `kv.scan` | `sessionId`, `cursor?`, `match?`, `count?` | `{ cursor, keys[] }` | ✗ | v0.2 |
| kv | `kv.get` | `sessionId`, `key` | `{ value, type, ttlMs? }` | ✗ | v0.2 |
| kv | `kv.set` | `sessionId`, `key`, `value`, `ttlMs?` | `{}` | **✓** | v0.2 |
| kv | `kv.del` | `sessionId`, `keys[]` | `{ deleted }` | **✓** | v0.2 |
| kv | `kv.ttl` | `sessionId`, `key` | `{ ttlMs }` | ✗ | v0.2 |
| kv | `kv.command` | `sessionId`, `args[]` | 原生命令结果 | **✓**（危险命令走 §6.2 ① 级 deny） | v0.2 |
| doc | `doc.collections` | `sessionId` | 集合列表 | ✗ | v0.3 |
| doc | `doc.find` | `sessionId`, `collection`, `filter`, `projection?`, `limit?`, `skip?` | `{ docs[], nextPageToken? }` | ✗ | v0.3 |
| doc | `doc.insert` | `sessionId`, `collection`, `docs[]` | `{ insertedIds[] }` | **✓** | v0.3 |
| doc | `doc.update` | `sessionId`, `collection`, `filter`, `update`, `multi?` | `{ matched, modified }` | **✓** | v0.3 |
| doc | `doc.delete` | `sessionId`, `collection`, `filter` | `{ deleted }` | **✓** | v0.3 |
| job | `job.plan` | `jobSpec` 或 `jobFile` | `{ plan, warnings[], estimatedRows?, estimatedBytes? }` | ✗ | v0.2 |
| job | `job.run` | `jobId`, `mode`: `dryRun`\|`execute`, `clientRequestId` | `{ jobId, startedAt }` | **✓**（`dryRun` 除外） | v0.2 |
| job | `job.status` | `jobId` | 阶段、进度、速率、计数、错误摘要 | ✗ | v0.2 |
| job | `job.cancel` | `jobId` | `{}`（协作式取消，检查点落盘） | ✗ | v0.2 |
| job | `job.resume` | `jobId`, `checkpoint?` | `{ jobId }` | **✓** | v0.2 |
| tools | `tools.list` | — | `{ methods: [{ name, paramsSchema, resultSchema, mutating, capability }] }` | ✗ | v0.1 |
| plugins | `plugins.list` | — | `{ plugins: [{ pluginId, version, state, source, capabilities }] }` | ✗ | v0.1 |

> **UI 侧动作不走这里**：点击/编辑/导出的"动作"是 intent（Rust 侧），不是 Java 方法（§6.0）。Java 只知道"能力"，不知道"按钮"。

---

## 5. intent 信封与词汇表

信封字段（**冻结**，v0.1 起不得改形状）：

```jsonc
{
  "intentId": "uuid-v4",
  "actionId": "ui.grid.editCell",        // 词汇表键，见下
  "params": { },                          // 按 actionId 的 Schema
  "actor": "user" | "ai",
  "origin": "slash" | "toolbar" | "menu" | "mcp" | "replay",
  "anchor": { "editorId": "q1", "range": [12, 3, 12, 9] }
          | { "objectRef": { "kind": "table", "id": "db.schema.tbl" } },
  "leaseId": "…",                         // 持写权限的租约；用户按键即失效（§6.0）
  "idempotencyKey": "…"                   // 回放/重放去重
}
```

五条不变量（与 §6.0 / §6.3 对齐）：

1. **同源**：`actor` 只影响审计与 UI 呈现，不影响裁决路径——UI 与 AI 走同一个 policy 引擎。
2. **params 必过 policy**：intent 的 params 是攻击面（`ui.export.start` 的路径、`ui.wizard.fill` 的目标连接），与 Java 方法同等对待（§6.7）。
3. **唯一事实在服务端会话**：不存在"AI 副本状态"；`ui.state` 只是快照接地。
4. **人永远赢**：用户按键 → 撤销 lease → AI 写入立即停（已写内容保留、可 Esc 撤回）。
5. **改文本即重签**：提案被编辑后 `paramsHash` 变化，旧 approval 令牌作废（§6.3）。

词汇表：v0.1 首批 10 个 actionId 的**名单与语义**由 `architecture.md` §6.0 维护；本文件负责每个 actionId 的 `params` / `result` Schema —— **`TBD`（等 §6.0 名单冻结后逐条补齐，这是 v0.1 的唯一硬前置）**。

---

## 6. 共享 Schema 索引（落地为 `contracts/*.schema.json`）

| 文件 | 用途 | 上游 | 状态 |
|---|---|---|---|
| `contracts/plugin-manifest.schema.json` | 插件清单（含 `manifest.json` 自身哈希的签名约定） | §4.3 / §4.6 | v0.1 必须 / `TBD` |
| `contracts/connection-profile.schema.json` | 连接配置（**不含凭据**，凭据只在 keyring） | §7 / §2.1 | v0.1 必须 / `TBD` |
| `contracts/tools-list.schema.json` | 自省输出（AI 工具目录的唯一来源） | §2.3 / §6.1 | v0.1 必须 / `TBD` |
| `contracts/intent.schema.json` | intent 信封 + 词汇表 | §6.0 / §6.8 | v0.1 必须 / `TBD` |
| `contracts/job.schema.json` | 迁移/同步 job 文件（`full`/`incremental`） | §5.2 | v0.2 / `TBD` |
| `contracts/approval.schema.json` | 审批令牌载荷（`method + connectionId + paramsHash + nonce + 过期`） | §6.3 | v0.2 / `TBD` |

生成链路：Rust 侧定义是唯一来源 → `json-schema-to-typescript` 产出前端类型；Java 侧用同一份 Schema 校验入参（**两侧都从文件读，不各自硬编码**）。

---

## 7. 演进规则

- 协议 `apiVersion` 主版本相等才允许握手；插件 `apiVersion` 同规则（§4.7 闸门）。**双版本同时在场一律拒绝**，不做兼容层。
- **只增不改**：新增方法、新增可选字段、新增错误码 → 不提版本。
- **改语义 / 删字段 / 改字段类型** → 提主版本；主版本内旧语义保留一整个小版本，并在日志里告警一次。
- 方法一旦标 `mutating: true` 就不得回退为 `false`（会静默绕过审批门）。

---

## 8. 待补清单（按 v0.1 开工顺序）

1. `initialize` / `conn.open` / `session.open|close` / `db.query` / `schema.tables|table` 六组完整 Schema（**v0.1 骨架必需**）
2. `tools.list` 输出 Schema —— AI 工具目录的唯一来源
3. intent 信封 + §6.0 的 10 个 actionId 参数 Schema（**intent 总线前置**）
4. `plugin-manifest.schema.json` —— v0.1 的 MySQL / PG 内置插件就要用它打包
5. 单帧上限、`PAGE_TOKEN_EXPIRED` / `SERVER_NOT_INITIALIZED` 两个错误码定稿
6. Mongo/BigDecimal/Date 的包装形式定稿（§3.5 标 `TBD` 的两格）
