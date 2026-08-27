# PROTOCOL-COMPAT: codex 协议兼容契约

> 本文档是 MindOS Desktop 与 codex app-server 协议之间的**契约**。
> 任何升级、改 Rust、改协议字段前必须先读本文档。
> 目标:一次升级从「2 天逆向调研」压缩到「30 分钟对照清单回归」。

## 1. 绑定版本

| 项 | 值 |
| --- | --- |
| 绑定二进制 | `codex-cli 0.149.0`(官方 release `codex-package-x86_64-pc-windows-msvc`) |
| 上游基线 tag | `rust-v0.149.0`(openai/codex) |
| 基线 tag(本仓库) | `v0.149.0-mindos-baseline`(codex-main fork 中) |
| 资产命名 | `codex-package-x86_64-pc-windows-msvc`(Windows) |
| 打包来源 | 官方 release 资产;仅当开始修改 Rust 源码时切换为 fork 构建流水线 |

**协议字段位置**(改造时唯一的权威来源):

- `electron/codex/app-server-manager.ts`
  - `buildThreadStartParams()` / `buildThreadResumeParams()` — thread/start、thread/resume 参数构造
  - `ON_REQUEST_APPROVAL_POLICY` — approvalPolicy 注入值(`'on-request'`)
  - `WINDOWS_SANDBOX_BASE_INSTRUCTIONS` — Windows 沙箱约束指令(baseInstructions)
  - `windowsSandboxBaseInstructions()` — 平台判定(仅 win32 注入)
- `electron/codex/event-converter.ts` — 服务端通知 → 前端事件白名单映射
- `electron/codex/audit.ts` — 审计事件解析(依赖事件 item 结构)

## 2. 高风险项(升级必查,任何一项回归即视为协议破坏)

### 2.1 thread/start 参数

构造在 `buildThreadStartParams()`,升级后逐一核对 wire 层(`schema/typescript/v2` 附近)是否仍接受:

| 字段 | 值 | 说明 |
| --- | --- | --- |
| `baseInstructions` | `WINDOWS_SANDBOX_BASE_INSTRUCTIONS`(仅 win32) | Windows 沙箱约束注入 |
| `approvalPolicy` | `ON_REQUEST_APPROVAL_POLICY`(见 2.2) | 按需审批(on-request) |
| `config.model_context_window` | `DEFAULT_CONTEXT_WINDOW` | snake_case,对齐 v2 协议 |
| `config.model_auto_compact_token_limit` | `DEFAULT_AUTO_COMPACT_TOKEN_LIMIT` | snake_case,对齐 v2 协议 |

> ⚠️ 历史遗留:`historyMode: 'paginated'` 曾注入到 params 顶层,但 v2 schema(0.146.0 起)的
> `ThreadStartParams` 中**从不存在该字段**,一直被服务端静默忽略;分页历史实际由
> `thread/turns/list` API 实现。2026-08-23 升级 0.149.0 时已删除该注入(含 spec 断言)。

风险点:字段改名、camelCase ↔ snake_case 漂移、`baseInstructions` 在非线程创建路径失效。

### 2.2 approvalPolicy

`ON_REQUEST_APPROVAL_POLICY` 当前值(对齐 `app-server-protocol/schema/typescript/v2/AskForApproval.ts`):

```ts
'on-request'
```

含义:需要审批的操作一律弹窗征询用户。显式提权请求(`sandbox_permissions: require_escalated`)
经 exec policy 判定为 NeedsApproval → 前端弹窗,批准后以无沙箱完整权限执行,拒绝则不执行;
普通命令在受限沙箱内执行(工作区外写入 Access denied 属正常现象)。

风险点:on-request 枚举名变化、提权请求不再走 NeedsApproval(需审批的判定被移除/改名),
导致工作区外写入从「弹窗批准」退化为「直接 Access denied」。

### 2.3 事件 schema

`convertServerNotification()` 白名单映射,升级后逐个对照:

- `thread/started` → `thread.started`(threadId 透传)
- `turn/started` → `turn.started`
- `turn/completed` → `turn.completed`(usage + threadId,四层兜底解析:params → turn → thread → sessionId)
- `turn/failed` → `turn.failed`(error)
- `item/started` / `item/updated` / `item/completed` → `item.*`(item 透传)
- `rawResponseItem.completed` → 原始 item 转发
- 白名单外通知一律丢弃

风险点:事件名改动、item 结构变化(工具调用/function_call 字段)、threadId 位置变化导致线程归属错乱。

### 2.4 sandbox 注入指令

`WINDOWS_SANDBOX_BASE_INSTRUCTIONS` 是注入到 `baseInstructions` 的约束指令,内容:

- 禁止在 restricted token 沙箱内执行 `%LOCALAPPDATA%\Microsoft\WindowsApps\` 下可执行文件(AppX reparse point,会报 Windows error 1920 / ERROR_CANT_ACCESS_FILE)
- 统一走 `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe` 与 `cmd.exe /c`
- 禁止沙箱逃逸;需要工作区外权限的命令必须用 `sandbox_permissions: require_escalated` 提权弹窗(approval policy: on-request),批准后完整权限执行

风险点:codex 沙箱指令语法变化(如新指令格式)、error 1920 场景重新出现。

## 3. 中风险回归表(升级后按表逐项人工回归)

| 场景 | 回归方法 | 判定 |
| --- | --- | --- |
| 并行工具调用 | 一次提问触发 ≥2 个工具并行执行,观察 item 事件顺序与 completion | 无交错错乱、审计条目完整 |
| Windows exec | 沙箱内执行 `powershell.exe` / `cmd.exe /c`,以及 WindowsApps 工具被拦截 | 无 error 1920,拦截行为符合 baseInstructions |
| TUI 启动 | 桌面端完整走一遍:启动 app-server → 对话 → 审批 → 关闭 | app-server 生命周期事件与 UI 状态一致 |
| hooks 校验 | 触发需要权限的操作,核对弹窗与拒绝路径 | approvalPolicy 行为与 2.2 一致 |

## 4. 升级 SOP

每次升级(含 alpha/beta)严格按以下步骤,任何一步失败即回退:

1. **换二进制**:`CODEX_SRC_DIR` 指向新版二进制目录,`npm run pack:win` 验证打包链路;开发模式临时用 `CODEX_BIN` 指到新版验证。
2. **对照清单回归**:先全量自动化(改代码前确认 2.1–2.3 字段仍被接受,必要时修正字段名——字段位置见 §1),再按 §3 表人工回归。
3. **更新绑定版本**:改本文档 §1 表;在 codex-main fork 打新基线 tag(格式 `v<x.y.z>-mindos-baseline`)。
4. **发版**:更新 `package.json` version,更新 CHANGELOG(含协议变更记录),产出发行资产并附 `codex-cli --version` 验证输出。
5. **记录决策**:升级原因、调研结论、异常项写回本文档「升级记录」表(如下)。

## 升级记录

| 日期 | 目标版本 | 基线 tag | 结论 | 高风险项状态 |
| --- | --- | --- | --- | --- |
| 2026-08-16 | 维持 0.146.0(暂不升级) | `v0.146.0-mindos-baseline` | 调研窗口:alpha.1.2 → 0.148.0-alpha.20;首次升级目标 0.148.0-alpha.20(决策点已记录,待执行) | 未变(无升级) |
| 2026-08-23 | 0.149.0 | `v0.149.0-mindos-baseline` | 预核对:ThreadStartParams/TurnStartParams/AskForApproval 三文件在 0.146.0/快照/0.149.0 逐字节一致;schema 纯增量(+32 快照段、+8 至 149,零删除零改名);新增 ThreadQueueChanged/ThreadReverted 通知(白名单外自动丢弃)。已删 historyMode 无效注入。运行时行为变化:resume/fork 还原持久化 cwd 与权限 profile、沙箱 deny 路径 fail-closed、本地项目显式信任提示——列入 §3 回归 | 已逐项对照通过 |
