# 功能文档 02 · 审批与安全

> 职责:计划/高危操作的人工审批闭环,以及贯穿全链路的写保护与行为约束(路径白名单、沙箱 worktree、写前快照、审计、计算机使用守卫)。
>
> 说明:文中行号基于当前工作区源码(`main` 分支),若发生重构请以源码为准重新核对。

---

## 1. 功能概述

- **审批流**覆盖"计划确认 → 执行中修改计划 → 审批结果回填"三段,由事件流驱动,面板 store 只负责展示。
- **安全层**是多道独立防线,互不替代:路径白名单(IPC 入口)、worktree 写隔离、写前快照、PowerShell 编码命令、按键白名单、计算机使用授权守卫、环境变量净化。

## 2. 架构与数据流

```text
            渲染进程                                主进程
┌──────────────────────────┐          ┌──────────────────────────────┐
│ taskExecutor 状态机       │          │ codex-rs / app-server         │
│  idle→planning→awaiting   │  ────▶   │  ServerRequest (approval)     │
│  _approval→executing      │ approve  │                              │
│  plan.modifyPlan 反馈     │  ◀────   │  item.autoApprovalReview      │
└───────────┬──────────────┘  推送     └──────────────┬───────────────┘
            │ approval.resolved / plan 事件           │
            ▼                                        │
   panelStore/approval.ts (展示)                     │
   stream.ts L1145 / L1156-1157 (回填)               │
┌─────────────────────────────────────────────────────┴──────────────┐
│ 安全防线:fs allowlist → worktree 隔离 → snapshot 写前快照 →          │
│          audit 审计 → computer-use guard → env-sanitize             │
└─────────────────────────────────────────────────────────────────────┘
```

## 3. 实现分解

### 3.1 审批闭环

| 位置 | 函数 | 说明 |
|---|---|---|
| `src/stores/taskExecutor.ts:106` | `setPlan` | `parsePlanItems`(L31,正则 `/^\d+[.、)]\s*/`)解析编号计划;条目 ≥2 → `awaiting_approval` |
| `src/stores/taskExecutor.ts:114` | `confirmPlan` | 批准 → `executing` |
| `src/stores/taskExecutor.ts:118` | `modifyPlan` | 修改计划:记录 `planModifyFeedback` 并回到 `planning` |
| `src/stores/taskExecutor.ts:133/137` | `setTaskStatus/getTaskStatus` | 状态写入/读取(无状态 → `idle`) |
| `chatStore.ts:541/551` | `confirmExecution`/`modifyPlan` | UI 入口 → executor + 远端同步 |
| `stream.ts:392` | `item.started` | planning/idle 态自动 `confirmPlan`(单步任务免审批) |
| `stream.ts:1145` | `approval.resolved` | 审批结果落地 |
| `stream.ts:1156-1157` | `item.autoApprovalReview.started/completed` | 自动审批事件 |
| `src/stores/panelStore/approval.ts:8/13` | `setPendingApproval`/`clearPendingApproval` | 纯展示层:写/清待审批项,**无 resolve/超时逻辑,由调用方处理** |
| `electron/server-request.ts` | `ServerRequest` | 主进程侧审批请求抽象(带 id + method 的 JSON-RPC 请求) |
| `src/stores/panelStore/subagents.ts:198` | `ensureMasterTask` | 无主任务时创建占位任务 |

**关键流程**:

1. 计划事件到达 → `setPlan` 解析 → 多步计划进入 `awaiting_approval`,面板弹出审批卡。
2. 用户批准 → `confirmPlan` → `executing`,计划开始执行;用户修改 → `modifyPlan` 反馈回 `planning`。
3. 执行中 `item.updated/completed` 驱动计划步骤状态(`setAgentPlanSteps`)。
4. `approval.resolved` 事件将远端审批结果回填;自动审批场景走 `autoApprovalReview` 事件,全程无需人工。
5. 失败重试:超过 `MAX_AUTO_RETRIES=3`(taskExecutor.ts:29)置 `failed`,自动重试次数由 `recordAutoRetry`(L142)计数。

### 3.2 错误分类(失败决策)

| 行号 | 函数/常量 | 说明 |
|---|---|---|
| `taskExecutor.ts:7` | `ErrorCategory` | 错误分类枚举 |
| `taskExecutor.ts:9-11` | 关键词常量 | timeout / format / permission 关键词(含 Windows error 1920) |
| `taskExecutor.ts:15` | `describeSandboxSpawnError` | 沙箱 spawn 错误描述 |
| `taskExecutor.ts:21` | `classifyError` | 按关键词分类,驱动是否自动重试 |

### 3.3 文件访问白名单

| 位置 | 说明 |
|---|---|
| `electron/fs/allowlist.ts`(79) | `isPathAllowed` 路径白名单校验:所有 fs handler 入口先校验再放行 |
| `electron/ipc/handlers/fs.ts`(389) | 读写操作统一过白名单 |
| `electron/ipc/handlers/git.ts:88` | GIT_DIFF 采用 `isPathAllowed` + `pathInRepo` **双闸**,拒绝 → `PERMISSION_DENIED` |
| `electron/preload.ts:34/148/151/154/168` | `sanitize` 5 处:sendMessage attachments、installMarketPlugin、cleanupDockerImage、testConnection、registerArtifact |

### 3.4 写前快照与审计

| 位置 | 说明 |
|---|---|
| `electron/codex/snapshot.ts`(122) | 写前快照:从 `apply_patch` / `edit` / `write_file` / `shell` 工具调用提取目标路径存内存;`MAX_SNAPSHOT_BYTES=2MB` |
| `electron/codex/audit.ts`(121) | 审计日志:`patchUpdated` + `apply_patch` 工具调用,缓冲 300ms(`FLUSH_DELAY_MS`)批量落盘 JSONL 到 `~/.mindos/audit` |
| `electron/codex/probe.ts`(45) | 诊断探针:环境变量 `MINDOS_PROBE=1` 或非打包态启用,JSONL 落盘 |

### 3.5 计算机使用守卫

| 位置 | 说明 |
|---|---|
| `electron/ipc/handlers/computer-use.ts:23` | `guardSession`:授权会话校验 |
| `electron/ipc/handlers/computer-use.ts:48` | `guardRoute`:双层守卫(会话 + 路由级) |
| `electron/codex/computer-use/session.ts`(60) | 授权状态机:**1 小时 TTL**,且校验「未过期 && 当前前台窗口 == 授权窗口」才放行 |
| `electron/codex/computer-use/capture.ts`(41) | 截图:`desktopCapturer` 全分辨率(物理分辨率 = bounds × scaleFactor) |
| `electron/codex/computer-use/native.ts`(282) | 原生操作(鼠标/剪贴板/SendKeys):PowerShell **`-EncodedCommand` UTF-16LE 防注入**;按键**白名单归一化**;剪贴板 + Ctrl+V 注入通道 |

### 3.6 环境与命令安全

| 位置 | 说明 |
|---|---|
| `electron/codex/app-server-manager.ts:30-33` | `PROVIDER_ENV_NAMES`:deepseek → `DEEPSEEK_API_KEY`、qwen → `QWEN_API_KEY` 映射 |
| `electron/codex/app-server-manager.ts:36+` | `WINDOWS_SANDBOX_BASE_INSTRUCTIONS`:WindowsApps AppX 别名、错误码 1920 处理 |
| app-server-manager env-sanitize | 启动 codex 时净化环境变量,避免泄漏宿主环境 |
| `electron/skills/seed.ts` + hooks | 技能/钩子注入 automated 轮次(见 08) |

## 4. 状态与数据模型

- **TaskStatus**(taskExecutor.ts:5):`idle | planning | awaiting_approval | executing | completed | failed`。
- **TaskState**(L52):每线程一份,含 `autoRetryCount`、`planModifyFeedback`。
- **PendingApproval**(panelStore/types.ts):面板展示结构,由 `setPendingApproval` 写入。
- **授权会话**:`{ 授权窗口标识, 授权时间 }`,`authorized && (now - grantedAt < 1h) && foreground === grantedWindow`。

## 5. 边界与异常处理

- 审批卡无超时逻辑:由调用方(事件流/用户)决定关闭,避免误杀长任务。
- 单步任务自动确认(planning/idle 直接 `confirmPlan`),多步才人工审批。
- 自动重试上限 3 次,超限置 `failed` 并展示分类错误。
- 白名单拒绝统一返回 `PERMISSION_DENIED`,不泄露路径信息。
- 审计与快照失败均**静默降级**,不阻塞主流程。
- 计算机使用越权(非授权窗口/过期)一律拒绝并引导重新授权。

## 6. 相关接口与文档

- 接口定义:[RENDERER-EVENT-MAIN-INTERFACES.md](../RENDERER-EVENT-MAIN-INTERFACES.md) §2.8(computer-use)、§3.5(approval 事件)。
- 任务状态机驱动源(事件流):[01-会话与消息引擎](01-session-messaging.md) §3.4/3.6。
- 写隔离细节:worktree 见 [03-文件系统与写隔离](03-filesystem.md) §3.1。

## 7. 关键文件索引

| 文件 | 角色 |
|---|---|
| `src/stores/taskExecutor.ts` | 任务状态机与错误分类 |
| `src/stores/panelStore/approval.ts` | 审批展示层 |
| `electron/server-request.ts` | 主进程审批请求抽象 |
| `electron/fs/allowlist.ts` | 路径白名单 |
| `electron/codex/snapshot.ts` / `audit.ts` / `probe.ts` | 写前快照/审计/探针 |
| `electron/ipc/handlers/computer-use.ts` + `codex/computer-use/*` | 计算机使用守卫与操作 |
| `electron/preload.ts` | 参数 sanitize 5 处 |
