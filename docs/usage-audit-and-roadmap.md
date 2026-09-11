# Juejin Usage：代码审计问题清单与 Agent 扩展路线

> 审计快照：2026-09-02  · 代码基线：`origin/main` / `ce78a26bcae01ade986cd96f5799e8a780237b04`
>
> 本轮只新增这份文档，没有修改运行时代码，也没有开始修复。
>
> **进展更新（2026-09-05，基线 `cc21d8e`）**：
>
> - ✅ SYNC-001 / SYNC-002：已由 [PR #61](https://github.com/juejin-cn/juejin-usage/pull/61) 修复合并（快照覆盖重扫 + `invalidateCursors` worker 消息）。
> - 🔄 CLI-001：[Issue #87](https://github.com/juejin-cn/juejin-usage/issues/87) + [PR #88](https://github.com/juejin-cn/juejin-usage/pull/88)（分支 `fix/cli/source-arg-contract`）待 review。注意审计后该问题曾恶化：帮助文本把 `all` 列为合法值。修复含 `normalizeSyncSource`、CLI 非零退出、API 400、帮助列表改由 registry 生成（补 `dsh`）、`skipped/error` 透出。
> - 🔄 PARSER-001：[Issue #89](https://github.com/juejin-cn/juejin-usage/issues/89) + [PR #90](https://github.com/juejin-cn/juejin-usage/pull/90)（分支 `fix/cli/copilot-incremental-project`）待 review。核实时发现比原记录更严重：除项目归属丢失外，扩范围重扫会让旧 `unknown` 行与重扫正确行并存（快照替换按含 project 的 key 配对），同一会话双倍计数——已用 E2E 复现（1600 → 3200）。修复为项目名随文件游标持久化（`ClaudeFileCursor.project` 字段已有，照 `dsh.ts` 惯例）。遗留：修复前的历史 `unknown` 行在重扫后仍会并存，"快照重扫时撤销未复现 stale 行"是 replay 协议的通用问题，待与维护者讨论。
> - 🔄 PARSER-002：[Issue #91](https://github.com/juejin-cn/juejin-usage/issues/91) + [PR #92](https://github.com/juejin-cn/juejin-usage/pull/92)（分支 `fix/cli/workbuddy-project-cwd`）待 review。SQLite 兜底用上已查出的 `row.cwd`；JSONL 路径懒加载 `sessions.id → cwd` 映射；补上 sqlite 兜底路径的首批测试。升级场景 E2E 实证历史 unknown 行重扫双计（900 → 1800），与 #89/#90 同一 replay 协议遗留问题，已在 PR 建议单独立项。毗邻发现：CodeBuddy parser 同样硬编码 `unknown` 但暂无已确认 cwd 来源（关联 Issue #69），未展开。
> - 🔄 PARSER-005：[Issue #95](https://github.com/juejin-cn/juejin-usage/issues/95) + [PR #96](https://github.com/juejin-cn/juejin-usage/pull/96)（分支 `fix/cli/trae-seen-cap`，2026-09-06）待 review。核实发现比原记录更严重：不是"可能重复"，而是超过 5 万条后**每轮同步复利式虚增**（E2E 实测每轮 +5,000 tokens）。修复为去重集合按本轮实际观察重建：不再截断、已删行自动清理、故障轮次保留状态；50k 条时 cursors.json ≈ 1.3MB，超大用户可后续做水位压缩。
> - PARSER-003（Cursor 费用语义）已完成代码分析（三个子问题：显式 0 被估价、bucketChanged 忽略费用导致修正永不落盘、tombstone 保留旧费用），方案已定，待动手；included 用量的 CSV 表示建议在 Issue 里向社区征集脱敏样本。
> - 🔄 PARSER-004：[Issue #104](https://github.com/juejin-cn/juejin-usage/issues/104) + [PR #105](https://github.com/juejin-cn/juejin-usage/pull/105)（分支 `fix/cli/jsonl-partial-tail-line`，2026-09-08）待 review。新增共享 `readJsonlTail()`（字节精确、只提交完整行、无换行尾行按 JSON 探测决定），**本批只迁 claude / codex**；其余 11 个同模式 parser（copilot、qoder、kimi、kiro、grok、pi、omp、openclaw、every-code、codebuddy、workbuddy）留待后续 2-3 个小 PR 分批迁移。受控 E2E：1200 → 7000 tokens（旧码丢失 83%）；两个 parser 回归测试已验证在旧源码上失败。附带修掉"游标存 stat size 而非实际消费偏移"导致的潜在双计。
> - 其余条目经逐项核对仍未解决；下一个建议：PARSER-004 剩余 11 个 parser 的分批迁移（机械但需逐个核对状态语义），或 PARSER-003（方案已备）、AtomCode（Issue #12，需报告者提供脱敏 fixture）。
> - 新输入：Issue #13（Qoder 计费改 Credits）、#69（CodeBuddy 检测不到，关联 DATA-003）、#70（web/桌面上传不一致）。

## 1. 本地代码状态

- 已确认仓库已经拉到本地：`/Users/chenchunjie/Desktop/Project/juejin-usage`（与 `lassist-v1` 同级的独立仓库）。
- 远端：`https://github.com/juejin-cn/juejin-usage.git`。
- 写文档前工作树干净；本轮之后唯一新增的是 `docs/`。当前 checkout 是 `main`，`HEAD=ce78a26`，已与 `origin/main` 对齐。
- 本文的文件/行号以 `origin/main` 快照为准。若后续在本地开 PR，先从最新 `origin/main` 建分支并重新核对行号。

## 2. 总结结论

项目已经有 32 个同步 source（`trae` 当前在目录中但 catalog disabled，公开启用约 31 个）。当前最影响用户体验的不是“再增加一个 Agent”，而是：

1. 时间范围扩大时可能重复统计或漏统计；
2. 增量 parser 在边界条件下会丢数据或丢项目归属；
3. 本地 API、登录链接和 CLI 日志的安全边界偏弱；
4. 估算成本和 UI 派生指标没有充分区分真实值与估算值；
5. 新增 source 需要手工同步多处注册信息，容易出现“能写代码但用户无法选择/触发”的半成品。

因此，贡献顺序应先修数据正确性和契约，再扩展 Agent。

## 3. 优先级总表

证据等级：

- **已复现**：用本地 fixture/临时数据得到稳定错误结果。
- **代码确认**：控制流已经足以证明风险，但尚未做完整端到端复现。
- **待外部确认**：需要真实用户日志、发布版格式或云端契约才能最终定案。

| ID | 优先级 | 问题 | 影响 | 证据 | 建议拆分 |
| --- | --- | --- | --- | --- | --- |
| SYNC-001 | P0 | 扩大时间范围会重复累计 | 总 token、会话数和费用失真 | 已复现 | 设计 replay/replace 协议，CLI 与 Desktop 一起修 |
| SYNC-002 | P0 | Desktop worker 保留过期 cursor cache | 扩大范围后可能漏扫旧日志 | 已复现 | cursor 删除/失效通知与 worker 测试 |
| SEC-001 | P0 | Local API 默认无鉴权且全域 CORS | 局部网络暴露时可被其他进程/页面调用 | 代码确认 | 本地 API auth、Origin/PNA 策略、配置写入保护 |
| CLI-001 | P0 | `--source=all` 被当成未知 source；拼错仍成功 | 用户以为同步成功，实际写入 0 条 | 代码确认 | registry 驱动帮助、参数校验、非零退出/HTTP 400 |
| PARSER-001 | P0 | Copilot 增量 tail 不恢复 project | 后续事件落到 `unknown` | 代码确认 | 保存/恢复 project，补两阶段追加 fixture |
| PARSER-002 | P1 | WorkBuddy 查到 cwd 但固定写 `unknown` | 项目维度报表失真 | 代码确认 | 使用查询到的 cwd，补 DB fixture |
| PARSER-003 | P1 | Cursor 零费用与费用修正语义错误 | 真实费用被重新估价或旧值残留 | 代码确认 | 明确 reported/fallback 语义与修正测试 |
| PARSER-004 | P1 | 多个 JSONL byte-offset parser 会跳过半截尾行 | 日志写完后记录永久丢失 | 代码确认（至少 11 个相似实现） | 先迁 Claude/Codex，再分批迁移共享 tail reader |
| PARSER-005 | P1 | Trae seen hash 只保留最近 50,000 条 | 长期运行后旧事件可能重复累计 | 代码确认 | 时间 + 稳定 ID 水位，移除简单 cap |
| DATA-001 | P1 | 未知模型使用默认价却显示为已定价 | 成本数字看似精确，实际可能偏差很大 | 代码确认 | 增加 `costSource/isEstimated/pricingVersion` |
| DATA-002 | P1 | Daily API 缺少明细，UI 用固定比例补 input/cache | 用户看到的是伪造的构成数据 | 代码确认 | 只展示真实字段，或显式标记估算 |
| DATA-003 | P1 | parser health 大多恒定返回 `ok` | 无法区分未安装、无数据和解析失败 | 代码确认 | `installed-no-data/healthy/partial/broken` 状态机 |
| PRODUCT-001 | P1 | “不限制”实际固定为 365 天 | 用户无法查看真正全量历史 | 代码确认 | 使用本地最早数据/服务端窗口，并显示边界 |
| OPS-001 | P2 | 月度 queue append-only，无 compaction | 长期运行磁盘和扫描成本持续增长 | 代码确认 | 去重压缩、原子替换、保留备份 |
| OPS-002 | P2 | config/cursor/manifest/upload state 多处非原子写入 | 中断可能损坏状态或重复上报 | 代码确认 | 临时文件 + fsync/rename + 恢复策略 |
| TEST-001 | P2 | 缺少 PR CI；signal 测试扫描真实用户目录 | 回归慢、环境依赖强、PR 风险高 | 代码确认 | fixture 隔离、CI matrix、可控路径注入 |
| DEP-001 | P2 | 依赖审计仍有 high/moderate 项 | 发布安全风险 | 已有 audit 结果 | 升级并跑完整构建/桌面回归 |

## 4. 详细问题记录

### SYNC-001 / SYNC-002：时间范围扩大后的 replay 不安全（最大影响）

现象：同一条 Claude 记录第一次按 7 天同步后，再把范围扩大到 90 天，实测结果从 `total_tokens=120, conversation_count=1` 变成 `total_tokens=240, conversation_count=2`。这是同一记录被旧 queue 和重扫结果相加，不是新增使用量。

代码路径：

- [`state.ts:324-374`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/src/server/state.ts#L324-L374) 清 cursor，但明确保留已有 queue；
- [`sync/index.ts:75-105`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/src/sync/index.ts#L75-L105) 默认对旧 bucket 和重扫 bucket 做加法合并；
- [`sync/index.ts:159-213`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/src/sync/index.ts#L159-L213) 只对本轮 touched bucket 做增量 append。

Desktop 还有独立漏算路径：主进程调用 `clearCursors()` 后，worker 进程仍持有 [`queue/index.ts:207-225`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/src/queue/index.ts#L207-L225) 的进程内 `cursorsCache`。实测首次 worker 读取到 1 个事件，主进程清磁盘 cursor 后，worker 再跑 90 天读取到 0 个事件。

建议的验收口径：

1. 同一 fixture 依次执行 7D → 90D，最终结果必须等于“从未同步过、直接执行 90D”的结果；
2. CLI 与 Desktop worker 结果一致；
3. 同步中途崩溃后重试既不重复也不丢失；
4. 缩小范围只改变展示窗口，不破坏已采集的更早数据；
5. replay 模式应明确采用 snapshot/replace 或先清理受影响窗口，而不是隐式 add。

这是当前影响最大的单个问题。实现前应先写协议/测试，避免只在某一个 parser 上打补丁。

### CLI-001：source 参数契约不完整

`SYNC_SOURCE_IDS` 已包含 `dsh`，见 [`sync/index.ts:476-510`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/src/sync/index.ts#L476-L510)，但 CLI 帮助的 source 列表 [`args.ts:123-129`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/cli/src/args.ts#L123-L129) 没有 `dsh`。

更严重的是：

- `jusage sync --source=all` 将 `all` 作为单个 source 传入；
- `syncOneSource()` 对未知值返回 `skipped + error`，而不是抛错；
- `cmdSync()` 仍会写 done 日志并以成功状态结束；
- Local API 的 `tud-trigger-sync` 也把这个结果包装成成功响应。

相关代码：[`sync/index.ts:514-602`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/src/sync/index.ts#L514-L602)、[`index.ts:365-403`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/cli/src/index.ts#L365-L403)、[`local-api.ts:562-578`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/src/server/local-api.ts#L562-L578)。

这是最容易合并的小 PR：帮助列表从 registry 生成；`all` 映射为空筛选；未知值返回非零/400；补 CLI/API contract test。

### PARSER-001：Copilot 增量 project 丢失

[`copilot.ts:88-151`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/src/parsers/copilot.ts#L88-L151) 每次从非零 offset 开始读取时都把 `currentProject` 初始化成 `unknown`。如果第一轮读到 `session.start`，第二轮只追加 `session.shutdown`，第二轮 bucket 会失去第一轮得到的项目名。

最小回归 fixture：

1. 写入 `session.start`（包含 project/cwd），运行 parser；
2. 追加带 usage 的 `session.shutdown`，再次运行 parser；
3. 断言 bucket.project 仍等于第一轮 project。

### PARSER-002：WorkBuddy 丢弃已经查出的 cwd

SQL 查询已经选择 `s.cwd`（[`workbuddy.ts:291-306`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/src/parsers/workbuddy.ts#L291-L306)），但写 bucket 时固定使用 `'unknown'`（[`workbuddy.ts:359-366`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/src/parsers/workbuddy.ts#L359-L366)）。应统一项目名解析规则，并覆盖 cwd 为空、路径不存在和正常路径三种情况。

### PARSER-003：Cursor reported cost 修正

[`parsers/cursor.ts:210-285`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/src/parsers/cursor.ts#L210-L285) 只在 `cost > 0` 时记录 reported cost；因此显式的 `Cost=0` 会落入估价路径。同步层的 [`bucketChanged()`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/src/sync/index.ts#L101-L105) 只比较 token 总量和会话数，成本变化或 token 构成变化可能不触发写入；tombstone 复制旧 row 时也可能保留旧 reported cost。

建议先定义三种状态：未上报、明确上报 0、已上报正数，再实现修正/撤销测试。

### PARSER-004：JSONL 半截尾行

多处 parser 使用“保存 byte offset → 从 offset 继续读”的模式，并在 `JSON.parse(line)` 失败后仍将 offset 推进到 `st.size`。例如 [`copilot.ts:95-151`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/src/parsers/copilot.ts#L95-L151)、Qoder、PI、OMP、OpenClaw 等有相似结构。若进程在一行尚未写完时扫描，该行可能永久跳过。

建议先抽共享的安全 tail reader：只提交完整、以换行结束的记录，cursor 停在最后一个完整行；先迁 Claude/Codex 做验证，再逐批迁移其余 parser。

### PARSER-005：Trae seen hash 上限

Trae 通过全表扫描并只保留最近 50,000 个 seen hash。历史记录超过上限后，最早的 hash 会重新进入扫描集合，可能被重复累计。应改用时间 + 稳定事件 ID 的水位，或持久化可压缩的分段索引。

### SEC-001：Local API、深链和日志的安全边界

确认的风险：

- [`local-api.ts:244-248`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/src/server/local-api.ts#L244-L248) 使用全域 `cors()`，本地 API 没有统一鉴权；默认只监听 loopback 能降低风险，但 `--host 0.0.0.0` 时会扩大暴露面；
- [`local-api.ts:490-557`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/src/server/local-api.ts#L490-L557) 允许 PUT 修改任意合法 HTTP(S) API URL，配置变化后会自动执行 full upload；
- [`deep-link.ts:64-144`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/apps/desktop/src/main/deep-link.ts#L64-L144) 只做格式校验，没有 state/nonce 或一次性绑定校验，合法格式的自定义协议参数会直接写入 token；
- CLI 的 start/status 会把完整上报 token 打到终端（[`cli/index.ts:200-203`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/cli/src/index.ts#L200-L203)、[`cli/index.ts:425-435`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/cli/src/index.ts#L425-L435)）；无效 deep link 也会原样写入日志（[`apps/desktop/src/main/index.ts:276-280`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/apps/desktop/src/main/index.ts#L276-L280)）。

复核说明：当前 `GET /functions/tud-config` 返回的是 `hasToken`，不是明文 token（[`local-api.ts:58-81`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/src/server/local-api.ts#L58-L81)）；因此不应把“该 GET 直接泄露 token”作为当前版本的已确认结论。安全问题建议先私下告知维护者，再拆成 API 鉴权、深链防重放、日志脱敏三个 PR。

### DATA-001 / DATA-002：估算值必须可见

内置 pricing 对未知模型使用默认值 input=1、output=5、cache_read=0.1、cache_write=1.25（[`pricing.json:2781-2786`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/src/pricing/pricing.json#L2781-L2786)），测试还明确断言未知模型被视为 priced（[`pricing.test.ts:266-278`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/test/pricing.test.ts#L266-L278)）。建议返回价格来源和估算标记，不要让 fallback 与官方精确价格混为一谈。

另外，Daily API 只提供有限的 tokens/cost/models/projects 字段，Dashboard/ Desktop 会用固定比例推导 input/output/cache，并把 duration 置 0；例如 Desktop 的 [`dashboard-data.ts:440-460`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/apps/desktop/src/renderer/lib/dashboard-data.ts#L440-L460)。这些字段应显示“估算”或暂不展示。

### DATA-003：Parser health 需要真实状态

`buildSyncStatus()` 对大多数 source 使用固定 `status: 'ok'`（[`state.ts:109-170`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/core/src/server/state.ts#L109-L170)），无法区分：没有安装、已安装但无数据、最近一次失败、部分文件解析失败。建议记录 `lastAttempt`、`lastSuccess`、parser version、文件数、事件数和 skip reason。

### PRODUCT-001： “不限制”不是全量

Dashboard 与 Desktop 的 `all` 选项都固定为 365 天：[`packages/dashboard/src/lib/time-range.ts:9-19`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/packages/dashboard/src/lib/time-range.ts#L9-L19)、[`apps/desktop/src/renderer/lib/time-range.ts:9-19`](https://github.com/juejin-cn/juejin-usage/blob/ce78a26/apps/desktop/src/renderer/lib/time-range.ts#L9-L19)。Local API 的 daily/hourly/model-breakdown 查询也 clamp 到 365 天。应改为依据本地最早 bucket/服务端可用窗口，并在 UI 显示实际起止日期。

## 5. 工程质量与运维

- queue 按月 append-only；manifest 更新也不是 compaction。长期使用会增加磁盘占用和读取成本，应设计可恢复的压缩流程。
- config、cursor、manifest、upload state 多处直接覆盖写文件，应统一临时文件、rename 和损坏恢复策略。
- 本地实测 `~/.ai-usage` 目录/JSON 权限偏宽（目录 755、文件 644）；需要在 macOS/Windows/Linux fresh install 上复核并收紧敏感文件权限。
- 当前 CI 主要是 release 同步，缺少每个 PR 都执行的 build/test gate。
- `sync-on-signal.test.ts` 会扫描开发者真实 `~/.claude`、`~/.codex`，单测约 56 秒且不隔离，应注入 fixture root。
- package 声明 Node `>=20`，但若干 SQLite parser 在运行时可能需要 Node `>=22.5` 或系统 `sqlite3`；应统一 engine 检查和错误提示。
- 最近 audit 记录有 4 个 high、4 个 moderate、1 个 low（包括 Hono、PostCSS、js-yaml、nanoid），升级前要跑完整构建和 Desktop 回归。

## 6. 新 Agent 支持路线

### 首选：AtomCode（已有 Issue #12）

[Issue #12](https://github.com/juejin-cn/juejin-usage/issues/12) 已提出 AtomCode 支持。源码显示本地会话位于：

```text
$ATOMCODE_HOME/sessions/<project_hash>/<id>.jsonl
$ATOMCODE_HOME/sessions/<project_hash>/<id>.meta
```

可从 JSONL/meta 对齐得到 `turn_id`、时间、provider/model、prompt/completion/cached token 和 working directory，不需要读取 prompt 正文。实现前仍应让报告者提供一份只保留 schema/数值字段的脱敏 fixture，确认发布版格式与源码一致。

### 第二候选：Continue

默认本地数据库为 `~/.continue/dev_data/devdata.sqlite`，`tokens_generated` 表包含 model/provider、prompt/generated token 和 timestamp。被动 SQLite parser 容易接入，但 token 可能是本地估算，必须标注成本来源。

官方格式说明：[Continue development data](https://docs.continue.dev/customize/deep-dives/development-data)。

### 第三候选：Aider

Aider 官方支持 `--analytics-log file.jsonl`，包含模型和 token 信息，但通常需要用户主动配置，自动发现体验弱于 AtomCode/Continue。

官方说明：[Aider analytics](https://aider.chat/docs/more/analytics.html)。

### 可观察但暂不优先

Reasonix（content-free telemetry）、Prime Agent（本地 session JSONL/RPC stats）、AnythingLLM Desktop（SQLite response metrics）都可以作为后续 source，但目前缺少稳定、可公开分发的真实 fixture。

不建议单独新增 Amazon Q CLI：官方产品已迁移为 Kiro，而仓库已有 Kiro source。[AWS 迁移说明](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/upgrade-to-kiro.html)。

### 新增 source 的验收清单

1. 被动读取本地数据，不上传 prompt/对话正文；
2. 有稳定的时间、模型、token、项目和事件 ID；
3. 首次扫描、增量扫描、文件截断、半截尾行、重写记录都有 fixture；
4. 同步 registry、tool catalog、source presence、CLI help/API 参数同时更新；
5. 定价来源和估算状态明确；
6. CLI 与 Desktop worker 都跑过同一组 fixture；
7. 无数据时显示“未安装/无数据”，不要静默显示为健康。

## 7. Issue / PR 去重记录

以本次审计快照为准，后续认领前先检查这些已有工作：

- Issue #49 已有 PR #56；
- Issue #31 已有 PR #54；
- DeepSeek Harness 已由 PR #42 合并；旧 PR #46 有冲突，应关闭或重基；
- PR #53、#50 当时处于冲突状态；
- Issue #29 尚未看到对应 PR，可单独评估依赖升级范围。

链接：

- [Issues](https://github.com/juejin-cn/juejin-usage/issues)
- [Pull requests](https://github.com/juejin-cn/juejin-usage/pulls)
- [Issue #29](https://github.com/juejin-cn/juejin-usage/issues/29)
- [Issue #31](https://github.com/juejin-cn/juejin-usage/issues/31) / [PR #54](https://github.com/juejin-cn/juejin-usage/pull/54)
- [Issue #49](https://github.com/juejin-cn/juejin-usage/issues/49) / [PR #56](https://github.com/juejin-cn/juejin-usage/pull/56)
- [PR #42](https://github.com/juejin-cn/juejin-usage/pull/42) / [PR #46](https://github.com/juejin-cn/juejin-usage/pull/46)

## 8. 下一轮建议

- **按影响最大选择**：先处理 `SYNC-001/SYNC-002`，因为它会直接改变用户看到的总量，且同时影响 CLI、Desktop 和上传结果。
- **按最容易合并选择**：先处理 `CLI-001`，它是边界清晰的小 PR；随后处理 `PARSER-001`，再做 AtomCode。
- 在范围 replay 协议确定前，不建议继续批量增加 parser，否则会把错误数据写入 queue 并放大后续迁移成本。

本轮没有开始上述任何修复，下一步可直接从本清单选择一个条目开分支。
