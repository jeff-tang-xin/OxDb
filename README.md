# OxDb

多库桌面客户端 + AI 工具层。

目标形态对标 Navicat：在一个桌面应用里管理 MySQL、PostgreSQL、Redis、MongoDB，并通过插件扩展到 Oracle、SQL Server、达梦等；同时把数据库能力以 MCP 工具的形式开放给 AI 助手——**但 AI 不能自行写入数据库**。

> **当前状态：架构设计阶段（仓库内只有文档，尚无可运行代码）。**
> 完整设计见 [`docs/architecture.md`](docs/architecture.md)。

## 架构

三层，职责严格隔离：

| 层 | 技术 | 负责 |
|---|---|---|
| 桌面宿主 | Rust / Tauri 2 | 窗口与 OS 集成、子进程监管、系统凭据、插件验签、**AI 工具层与审批** |
| 界面 | React + TypeScript | 渲染与交互（数据网格、SQL 编辑器、对话面板），**零业务判断** |
| 能力引擎 | Java 17（stdio JSON-RPC） | 驱动、连接池、会话与事务、元数据、迁移/同步引擎、SSH 隧道 |

宿主与引擎之间走 **stdio + NDJSON(JSON-RPC 2.0)**，对齐 MCP 的传输形态：无本地端口、无 token、父进程退出即自杀，不留孤儿进程。

## 四条设计原则

1. **插件即连接器**——"支持一种数据库" = 一个插件包（`manifest.json` + 驱动 jar），核心代码里没有 `if (mysql)`；侧边栏、标签页、单元格编辑器由插件声明的能力位驱动。
2. **迁移/同步是引擎的两种模式，不是插件**——Job 是 JSON 数据（`full` / `incremental`），CLI 与 UI 同源；插件只负责"读一行/写一行"，框架负责分片、并行、限速、进度、断点与错误策略。
3. **安全判断在服务端**——只读锁、危险语句拦截、生产环境确认由 Java 侧强制，界面上的"灰按钮"只是提示，改前端绕不过去。
4. **AI 只能申请，不能执行**——所有 mutating 调用先变成待审批动作，由人在界面批准后才签发 Ed25519 审批令牌（绑定 method + connectionId + 参数哈希 + 过期时间 + 一次性 nonce）。

## 仓库结构

```
docs/                 设计文档（architecture.md 为基线）
apps/                 桌面应用（Tauri 宿主 + React 界面）        [待建]
crates/               Rust crate：宿主核心、CLI、MCP server      [待建]
backend/              Java 能力引擎（core + plugins + plugin-api）[待建]
contracts/            JSON Schema：RPC 方法、工具描述、插件 manifest、job [待建]
```

## 路线图

| 阶段 | 目标 | 预计 |
|---|---|---|
| v0.1 骨架 | Tauri 壳 + stdio 客户端 + MySQL/PG 插件 + 连接管理 + 对象树 + 只读网格 + SQL 执行 | 3–4 周 |
| v0.2 可用 | 内联编辑（事务提交 + 冲突检测）+ 导入导出 + Redis 全功能 + SSH 隧道 + i18n + 历史 | 8–11 周 |
| v0.3 商业化 | MongoDB + AI 只读助手 + 审批流 + MCP server + 安装器/签名/自动更新 + 插件市场 | 6–8 周 |

## 开发前置条件

| 组件 | 版本 | 当前环境 |
|---|---|---|
| JDK | 17+ | ✅ OpenJDK 17 |
| Maven | 3.6+ | ✅ 3.6.3 |
| Node.js | 20+ | 待确认 |
| Rust | stable | ❌ **未安装**（需 rustup + MSVC `link.exe`） |

## 许可证

待定。
