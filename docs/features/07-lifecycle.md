# 07 · 生命周期与持久化

> 职责:管理 codex CLI `app-server` 子进程的启动/退出、WebSocket 连接与断线重连状态机、线程数据缓存(内存 LRU + 磁盘)、patch/apply_patch 审计日志,以及全局数据在磁盘与 localStorage 中的持久化。
>
> 行号免责声明:本文所有行号基于当前工作区 main 分支代码,若后续改动请以源码为准。

## §1 功能概述

系统以 **子进程 + WebSocket JSON-RPC** 的方式承载 codex 运行时:

1. 主进程 spawn `codex app-server --listen ws://127.0.0.1:<port>` 子进程,等待 stderr 出现 `listening on:` 横幅后建立 WebSocket 连接;
2. `WsClient` 封装 JSON-RPC 2.0:请求/响应、通知、ServerRequest(服务端向客户端发起、需回响应)三类消息;
3. 服务端推送的 `item/*`、`turn/*`、`thread/*` 通知经 `event-converter.ts` 归一化为 `CodexStreamEvent`,转发给渲染层 store;
4. 断线后按指数退避(1s → 30s)自动重启子进程并重连,期间 `threadRead` 等数据路径在等待队列上排队,`readyWaiters` 统一放行;
5. 线程 turns 采用 **基线 + 增量** 两级缓存(内存 LRU 256 条 + 磁盘逐 key 文件,TTL 7 天),跨重启保留;
6. `apply_patch` 与文件变更通知落 JSONL 审计日志(`~/.mindos/audit/apply-patch-YYYYMMDD.jsonl`,300ms 合并刷盘);
7. 应用退出时按序清理:WS 连接 → taskkill 子进程树 → 终端管理器 dispose。

## §2 架构与数据流

```text
┌────────────────────────── 主进程 ──────────────────────────┐
│  main.ts  L32: const appServerManager = new AppServerManager() │
│        │ start(options)  L428                                 │
│        ▼                                                      │
│  AppServerManager                                            │
│  ① findCodexPath L297(打包内置 / CODEX_BIN / where / 常见路径) │
│  ② pickFreePort L238(默认 9876,MINDOS_APP_SERVER_PORT 覆盖)   │
│  ③ spawn codex app-server --listen ws://127.0.0.1:<port>      │
│     sanitizeSpawnEnv L220(剔除 PATH 中 WindowsApps 别名目录)   │
│     providerEnv(PROVIDER_ENV_NAMES L30,API Key 注入)          │
│  ④ stderr 检测 "listening on:" L511 → connectWs L594          │
│     ├─ WsClient(url) → initialize(15s) → initialized 通知      │
│     ├─ onNotification L599:                                   │
│     │   ├─ auditNotification L618(审计)                       │
│     │   ├─ fileSnapshotOnToolCall L621(快照)                  │
│     │   ├─ mcpServer/startupStatus/updated → McpManager L625  │
│     │   ├─ turn/completed|failed → 缓存失效 L630-635          │
│     │   └─ convertServerNotification L638 → options.onEvent   │
│     └─ onServerRequest L656 → onServerRequest → 渲染审批      │
│  ⑤ close → emitStatus('reconnecting') → scheduleRestart L569  │
│     指数退避重启 → 回到 ①                                   │
│                                                               │
│  threadRead L779(基线命中→零 API / 翻页水合→resolveTurns)     │
│    └── getSharedThreadCache L340(userData/thread-cache,7天)   │
│  audit.ts 300ms 合并 → ~/.mindos/audit/apply-patch-*.jsonl    │
│                                                               │
│  退出链:window-all-closed L395 → cleanup L1017                 │
│        before-quit L404 / will-quit L416 → disposeAllTerminals │
└──────────────────────────────────────────────────────────────┘
                          │  ws 通知(JSON-RPC)
                          ▼
┌────────────────────── 渲染层 ──────────────────────┐
│  appServerStore.ts(46 行)                         │
│  init → getAppServerStatus + onAppServerStatus     │
│  handleStatus L16:ready 上升沿 → chat.handleServerRestored │
│                下降沿 → chat.handleServerDisconnected │
└──────────────────────────────────────────────────┘
```

## §3 实现分解

### 3.1 app-server-manager.ts(1041 行)——子进程生命周期与数据路由

**常量与策略**(L20-202):

| 符号 | 行号 | 说明 |
| --- | --- | --- |
| `AppServerStartOptions` | L20 | port / configOverrides / onEvent / onServerRequest / onReady / onClose / onStatus |
| `PROVIDER_ENV_NAMES` | L30 | provider → 环境变量名(deepseek→`DEEPSEEK_API_KEY`、qwen→`QWEN_API_KEY`),仅 spawn 时注入 |
| `WINDOWS_SANDBOX_BASE_INSTRUCTIONS` | L36 | Windows 受限沙箱约束指令:WindowsApps 下 AppX 别名(reparse point)在受限 token 下报 Windows error 1920 |
| `ON_REQUEST_APPROVAL_POLICY` | L58 | `'on-request'`:所有需审批操作均弹窗征询(见 02 文档) |
| `MINDOS_SEND_FILE_TOOL` / `MINDOS_SEND_FILE_INSTRUCTIONS` | L64 / L82 | 动态工具:AI 主动把文件以卡片发送到对话流,无需批准 |
| `MINDOS_VIEW_IMAGE_TOOL` / `MINDOS_VIEW_IMAGE_INSTRUCTIONS` | L93 / L112 | 动态工具:AI 查看本地图片,自动注入对话 |
| `COMPUTER_USE_INSTRUCTIONS` | L120 | computer-use 单步协议指令(仅 win32 注入) |
| `windowsSandboxBaseInstructions` | L143 | win32 返回沙箱指令,其余平台 undefined |
| `buildThreadStartParams` | L148 | thread/start 参数覆盖:强制审批策略 + 默认上下文窗口 + 自动压缩阈值 + 沙箱/发送文件/看图片/computer-use 指令与动态工具 + `historyMode: 'paginated'` |
| `buildThreadResumeParams` | L178 | thread/resume 同套覆盖,支持 cwd / runtimeWorkspaceRoots / sandbox |
| `READY_TIMEOUT_MS` | L197 | 30s:等 "listening on" 横幅上限 |
| `THREAD_BASE_FRESH_MS` | L198 | 60s:基线超过 60s 视为过期,全量读强制重建 |
| `THREAD_LIST_TTL_MS` / `MODEL_LIST_TTL_MS` | L199 / L200 | 1h / 24h 列表缓存 TTL |
| `RECONNECT_BASE_MS` / `RECONNECT_MAX_MS` | L201 / L202 | 重连退避 1s → 30s |

**spawn 环境净化**(L210-236):

- `isWindowsAppsAliasDir` L210:判断 PATH 条目是否为 WindowsApps 目录(别名是 reparse point,受限 token 下 CreateProcessAsUserW 失败,codex 沙箱内 `which("pwsh")` 命中别名即报 1920);
- `sanitizeSpawnEnv` L220:剔除 PATH 中所有 WindowsApps 别名目录;净化无变化或 PATH 被剔空时返回原对象兜底。

**类状态**(L259-277):

- `threadCwd` Map(L275):threadId → 启动/恢复时注入的 cwd,供 patch 审计记录工作区上下文;
- `providerApiKeys` Map(L273):`setProviderApiKey` L290 记录用户提供的 provider Key,仅 spawn 时注入;
- `pendingRefresh` Set(L268):一轮结束置增量刷新标记(不再全清缓存);
- `readyWaiters`(L272):断线重连期间所有 `waitForReady` 等待者共享同一 waiter,重启完成统一放行(L393-426)。

**start 启动流程**(L428-553):

1. `findCodexPath` L297(静态缓存):打包时 `process.resourcesPath/codex.exe`;否则 `CODEX_BIN` env → win32 `where` → 常见安装路径(APPDATA npm / WinGet Links / ProgramFiles / LOCALAPPDATA Programs)→ `npm prefix -g` 兜底(L304-343);
2. 端口:默认 9876,`MINOS_APP_SERVER_PORT` env 可覆盖,`pickFreePort` L238 失败时回退 OS 分配;
3. 参数:`app-server --listen ws://127.0.0.1:<port>` + configOverrides 逐项 `-c key=value`;
4. 环境:`NO_COLOR=1`、`RUST_LOG`(无则 `codex_api=info`)+ provider 环境变量 + PATH 净化(L460-465);`.cmd/.bat` 时 `shell: true`(win32);
5. 就绪判定:stderr 中出现 `listening on:`(L511)→ `connectWs`(L594)→ `onReady` 回调 → `okOnce`(emitStatus('ready') + settleReadyWaiters);30s 无横幅或进程提前退出 → `failOnce` + `cleanup`。

**connectWs 连接与订阅**(L594-671):

- 新建 `WsClient`,注册 `onNotification` 回调 L599:
  - threadId 兜底解析(L601-610):`params.threadId` → `turn.threadId` → `turn.thread_id` → `thread.id` → `thread.threadId` → `sessionId`;
  - `auditNotification` L618(见 3.3);`fileSnapshotOnToolCall` L621(非 Git 项目红绿 diff 快照,见 03 文档);
  - `mcpServer/startupStatus/updated` → 转发给 McpManager L625(见 05 文档);
  - `turn/completed` 或 `turn/failed` → `pendingRefresh.add(threadId)` + thread-cache `invalidateMeta` + list-cache `invalidate('thread-list')` L630-635(事件驱动失效,不靠 TTL);
  - `item/*`、`rawResponseItem/*`、`turn/*`、`thread/*`、`error`、`sub_agent_activity` → `convertServerNotification`(3.5)→ `options.onEvent(event, threadId)` L637-642;
- `onClose` L645:置 `reconnecting` 状态 + `scheduleRestart`(指数退避);
- 握手:`initialize`(15s,clientInfo MindOS Desktop 0.1.0,capabilities.experimentalApi)+ `initialized` 通知(L662-667)。

**断线重连**(L555-585):

- `scheduleRestart` L569:delay = min(30s, 1s × 2^attempt),失败后递归排程;`restart` L555:复位计数 → cleanup → 重新 start。

**数据路径**(L673-1015):

| 方法 | 行号 | 说明 |
| --- | --- | --- |
| `threadStart` | L678 | 注入动态工具/指令;记录 `threadCwd`;失效 thread-list 缓存 |
| `turnStart` | L702 | 强制 `approvalPolicy`,支持 model / effort |
| `turnInterrupt` / `subagentInterrupt` | L716 / L723 | 后者 turnId 传空串 |
| `threadList` | L730 | 全量(limit ≥ 500)走 refreshThreadList + stale 缓存兜底(L741);分页直呼 |
| `modelList` | L758 | stale-while-revalidate:24h 内直接返回,后台刷新 |
| `threadRead` | L779 | 见 3.2 |
| `readThreadPage` | L916 | 分页读 turns,不写基线;legacy 保底 `thread/read(includeTurns:true)` |
| `threadResume` | L987 | 记录 workspace.cwd → threadCwd |
| `threadDelete` | L996 | 清 threadCwd / list 缓存 / thread-cache `clear` |
| `threadSetName` | L1007 | list 缓存 + meta 缓存双失效 |

**cleanup 清理**(L1017-1040):清重连定时器 → `wsClient.dispose()` → win32 `taskkill /F /T /PID`(进程树)或 SIGTERM → 状态置 `disconnected` → `settleReadyWaiters`(reject 所有等待者)。

### 3.2 threadRead 水合策略(app-server-manager.ts L779-913)

- **meta 变体**(includeTurns=false):独立缓存 key(`turns:0`);线程未物化(错误含 `not materialized`)等价空线程返回 `{ thread: { threadId } }`(L803-807);
- **基线命中**:`readBase` + `sinceInRange`(since 为空、等于 base.startId 或存在于 base.turns,且基线 60s 内新鲜 L817-819)+ 非空基线 → 零 API 切分返回;若带 `pendingRefresh` 标记则先执行 `refreshThreadTurns`(desc 翻页遇 endId 即停,≤2 页)再切分(L828-841);
- **翻页水合**(无基线/since 洞):`thread/read(includeTurns:false)` 取 meta + `thread/turns/list`(desc、50/页、itemsView:full)逐页翻;遇 `base.endId` 停止(增量,`sawEndId`),整条重建时新建基线;`resolveTurns`(thread-cache.ts L81)三种情形:
  - 有基线且翻到 endId → mergeTurns 按 turn id 去重合并,基线上移;
  - 有基线未遇 endId → 全量重建基线;
  - 无基线 → 全新基线;
- **legacy 保底**:turns/list 报 `no rollout found` / `not supported` / `Invalid params` 时回退 `thread/read(includeTurns:true)` 一次性全量(L900-911 / L936-952)。

### 3.3 ws-client.ts(243 行)——JSON-RPC 传输层

- 类型:JsonRpcRequest L6、JsonRpcResponse L13、JsonRpcNotification L20、JsonRpcServerRequest L26;
- `class WsClient` L37:单 socket + pending Map(id → resolve/reject/timeout)+ connectPromise 去重(L45, L63-72);
- `doConnect` L74:`isCurrent` 拦截过期 socket;连接建立前 error → 置 null + reject;close → rejectAllPending + closeHandler(由 AppServerManager 决定是否重启);
- `handleMessage` L116 分发顺序:先判 ServerRequest(有 id 且有 method,服务等客户端 Response,走 `serverRequestHandler`)→ Response(有 id,resolve/reject)→ Notification(有 method);
- `call` L169:默认 30s 超时,id 用 `randomUUID`;`dispose` L206:rejectAllPending('disposed') + removeAllListeners + close;`ensureOpen` L216:未连接先 connect;`sendNotification` L224 / `sendResponse` L232 / `sendErrorResponse` L238。

### 3.4 thread-cache.ts(343 行)——两级线程缓存

- 常量:TTL 7 天 L6、内存 LRU 256 条 L7;
- key 设计:`makeThreadCacheKey` L19 → `id:<threadId>|turns:0|1|since:<sinceId>`;基线 `makeThreadBaseKey` L27 → `id:<threadId>|turns:1|base`;
- 磁盘布局:目录 `userData/thread-cache/<safeFileName(threadId)>/`,每个 key 独立文件 `<key 的 hex>.json`(L122-124);旧版单文件 `<threadId>.json` 启动访问时惰性迁移,损坏即删(L163-186);
- 内存层:LRU Map(memGet 访问提升队尾,memPut 超限淘汰队头,L130-147);
- 写串行化:每线程一条 promise 链锁(L149-160),避免并发 read-modify-write 丢条目;
- 写盘:`writeEntryFile` L188 采用 tmp + rename 原子替换,Windows rename 偶发覆盖失败时退化 先删后改名;
- 过期条目惰性删除(L234-239 / L276-280);`invalidateMeta` L305 / `clear` L318(递归删线程目录);
- `getSharedThreadCache` L340:惰性共享单例,app ready 后才安全调用(`app.getPath('userData')`)。

### 3.5 event-converter.ts(216 行)——服务端通知 → 渲染事件

`convertServerNotification(method, params)` L3 单函数映射:

| 服务端 method | 渲染事件 | 行号 |
| --- | --- | --- |
| thread/started | thread.started | L10 |
| turn/started | turn.started | L17 |
| turn/completed(status=failed) | turn.failed(带 error.message) | L40-48 |
| turn/completed(status=interrupted) | turn.completed(status:'interrupted') | L50-52 |
| turn/completed(其余) | turn.completed(usage / threadId / turnId 多字段兜底) | L20-53 |
| error | error(willRetry) | L56 |
| serverRequest/resolved | approval.resolved | L69 |
| warning / guardianWarning | warning(guardian 标记) | L79-89 |
| turn/plan/updated | turn.plan.updated(step/status 归一) | L91 |
| item/started / updated / completed | item.* | L115-137 |
| rawResponseItem/completed | rawResponseItem.completed | L139 |
| item/agentMessage/delta、item/reasoning/textDelta、item/commandExecution/outputDelta、command/exec/outputDelta | item.updated(delta + item 类型) | L147-172 |
| item/reasoning/summaryTextDelta | **null(忽略)** | L155 |
| item/fileChange/patchUpdated | item.updated(kind 解包为 type) | L174 |
| thread/tokenUsage/updated、thread/turn/tokenUsage | thread.tokenUsage.updated | L195 |
| sub_agent_activity | sub_agent_activity | L207 |
| 其余 | null | L213 |

### 3.6 audit.ts(121 行)——apply_patch 审计日志

- `DEFAULT_AUDIT_DIR` L11:`~/.mindos/audit`(`setAuditDirForTest` L40 供测试注入);
- `auditFileFor` L45:按天轮转 `apply-patch-YYYYMMDD.jsonl`;
- `auditNotification` L67 入口(由 3.1 connectWs L618 调用):
  - `item/fileChange/patchUpdated` → 记录 `{t, method, threadId, cwd, status, changes}`(changes 由 `extractChanges` L52 提取,kind 解包为 type);
  - `item/tool/call` → 仅 `toolName === 'apply_patch'` 记录 `{toolName, toolInput}`;
- 落盘:`writeAuditEntry` L92 入缓冲 + 300ms 定时器合并;`flushNow` L102 JSONL 同步追加;`flushAuditSync` L113 供进程退出/测试清空缓冲;所有失败静默降级。

### 3.7 渲染侧 appServerStore.ts(46 行)——连接状态消费

- `status` ref L7(初始 `{state:'starting'}`);`isConnected` L11 = state==='ready';`isDisconnected` L12 = disconnected|error;
- `handleStatus` L16 边沿触发:进入 ready → `chat.handleServerRestored()` + `codexInstalled === false` 时重查 `checkCodex()`;离开 ready → `chat.handleServerDisconnected()`;
- `init` L29:`getAppServerStatus()` 拉快照 + `onAppServerStatus` 订阅;`dispose` L40 退订。

### 3.8 main.ts 生命周期接线(electron/main.ts)

| 行号 | 说明 |
| --- | --- |
| L32-33 | `new AppServerManager()`、`new McpManager(appServerManager)`(见 05) |
| L188 | `registerIpcHandlers(mainWindow, appServerManager, mcpManager)` |
| L237 | `appServerManager.start({...})`(onReady / onEvent / onStatus 接线) |
| L278 | ServerRequest 响应回传 `sendServerResponse` |
| L308 | 就绪日志 + 端口 |
| L350 | 启动期 `threadList({limit:500})` 预热 |
| L395-398 | `window-all-closed` → `appServerManager.cleanup()` |
| L404 | `before-quit`(取消事件可拦退出) |
| L416-419 | `will-quit` → `disposeAllTerminals()`(见 06) |

## §4 状态与数据模型

### 4.1 连接状态机(AppServerStatusPayload,channels.ts L232)

```text
        start()                       stderr "listening on:"
 [disconnected] ─────────► [starting] ──────────────► [ready]
      ▲                        │                         │
      │                   30s 超时 /                close/异常
      │                   spawn 失败                    ▼
      └──── cleanup() ◄── [error]           [reconnecting] ──► scheduleRestart(退避) ──► start()
```

状态值:`'starting' | 'ready' | 'reconnecting' | 'disconnected' | 'error'` + 可选 message;渲染侧经 `APP_SERVER_STATUS`(channels.ts L71)推送、`APP_SERVER_STATUS_GET`(L72)拉取。

### 4.2 持久化清单

| 位置 | 内容 | 管理方 |
| --- | --- | --- |
| `userData/thread-cache/<threadId>/<keyhex>.json` | 线程 turns/meta 两级缓存(7 天 TTL) | thread-cache.ts 3.4 |
| `~/.mindos/audit/apply-patch-YYYYMMDD.jsonl` | apply_patch / 文件变更审计 | audit.ts 3.6 |
| `userData/plugins.json`(STORAGE_KEY `mindos-plugins`) | 插件清单(内置版本号 key) | pluginStore.ts(见 05) |
| `~/.codex/config.toml` | MCP server 合并导出(config/mcpServer 段) | mcp-export.ts(见 05) |
| `userData`(worktree manifest) | 写隔离工作区注册表 | worktree.ts(见 03) |
| `localStorage`(mindos-plugins 等 key) | 渲染侧会话持久化 | 渲染 store |
| 进程内 Map(threadCwd / providerApiKeys / pendingRefresh) | 会话级,重启即失 | AppServerManager L268-277 |

## §5 边界与异常

- **启动超时**:30s 无 `listening on:` 横幅 → failOnce + cleanup(子进程按树清理);
- **握手超时**:`initialize` 15s;
- **断线重连**:指数退避 1s→30s;重连期间数据调用经 `ensureReadyOrWait` L423 排队,`waitForReady` 默认 30s 超时 reject(L393);
- **线程未物化**(`not materialized`):turns 相关调用等价空线程返回,不视为故障(L786-787 / L918-919);
- **基线新鲜度**:60s 过期基线强制重建(L817);空基线不可信强制翻页(L821);
- **legacy 兼容**:turns/list 失败保底 `thread/read(includeTurns:true)`(L900 / L936);
- **Windows 沙箱**:PATH 净化剔除 WindowsApps 别名目录(L210-236),沙箱指令注入 baseInstructions(L143-145);
- **审计静默**:审计任何失败均静默降级,不影响主流程(3.6);
- **清理语义**:cleanup 是幂等操作(start 超时 / restart / 退出共用);退出链 `window-all-closed` → cleanup,`will-quit` → disposeAllTerminals,WS 断开不再触发重启(disposed 标记,ws-client.ts L111)。

## §6 相关接口与文档

- IPC:channels.ts `APP_SERVER_STATUS` L71、`APP_SERVER_STATUS_GET` L72、`AppServerStatusPayload` L232(类型单源);
- 关联文档:01(事件流消费)、02(审批链路 onServerRequest)、03(文件快照 fileSnapshotOnToolCall)、05(MCP 启动状态订阅)、06(退出链 disposeAllTerminals);
- 测试:src/stores/__tests__/appServerStore.spec.ts(渲染侧状态机)。

## §7 关键文件索引

| 文件 | 职责 |
| --- | --- |
| electron/codex/app-server-manager.ts | 子进程生命周期、重连、缓存失效、数据路径 |
| electron/codex/ws-client.ts | WebSocket JSON-RPC 传输 |
| electron/codex/thread-cache.ts | 线程两级缓存 |
| electron/codex/audit.ts | 审计日志 |
| electron/codex/event-converter.ts | 通知归一化 |
| electron/codex/list-cache.ts | thread-list / model-list 列表缓存(stale-while-revalidate) |
| src/stores/appServerStore.ts | 渲染侧连接状态 |
| electron/main.ts | 生命周期接线与退出链 |
