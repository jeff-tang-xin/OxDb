# OxDb 手工验收清单

> 定位：把 `docs/architecture.md` §14 的 12 条验收基线翻译成**能照着一步步点的手工测试**。自动化测试补上后，本文件退化为"发版前冒烟 + 回归"。
> 用法：每期交付前按 §6 的该期勾选表走一遍；任一 ★ 项失败即**阻塞交付**。
> 记录：每条结果写进 §7 的模板，不许口头"过了"。

---

## 1. 环境与数据准备

### 1.1 目标库（本机 Docker 或本地实例均可）

```bash
docker run -d --name ox-mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=oxdb -e MYSQL_DATABASE=oxdemo mysql:8.4
docker run -d --name ox-pg    -p 5432:5432 -e POSTGRES_PASSWORD=oxdb -e POSTGRES_DB=oxdemo postgres:16
docker run -d --name ox-redis -p 6379:6379 redis:7
# v0.3（Mongo 走市场安装后才有）
docker run -d --name ox-mongo -p 27017:27017 mongo:7
```

没有 Docker 就用本地安装的实例，只需保证端口与连接 profile 一致。

### 1.2 种子数据（每条验收都依赖它）

**MySQL**

```sql
SET SESSION cte_max_recursion_depth = 200000;

CREATE TABLE t_big (
  id INT PRIMARY KEY AUTO_INCREMENT, name VARCHAR(64),
  amount DECIMAL(20,4), created DATETIME, note TEXT
) ;
INSERT INTO t_big (name, amount, created, note)
WITH RECURSIVE seq(n) AS (SELECT 1 UNION ALL SELECT n+1 FROM seq WHERE n < 100000)
SELECT CONCAT('row-', n), n/7, NOW() - INTERVAL n SECOND,
       CASE WHEN n % 3 = 0 THEN NULL ELSE '' END FROM seq;

CREATE TABLE t_nopk (a INT, b VARCHAR(32));           -- 无主键表：必须只读
INSERT INTO t_nopk VALUES (1,'x'),(2,NULL),(3,'');

CREATE TABLE t_i18n (id INT PRIMARY KEY, txt VARCHAR(128));  -- 中文/emoji/组合字符
INSERT INTO t_i18n VALUES (1,'中文测试'),(2,'emoji 😀🚀'),(4,NULL),(5,'');
INSERT INTO t_i18n VALUES (3, CONCAT(CONVERT(X'65CC81' USING utf8mb4), ' 组合'));  -- e + U+0301，组合字符（不可被 NFC 归一化）

CREATE TABLE t_dec (id INT PRIMARY KEY, v DECIMAL(38,10));   -- 精度杀手
INSERT INTO t_dec VALUES (1, 1234567890123456789012345678.0123456789);
```

**PostgreSQL**

```sql
CREATE TABLE t_big (id INT PRIMARY KEY, name TEXT, amount NUMERIC(20,4), created TIMESTAMP, note TEXT);
INSERT INTO t_big
SELECT g, 'row-'||g, g/7.0, now() - (g||' seconds')::interval,
       CASE WHEN g%3=0 THEN NULL ELSE '' END FROM generate_series(1,100000) g;

CREATE TABLE t_nopk (a INT, b TEXT); INSERT INTO t_nopk VALUES (1,'x'),(2,NULL),(3,'');
CREATE TABLE t_i18n (id INT PRIMARY KEY, txt TEXT);
INSERT INTO t_i18n VALUES (1,'中文测试'),(2,'emoji 😀🚀'),(4,NULL),(5,'');
INSERT INTO t_i18n VALUES (3, 'e' || chr(769) || ' 组合');  -- e + U+0301，组合字符

CREATE TABLE t_dec (id INT PRIMARY KEY, v NUMERIC(38,10));   -- 精度杀手
INSERT INTO t_dec VALUES (1, 1234567890123456789012345678.0123456789);
```

> ⚠️ `t_i18n` 第 3 行故意用组合字符——用来验"字符不归一化"（不能把 `e´` 强行 NFC 化，那等于悄悄改用户数据）。

### 1.3 造"生产连接"

同一实例，但把连接标成 `env=prod`（§2.6 / §6.2）。**这一条必须单独建 profile**，不要临时改标签。

### 1.4 故障手段

| 手段 | 做法 |
|---|---|
| 杀宿主 | 任务管理器结束 `OxDb.exe` |
| 杀服务 | 任务管理器结束 `java.exe`；或用 IDE 的 `stop_process`（杀 runId） |
| 断网 | 断开网卡 / 防火墙出站全禁 / hosts 屏蔽 `api.openai.com` 与 `github.com` |
| 插件篡改 1 字节 | 解压 `.dbplugin` 改其中一个字节再重打包，或直接改已装目录里某个 jar 的 1 个字节（用 `certutil -hashfile` 前后对比） |
| 缺 JRE | 把 `resources\jre` 改名为 `jre.bak` |
| 慢库 / 超时 | 用 `docker pause ox-mysql` 模拟黑洞（不要只拔网线，暂停更容易复现挂起） |

---

## 2. 目录与日志约定（复现用）

| 位置 | 内容 |
|---|---|
| `%APPDATA%\OxDb\plugins\<id>\<version>\` | 用户插件（含 `install-receipt.json`） |
| `%APPDATA%\OxDb\downloads\` | 插件下载暂存（断点续传） |
| `%APPDATA%\OxDb\logs\` | 宿主日志 + 服务 `stderr` 落盘（**+`stdout` 协议独占，不在此处**） |
| `%APPDATA%\OxDb\pending\<sessionId>.json` | 未提交编辑的恢复文件（重启服务前写） |
| `resources\plugins\` | 随包内置插件（只读、预置 receipt） |

排障第一步永远是看 `%APPDATA%\OxDb\logs\` 里服务端 stderr 的**第一段**（JVM 启动参数、插件装载、失败哈希）。

---

## 3. A 组：§14 十二条基线

每条格式：`前置 / 步骤 / 预期`。★ = 该项失败即阻塞该期交付。

### A1 ★ 新增一个库插件，核心与 UI 零改动（v0.2）

- 前置：`resource\plugins` 里没有 H2/SQLite 这类库；工作区干净（`git status` 无改动）。
- 步骤：用 `dbdesk-plugin` 模板生成一个只含 `manifest.json` + 驱动 jar 的插件 → `pack` → `sign` → 应用内手动导入 `.dbplugin` → 重启服务 → 新建连接、浏览表、查询 100 行。
- 预期：连接类型列表里出现该库；全程**没有重新构建宿主、没有改一行 core/UI 代码**；`plugins.list` 里 `state=ready`、`source=user`。

### A2 ★ 10 万行首屏 ≤1.5s 且内存不随行数线性增长（v0.1）

- 前置：`t_big` 已灌 10 万行；服务 `-Xmx512m`（§10.1）。
- 步骤：双击 `t_big` 打开数据标签（不写字面 `LIMIT`）→ 用秒表/日志 `elapsedMs` 记首屏 → 连续翻 20 页（默认 200 行/页）→ 任务管理器盯 `java.exe` 内存。
- 预期：首屏 ≤1.5s；翻页 20 页后 `java.exe` 内存**不呈线性上涨**（±50 MB 内）；`rowCount` 显示 10 万而不是只显示"当前页"。

### A3 ★ 三种故障下无孤儿进程、不丢未提交编辑（v0.1）

- 步骤（三种各做一次，每次做完检查进程列表）：
  1. 杀宿主 → 观察 `java.exe` 是否 ≤2s 内自行退出（父死子亡）。
  2. 杀服务 → 宿主是否提示"服务已重启"并在 ≤2s 内自动拉起；编辑器里的 SQL 与表格滚动位置不丢。
  3. `docker pause ox-mysql` → 查询挂起 → 结果：`TIMEOUT` 而非白屏；恢复 `unpause` 后连接自动可用。
- 预期：任何一次故障后 `java.exe` 残留数为 0；未提交的 **pending changes** 有恢复文件（`%APPDATA%\OxDb\pending\`）；未保存的 SQL 草稿不丢。

### A4 ★ AI 无法在无批准下执行任何 `mutating`（v0.3）

- 步骤：接一个 MCP 客户端（Claude Desktop 或自写脚本）→ 让它调 `db` 写方法与 `kv.set` → 再手工伪造一个 approval 令牌（改 `paramsHash` 或改 `connectionId`）重放。
- 预期：第一次返回 `APPROVAL_REQUIRED` 且**库里查不到任何写入痕迹**；伪造令牌返回 `APPROVAL_INVALID`；日志里两次都有完整记录（谁、哪个连接、什么参数）。

### A5 ★ 插件包篡改 1 字节即装载失败（v0.2）

- 步骤：改已装目录里某个插件 jar 的 1 个字节 → 重启服务。
- 预期：该插件在 `plugins.list` 里 `quarantined`（或干脆未注册），**其余插件正常、服务启动成功**；日志明确写出"哈希与 receipt 不符"；UI 提示该插件需重装。

### A6 ★ 同一 job 文件 UI 与 CLI 结果一致（v0.2）

- 步骤：准备一份 `job.json`（MySQL `t_big` → PG `t_big`）→ UI 跑一次（先 `dryRun` 再 `execute`）→ 清空目标表 → CLI `oxdb-cli job run --file job.json` 跑一次。
- 预期：两次行数相同、目标表逐列校验和相同；进度语义（阶段名、百分比、速率、错误策略）字段一致；中途取消均留下检查点且 `job.resume` 能续。

### A7 ★ 权限引擎 fail-closed + prod 不可绕过（v0.3）

- 步骤：
  1. 故意把权限配置写坏（非法 JSON / 未知 mode）→ 启动 → 预期 **AI 层拒绝启动**并给出明确原因，**不是**静默降级为放行。
  2. 连到 `env=prod` 的 profile → AI 请求 `db.ddl.preview` 之外任何写操作 → 在 UI 点"永久允许" → 再次执行。
- 预期：第 1 步界面显示"AI 不可用（权限配置无效）"，其余功能照常；第 2 步"永久允许"对 prod **无效或压根不提供**，仍要逐次确认（§6.2 第 ③ 级 bypass-immune）。

### A8 ★ 同一句话在 `/` 与 MCP 通道产出相同 intent 与相同裁决（v0.3）

- 步骤：用一句**确定性**话术两次调用（例："把 `t_i18n` 第 2 行 txt 改成 `测试`"），一次从编辑器 `/` 入口、一次从 MCP。
- 预期：两次 intent 序列 `actionId` 顺序相同、params 语义相同、裁决相同（该拒都拒、该批准都要批准）；把两次日志的 intent 序列 diff 出来**只允许 `intentId`/`leaseId` 不同**。

### A9 ★ Esc 撤回提案、人改的提案不被整段覆盖（v0.3）

- 步骤：`/` 让 AI 生成一段尚未执行的 SQL 提案 → 按 Esc → 再让 AI 生成一段 → 手工删掉其中两行 → 让 AI"再完善一下"。
- 预期：第一次 Esc 后 buffer 恢复原文（AI 从未落地）；第二次 AI **不得整段覆盖**，只能在光标处/追加位置插入，人手删的两行不被复活（§6.8 三向互动）。

### A10 ★ 人按键即中断 AI 写入（v0.3）

- 步骤：让 AI 生成一段长 SQL（流式写入，>200 字）→ 在写入过程中随便按一个键。
- 预期：写入立即停止；已写入的部分**保留**（不消失、可 Esc 撤回）；lease 撤销；AI 侧后续不再有字节进入 buffer。

### A11 ★★ 发布→下载→验签→安装→注册 ≤5s（v0.3，用 MongoDB 官方插件真实走一遍）

- 步骤：打 tag → CI 产出 `.dbplugin` + 签名 `index.json` 上传 GitHub Releases → 应用内市场点"安装" → 记时到 `plugins.list` 出现 `state=ready` → 再断网重试一次 → 再让下载在中途中断一次。
- 预期：点击到 ready ≤5s（本地百兆网）；**服务进程 pid 在下载/验签失败时不变**（不重启，§4.7）；下载中断可续传；安装成功后此前打开的连接自动重连、pending changes 从恢复文件回放。

### A12 ★ 关闭 AI 或断网后全链路可用（v0.3）

- 步骤：设置里关闭 AI → 走一遍连接 / 浏览 / 查询 / 内联编辑 / 导入导出 / 迁移 → 再断网重复一遍。
- 预期：全部功能可用；AI 相关只剩"一次"提示（引导开启），**没有错误提示风暴**；布局不留空白格（§1.2）。

---

## 4. B 组：故障注入（每期都做）

| # | 注入 | 预期 |
|---|---|---|
| B1 | 杀 `java.exe` | 宿主自动拉起 ≤2s；UI 明确提示"服务已重启，事务会话丢失"；不卡死、不白屏 |
| B2 | 杀宿主 | `java.exe` 自杀 ≤2s；无孤儿（进程树全清） |
| B3 | 服务 OOM（临时把 `-Xmx` 改 64m，跑大结果集） | `ExitOnOutOfMemoryError` 生效：进程快速退出、宿主重启、给明确错误；**不出现卡死或半死状态** |
| B4 | 缺内置 JRE（改名 `resources\jre`） | 启动即明确报错并给出期望路径；**绝不静默退回系统 Java**（§10.1） |
| B5 | 插件目录里删掉一个依赖 jar | 该插件 quarantined，其他插件与服务照常 |
| B6 | 临时目录只读 / 磁盘满 | 大对象导出失败返回明确错误码，不崩、不写半个文件 |
| B7 | 双开应用 | 单实例锁生效：第二实例聚焦第一实例窗口，不起第二个服务进程 |
| B8 | 半小时闲置后再操作 | 池自愈：连接可用，无需手动重连；日志有连接重建记录 |

---

## 5. C 组：安全红线（v0.2 起每期都做）

| # | 场景 | 预期 |
|---|---|---|
| C1 | `env=prod` 连接上加只读锁 → 任何写操作 | 服务端返回 `READONLY_VIOLATION`（**改前端绕不过去**） |
| C2 | `env=prod` 上的 DDL | 必须先 `db.ddl.preview`，确认需输入连接名（或等强确认），否则不执行 |
| C3 | 无主键表（`t_nopk`） | 内联编辑不可用（只读）；不许出现"改了但匹配不到"的静默失败 |
| C4 | NULL ↔ 空串互转 | `NULL→''`、`''→NULL` 都正确落库并回读一致；**任何一层不得归一化** |
| C5 | 并发改同一行（`WHERE 主键 AND 旧值`） | `affectedRows=0` → 冲突提示；**不覆盖**别人的写 |
| C6 | pending changes 中一条失败 | 整单回滚，不留部分提交（事务语义） |
| C7 | 多语句 `;` 拼接 / 无 `WHERE` 的 `UPDATE` / `FLUSHALL` / `DROP DATABASE` | ① 级 deny 硬拦，连审批都不给（§6.2） |
| C8 | 对 MySQL 连接调 `kv.*`；对 Redis 调 `schema.tables` | `UNSUPPORTED_CAPABILITY`，不让"看起来像支持"的部分执行 |

---

## 6. D 组：分期回归勾选表

### v0.1 骨架（本机首个可跑版本）

- [ ] A2 十万行首屏
- [ ] A3 三种故障无孤儿（第三种可放宽为"超时而非白屏"）
- [ ] B1 / B2 / B4 / B7
- [ ] `tools.list` 与 `plugins.list` 有输出，且 `mutating` 标志正确
- [ ] 无 AI 环境下全部功能可用（A12 的 v0.1 版本：本就没有 AI 入口）
- [ ] 中文字段名/中文数据/emoji 不乱码（`-Dfile.encoding=UTF-8` 生效）

### v0.2 可日常用

- [ ] A1 新库插件零改动
- [ ] A5 篡改 1 字节
- [ ] A6 job 同源
- [ ] C1 / C2 / C3 / C4 / C5 / C6 / C8
- [ ] B3 / B5 / B6 / B8
- [ ] 内联编辑走真实事务提交；导入导出（file 插件 + transfer 引擎）跑通 10 万行
- [ ] SSH 隧道连接可用；i18n 切换不破版

### v0.3 商业化 + AI

- [ ] A4 / A7 / A8 / A9 / A10 / A11 / A12
- [ ] C7 危险语句清单全覆盖（Redis / Mongo 各自的销毁类命令）
- [ ] MCP Server 接入第三方客户端只读可用，写操作必须回 UI 批准
- [ ] 关 AI / 断网 / 未配置三种状态各走一遍（A12）
- [ ] 安装器 + 签名 + 自动更新（O-8 定稿后补测）

---

## 7. 缺陷记录模板

```
编号：OXDB-YYYYMMDD-NN
测试项：A3 / B2
构建：v0.1.0+d16eed6（含日期）
环境：Win 11 / JDK 21 / MySQL 8.4 docker
复现：1) … 2) … 3) …
预期：…
实际：…（附日志片段：%APPDATA%\OxDb\logs\…）
阻塞：是/否   （是 = 该项所属阶段不可交付）
归属：core / plugin / ui / docs
```

判定口径：

- **阻塞（★ 项或安全红线失效）** → 当期不可交付，必须修复后重跑该条。
- **不阻塞**（体验、文案、性能余量） → 可进下一期，但要写进该期待办。
- 同一项连续两次修复仍失败 → 停下，回到 `docs/architecture.md` 对应章节，先判断是**设计问题**还是**实现问题**，不要带病往下叠。
