# 功能文档 01 · 会话与消息引擎

> 职责:管理线程(会话)的 CRUD、消息发送与接收、事件驱动的消息渲染、断线恢复、快照与回滚。这是整个聊天界面的核心。
>
> 说明:文中行号基于当前工作区源码(`main` 分支),若发生重构请以源码为准重新核对。

---

## 1. 功能概述

- **线程(Thread)** 是会话的基本单位,与远端 codex 服务的会话一一对应,渲染层通过 `threadId` 组织消息与面板数据。
- **消息**不主动轮询:远端推送 JSON-RPC 事件,经主进程转发,渲染层按事件类型增量构建/更新消息与面板。
- 本引擎横跨渲染进程(`src/stores/chat/`)、主进程(`electron/ipc/handlers/threads.ts`、`electron/codex/thread-cache.ts`)。

## 2. 架构与数据流

```text
┌────────────┐   sendMessage   ┌─────────────────┐
│ UI 组件     │ ──────────────▶ │ chatStore        │
└────────────┘                 │  (队列/图片注入)  │
                               └────────┬────────┘
                                        │ performSend (automated 或 user)
                                        ▼
        ┌────────────────  electron/ipc  ────────────────┐
        │ threads.ts handler ─▶ codex-rs ─▶ 远端服务       │
        └────────────▲────────────────────────────────────┘
                     │ 推送事件 (JSON-RPC)
        ┌────────────┴────────────┐
        │ ws-client → event-converter → event-forwarder → preload │
        └────────────▲────────────┘
                     │ 订阅 (on:turn.* / on:item.* ...)
┌────────────────────┴──────────────────────────────────────────┐
│ EventRouter (chatStore 装配) ──▶ messageStore / subagent      │
│  ├─ stream.ts 各 case:item/turn/approval/error/warning        │
│  ├─ subagent.ts:子代理事件归集到父线程面板                     │
│  └─ 输出:threadMessages / streamingItems / panelStore 各面板   │
└───────────────────────────────────────────────────────────────┘
```

## 3. 实现分解

### 3.1 线程 CRUD 与切换

| 位置 | 函数 | 说明 |
|---|---|---|
| `src/stores/chat/threadStore.ts:74` | `useThreadStore` | 线程列表/当前线程;`createThread/switchThread/deleteThread/renameThread/getThreadIntentChain` |
| `src/stores/chat/chatStore.ts:217` | `createThread` | 创建线程 → 自动激活 |
| `src/stores/chat/chatStore.ts:245` | `switchThread` | 切换线程:loadDirectory + `onThreadLoaded` 链路 |
| `src/stores/chat/chatStore.ts:265` | `deleteThread` | 删除线程(远端 + 本地) |
| `src/stores/chat/chatStore.ts:287` | `renameThread` | 重命名 |
| `src/stores/chat/chatStore.ts:371` | `onThreadLoaded` | 线程激活回调:重置会话摘要 → 加载快照 → 恢复模式/模型/推理强度 → 并行预热技能指令与项目上下文 |
| `electron/ipc/handlers/threads.ts:256`(文件总长) | 线程 handler 组 | `thread:*` 通道:create/list/messages/delete/rename/intent-chain |
| `electron/codex/thread-cache.ts:343` | 线程缓存 | 内存 LRU + 逐 key 磁盘文件 + 串行写队列;旧版单文件自动迁移;`invalidateMeta`(L305)/`clear`(L318)/`getSharedThreadCache` 惰性共享(L340) |

**关键流程 —— 切换线程**:`switchThread` → messageStore load hook(`activateThread` → 按 project.localPath 加载目录树,兜底 cwd) → 空线程则 `loadThreadMessagesFromServer` → `onThreadLoaded` 恢复快照并预热上下文。

### 3.2 发送链路(含队列与图片注入)

| 位置 | 函数 | 说明 |
|---|---|---|
| `chatStore.ts:334` | `sendMessage` | 发送入口:进入队列或直接发送;图片注入去重窗口 `IMAGE_INJECT_DEDUP_MS=30s`(L29) |
| `chatStore.ts:470` | `performSend` | 实际发送;`force` 参数位被自动化引擎复用 |
| `chatStore.ts:645/653/662` | `flushQueuedSends`/`cancelQueuedSend`/`queuedSendCount` | 待发队列:连接恢复后冲刷、可取消、可计数 |
| `chatStore.ts:693` | `flushPendingImageInjects` | 冲刷待注入图片,超时 `IMAGE_INJECT_FLUSH_TIMEOUT_MS=4s`(L31) |
| `chatStore.ts:139` | `initModelFromSettings` | 用 settings 默认模型初始化连接 |
| `chatStore.ts:137` | `checkCodex` | 探测 codex 可执行文件是否存在 |

**关键流程**:

1. UI 调用 `sendMessage`(文本 + 可选附件/图片)。
2. 若存在待发队列(断线或竞态)则入队,否则立即 `performSend` 走 `threads` 通道发送。
3. 远端开始执行,随后事件流驱动消息渲染(见 3.3)。
4. 发送失败 → `handleStepFailure` 构造重试 prompt(`buildRetryPrompt`,messageStore.ts:258),由用户或自动化决定是否重试。

### 3.3 事件路由与消息渲染(核心)

| 位置 | 函数 | 说明 |
|---|---|---|
| `chatStore.ts:444-461` | `createEventRouter` | 装配全部 stream 依赖(含 `requestImageInjection`/`flushPendingImageInjects` 图片注入回调) |
| `src/stores/chat/events/index.ts:7-10` | 事件分流 | `sub_agent_activity` → `handleSubagentEvent`,其余 → `handleStreamEvent` |
| `src/stores/chat/events/stream.ts` | 主 case 表 | 见下方 3.4 |
| `src/stores/chat/events/subagent.ts:6` | `createSubagentHandler` | 子代理事件(见 3.5) |
| `messageStore.ts:201` | `pushMessageToThread` | 激活线程直接 push,否则写 `threadMessages` 缓存;尾部 `touchThread` |
| `messageStore.ts:212` | `findMessageInThread` | 按 id 查消息(激活/缓存双通道) |

### 3.4 事件 → 消息映射表(events/stream.ts)

| 行号 | 事件 | 输出/变更 |
|---|---|---|
| 365 | `thread.tokenUsage.updated` | `recordTokenUsage` 代币统计 |
| 373 | `sub_agent_activity` | 转发子代理处理 |
| 392 | `item.started` | planning/idle 自动 `confirmPlan`;三类子分支(见下) |
| 491 | `item.updated` | 增量打字机 |
| 554 | `item.completed` | 终态落卡 |
| 814 | `rawResponseItem.completed` | 原生工具调用兜底 |
| 960 | `turn.completed` | 清 pending、flushTyping、组装最终 assistant 消息、原型卡片注入 |
| 1063 | `turn.failed` | 失败收尾(见 3.6) |
| 1105 | `error` | `useErrorStore().handleError`(willRetry → info,否则 warning) |
| 1112 | `warning` | `data.guardian` 存在时高亮警告 |
| 1123 | `turn.plan.updated` | 计划步骤状态映射 → `setAgentPlanSteps` |
| 1145 | `approval.resolved` | 审批结果落地 |
| 1156-1157 | `item.autoApprovalReview.started/completed` | 自动审批事件 |

**item 子分支(stream.ts)**:

| 行号 | 子类型 | 说明 |
|---|---|---|
| 397 | `item.started · commandExecution` | 推 action 卡 + running、`pendingShellItems` 登记、console 日志、masterTask、`registerShellWriteChange` |
| 430 | `item.started · fileChange` | 逐条 `addFileChange(pending)`;html 原型路径 → `upsertPrototypeCard` |
| 448 | `item.started · dynamicToolCall` | 推卡 + `pendingToolCallItems/Logs` 登记;`mindos_view_image` → 图片注入 |
| 492 | `item.updated · todoList` | 计划卡 upsert |
| 503 | `item.updated · reasoning` | `pushTyping` 增量 |
| 514 | `item.updated · agentMessage` | 打字机 + `[[action:{json}]]` 跨 delta 累积解析(动作块不进可见内容) |
| 535 | `item.updated · fileChange` | `toUnifiedDiff + diffStats` → `updateFileChange(in_progress)` |
| 555 | `item.completed · collabAgentToolCall` | `syncSubagentStatus` 批量 + `appendSubagentInteraction` |
| 588 | `item.completed · agentMessage` | 兜底补写打字机;planning 态 → `executor.setPlan`;`updateSessionSummary` |
| 618 | `item.completed · reasoning` | 空块补写 + `appendThinking` |
| 642 | `item.completed · commandExecution` | 反查匹配卡片、`displayOutput`(失败前缀 `[exit N]`)、console 状态更新 |
| 719 | `item.completed · dynamicToolCall` | `call_id` 关联,丢失时反向匹配同名 running 卡;`mindos_send_file` → 文件卡 |
| 778 | `item.completed · fileChange` | 终态 applied/rejected/failed,有 diff 则统计 |
| 804 | `item.completed · todoList` | 追加计划卡 |
| 815 | `raw · function_call/custom_tool_call/local_shell_call/tool_search_call` | 工具卡 + pending 登记 |
| 877 | `raw · dynamicToolCall` | 补 done 态卡片 |
| 901 | `raw · function_call_output 等` | `input_image` → 图片卡;`detectArtifact + detectShellHtmlWrites` → 原型卡 |

### 3.5 子代理事件归集(subagent.ts)

| 行号 | 函数/分支 | 说明 |
|---|---|---|
| 6 | `createSubagentHandler` | parentTid = activeThreadId 或 tid;子代理数据**全部落在父线程面板** |
| 18 | `item.started` | 仅 commandExecution:父线程推 running action 卡 + console 日志 |
| 46 | `item.completed` | 四子类:commandExecution(47-94,stdout 逐行、**从后往前**匹配最近 running 同命令卡片防错配 L74-83);reasoning/agentMessage(95-115);plan(116-128) |
| 131 | `item.updated` | 仅 todoList → `setAgentPlanSteps(todo-N)` |
| 144 | `turn.failed` | `setAgentError` + errorStore warning |

### 3.6 超时、失败与断线恢复

| 位置 | 函数 | 说明 |
|---|---|---|
| `messageStore.ts:219-256` | `resetStreamTimeout` | **180s 超时兜底收尾**:flushTyping → 组装 assistant 消息 → push 超时 system 消息 → `handleStepFailure` → `abortTurn` |
| `messageStore.ts:258` | `buildRetryPrompt` | 失败重试 prompt 构造 |
| `messageStore.ts:601-620` | `handleServerDisconnected` | 复位流状态、清超时/打字机、push 断线 system 消息、`failRunningSubagents` |
| `messageStore.ts:622-645` | `handleServerRestored` | 收集 tids → `refreshIncremental`,失败回退 `loadThreadFull`;合并 `pendingLocal` 本地未发送内容 |
| `chatStore.ts:517/530` | `handleServerDisconnected/Restored` | 转发 messageStore 同名逻辑 |

**失败收尾链路(turn.failed,stream.ts:1063-1104)**:appendFileChangeSummary → 原型卡注入 → `ensureMasterTask` → `setAgentPlanSteps(buildPlanStepsFromActions)` → 错误按 willRetry 降级为 info。

### 3.7 快照、恢复与回滚

| 位置 | 函数 | 说明 |
|---|---|---|
| `chat/restore.ts:51` | `restoreFromSnapshot` | 核心恢复:消息/面板/意图链;`guessTool`(L28)与 `recomputeStepDescription`(L35)重算类型 |
| `chat/restore.ts:86` | `restorePanelStateFromSnapshot` | 面板状态恢复 |
| `chat/snapshot.ts:35` | `saveSnapshotNow` | 立即保存 |
| `chat/snapshot.ts:54` | `scheduleSnapshotSave` | 防抖调度 |
| `chat/snapshot.ts:67` | `isTaskActive` | 有流式活动时保活心跳(10s 心跳,`SNAPSHOT_HEARTBEAT_MS`) |
| `chatStore.ts:308` | `rollbackToTurn` | 回滚:`buildRollbackPayload` 把旧线程变更/搜索/控制台浓缩为单条系统消息 → 新建线程以 automated 模式注入浓缩历史 |
| `chatStore.ts:594/613` | `buildCurrentSnapshot`/`restoreFromSnapshot` | 快照构建/恢复入口 |
| `sessionStore.ts:131/149/163/177` | `saveSnapshot/loadLatestSnapshot/loadSnapshotByThreadId/cleanupOldSnapshots` | 会话快照持久化(JSON 深拷贝 → `api.saveSessionSnapshot`;7 天过期清理) |

### 3.8 自动化引擎与事件总线

| 位置 | 说明 |
|---|---|
| `chatStore.ts:463-495` | `createAutomationEngine`:performSend 走 force 位;pushSystemMessage / 取尾 30 条消息 / `setMessageMeta` 写 decisionSummary |
| `chatStore.ts:497-503` | 事件总线七路注册:`git:commit`、`git:branch-switched`、`agent:turn-finished`、`agent:turn-completed`、`task:completed`、`git:before-commit`、`fs:after-save` |

### 3.9 上下文注入(contextStore)

| 行号 | 函数 | 说明 |
|---|---|---|
| `contextStore.ts:6` | 四段 ref | skillInstructions / projectContext / recentSnapshotSummary / currentSessionSummary |
| `contextStore.ts:12` | `fullContext` | computed 拼接:技能 → 项目 → 快照摘要 → 会话摘要 |
| `contextStore.ts:59` | `refreshSkillInstructions` | `api.readSkills` 拼接提示文本 |
| `contextStore.ts:80` | `refreshProjectContext` | 项目上下文读取 |
| `contextStore.ts:118` | `buildFileTreeSummary` | 文件树摘要(maxEntries 限制) |
| `contextStore.ts:146/150` | `loadRecentSnapshotSummary`/`resetSessionSummary` | 快照摘要/会话摘要维护 |

## 4. 状态与数据模型

- **消息缓存双通道**:`messages`(激活线程)+ `threadMessages`(线程→消息 Map 缓存),切换线程零加载。
- **流式条目**:`streamingItems` 保存进行中 item,`turn.completed` 时收拢为最终 assistant 消息(prototypeCard 以 `msg-{nextMsgId()}` 注入)。
- **任务状态机(taskExecutor.ts:5)**:`idle → planning → awaiting_approval → executing → completed / failed`,每线程独立 `TaskState`(`createTaskState` L52,含 `autoRetryCount`;`MAX_AUTO_RETRIES=3` L29)。
- **代币记录**:`recordTokenUsage` / `getThreadTokenUsage` 挂在 messageStore,事件 `thread.tokenUsage.updated` 驱动。
- **队列**:`enqueueSend/flushQueuedSends/cancelQueuedSend/queuedSendCount` 保障断线期间发送不丢失。

## 5. 边界与异常处理

- 180s 流超时强制收尾,防止无限挂起(多代理长任务由 `noteTaskActivity` 保活)。
- 断线:只复位状态 + 提示 + 子代理标记失败;**恢复时本地 pending 与远端刷新合并**,用户输入不丢。
- 增量刷新失败 → `loadThreadFull` 全量兜底(吞 not materialized / no rollout 错误)。
- 热数据超 `HOT_LIMIT` → `evictOverflow` 冻结冷归档(`loadOlder` 历史回翻)。
- 子代理命令完成匹配采用**从后往前**策略,防止同命令连发错配。
- `IMAGE_INJECT_DEDUP_MS=30s` 防图片重复注入;`GIT_SYNC_MAX_FILES` 防 git 同步列表爆炸。

## 6. 相关接口与文档

- 接口定义:见 [RENDERER-EVENT-MAIN-INTERFACES.md](../RENDERER-EVENT-MAIN-INTERFACES.md) §2.1-2.6(线程/消息/订阅)、§3(事件层)。
- 事件推送管线(主进程):见 [07-生命周期与持久化](07-lifecycle.md) §3.3(ws-client/event-converter/event-forwarder)。
- 断线重连后的状态机联动:见 [07-lifecycle.md](07-lifecycle.md) §3.1(app-server 状态机)。
- 任务状态机与审批:见 [02-审批与安全](02-approval-security.md)。

## 7. 关键文件索引

| 文件 | 角色 |
|---|---|
| `src/stores/chat/chatStore.ts`(747) | 引擎总装:发送/队列/事件路由/自动化 |
| `src/stores/chat/messageStore.ts`(694) | 消息缓存/渲染/超时/断线恢复 |
| `src/stores/chat/events/stream.ts`(1164) | 事件 → 消息映射主表 |
| `src/stores/chat/events/subagent.ts`(152) | 子代理事件归集 |
| `src/stores/chat/threadStore.ts` / `restore.ts` / `snapshot.ts` / `typing.ts` / `gitSync.ts` | 线程/恢复/快照/打字机/git 同步 |
| `electron/ipc/handlers/threads.ts` | 线程 IPC 入口 |
| `electron/codex/thread-cache.ts` | 线程消息缓存(主进程) |
