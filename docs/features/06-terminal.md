# 06 终端管理

> 说明:本文基于当前工作区源码(main 分支),行号均为编写时核对的真实位置;代码重构后需重新核对。

职责:通过 node-pty 提供真实 PTY 终端;主进程持有会话(含滚动缓冲),渲染侧 xterm.js 负责渲染与交互;终端创建时对工作目录做白名单/写隔离解析,保证终端不落在白名单外目录。

## 1. 功能概述

- **PTY 会话**:`terminal-manager.ts` 用 `node-pty` 启动默认 shell(PW7 → PS5 → ComSpec,非 Windows 用 `$SHELL`/bash),会话按 `terminal-<seq>` 编号登记在内存 Map;每个会话维护 1MB 上限的滚动缓冲(`TERMINAL_BUFFER_LIMIT`),供渲染侧 reattach 回放。
- **IPC 面**:`handlers/terminal.ts` 提供创建/写入/调整尺寸/关闭/挂载 5 个入口;创建时若 cwd 不在白名单,走写隔离解析(ensureWorktree),并把隔离结果以 `workspaceNote` 回传;终端输出与退出以 `TERMINAL_DATA` / `TERMINAL_EXIT` 推送到渲染侧。
- **渲染侧**:`IntegratedTerminal.vue`(xterm + FitAddon)与面板 store 的「线程 ↔ 终端」映射配合;终端跟随活动线程切换,输出分块回放防卡顿,选区复制、自适应尺寸、关闭/重开均由组件自理。

## 2. 架构与数据流

```text
IntegratedTerminal.vue(xterm.js)
  ├─ createTerminal({cwd, cols, rows}) ─┐
  ├─ writeTerminal(id, data)  ← onData │
  ├─ resizeTerminal(id, cols, rows)    │ IPC(preload)
  ├─ killTerminal(id) / attachTerminal │
  └─ onTerminalData / onTerminalExit ──┴─ 推送到渲染
        │
        ▼
handlers/terminal.ts registerTerminalHandlers
  ├─ TERMINAL_CREATE: isPathAllowed(cwd)? 否 →
  │    stripWtSuffix / lookupProjectRoot → ensureWorktree(term-*) →
  │    成功: cwd=worktree, note=「已切换至写隔离工作区」
  │    tooLong: cwd=undefined, note=「…路径过长…默认目录」
  │    失败: allowRoot(projectRoot), cwd=projectRoot, note=「…使用项目目录」
  ├─ createTerminal(cwd, cols, rows, onData→TERMINAL_DATA, onExit→TERMINAL_EXIT)
  └─ write / resize / kill / attach → terminal-manager.ts
        │
        ▼
terminal-manager.ts(Map<terminalId, TerminalSession>)
  pty.spawn(defaultShell(), [], { cols, rows, cwd, env, name:'xterm-256color' })
  onData: buffer += data; trimBuffer(1MB 半切换行对齐); 转发 onData
  onExit: 移除会话; 转发 onExit(exitCode)
```

## 3. 实现分解

| 模块 | 行号 | 函数/常量 | 说明 |
|---|---|---|---|
| codex/terminal-manager.ts | L7 | TERMINAL_BUFFER_LIMIT = 1MB | 滚动缓冲上限 |
| | L9-16 | TerminalSession / terminals / terminalSeq | `{ proc: IPty, cwd, buffer }`;内存 Map 与自增序号 |
| | L18-26 | defaultShell | win32:PW7(`C:\Program Files\PowerShell\7\pwsh.exe`)→ PS5 → `ComSpec`;其他:`$SHELL` \|\| bash |
| | L28-37 | resolveCwd | cwd 存在且为目录 → resolve;否则回退 homedir |
| | L39-69 | createTerminal | spawn PTY(env 注入 `TERM=xterm-256color`);onData 累缓冲 + trim + 回调;onExit 删会话 + 回调;返回 terminalId |
| | L71-76 | trimBuffer | 超限后取后半段起始换行处切(避免切碎行) |
| | L78-82 | attachTerminal | 返回 `{ exists, cwd?, buffer? }`(渲染侧回放用) |
| | L84-96 | writeTerminal / resizeTerminal | proc.write / proc.resize;无会话返回 false |
| | L98-104 | killTerminal | proc.kill + 删除登记 |
| | L106-115 | disposeAllTerminals | 逐个 kill + 清空(main.ts:419 退出时调用) |
| handlers/terminal.ts | L11-13 | randomTerminalKey | `term-<base36 时间戳>-<随机 6 位>`(worktree key) |
| | L16-49 | TERMINAL_CREATE | 见下方关键流程 |
| | L51-58 | TERMINAL_WRITE | 转发 writeTerminal |
| | L60-67 | TERMINAL_RESIZE | 转发 resizeTerminal |
| | L69-76 | TERMINAL_KILL | 转发 killTerminal |
| | L78-82 | TERMINAL_ATTACH | 转发 attachTerminal(回传缓冲) |
| 渲染 IntegratedTerminal.vue | L29-56 | REPLAY_CHUNK_SIZE=64KB / replayBuffer | 缓冲回放:分块 + setTimeout 步进,防 1MB 一次性写入卡顿;disposed 立即终止 |
| | L58-68 | resolveTerminalCwd | 优先级:线程 cwd → 线程项目 localPath → 选中项目 localPath → fileStore.currentDir → 首个项目 |
| | L70-82 | scheduleResize | 100ms 防抖:fitAddon.fit + 同步 resizeTerminal |
| | L84-128 | spawnTerminal | 优先 attach 已存在会话(成功则 reset + 回放缓冲,无缓冲时打印 cwd 提示);否则 createTerminal 并按返回值打印实际 cwd/隔离提示;异常打印失败文案 |
| | L130-146 | closeTerminal / reopenTerminal | kill + 清除映射 + 置 closed;重开重新 spawn |
| | L148-212 | onMounted | xterm 实例(主题/字体/5000 行回滚)、FitAddon、选区自动复制、Ctrl/Cmd+C 带选区时复制并拦截、onData→writeTerminal、订阅 onTerminalData/onTerminalExit(退出打印 exitCode 并清映射)、初始 spawn、ResizeObserver+window.resize |
| | L214-219 | watch(activeThreadId) | 切换活动线程自动重新 spawnTerminal |
| | L221-234 | onUnmounted | 退订事件、断开 observer、dispose term |
| 渲染 panelStore/terminal.ts | L8-14 | getTerminalIdForThread / setTerminalIdForThread | 线程 ↔ terminalId 映射 |
| | L16-30 | removeTerminalForThread / removeTerminalByTerminalId | 按线程/按终端 id 移除映射 |

### 3.1 关键流程:TERMINAL_CREATE 的写隔离解析(terminal.ts:16)

1. cwd 未提供或已在白名单 → 直接创建。
2. 不在白名单:先 `stripWtSuffix(cwd)`(若已是 worktree 内路径,恢复项目根),再 `lookupProjectRoot(cwd)` 找所属项目根;两者都失败 → 「路径不在允许范围内」(`PERMISSION_DENIED`)。
3. `ensureWorktree(randomTerminalKey(), projectRoot)`:成功 → cwd 指向隔离工作区(「已切换至写隔离工作区」);`tooLong` → 使用默认目录;其他失败 → `allowRoot(projectRoot)` 临时放行并直接用项目根(「工作区隔离不可用,终端使用项目目录」)。
4. `createTerminal(cwd, cols ?? 80, rows ?? 24, onData, onExit)`,返回 `{ terminalId, cwd, workspaceNote }`。

## 4. 状态与数据模型

- `TerminalCreateOptions = { cwd?: string; cols?: number; rows?: number }`;`TerminalCreateResult = { terminalId: string; cwd?: string; workspaceNote?: string }`;`TerminalAttachResult = { exists: boolean; cwd?: string; buffer?: string }`(channels.ts,env.d.ts:168)。
- 会话状态仅存主进程(进程 + 1MB 缓冲);渲染侧只持线程→terminalId 映射(不存输出)。
- 终端退出以 `TERMINAL_EXIT { terminalId, exitCode }` 推送,渲染侧据此清理映射与提示。

## 5. 边界与异常

- 白名单:创建时目录校验;越权路径明确 `PERMISSION_DENIED`;写隔离失败(非 tooLong)会 `allowRoot` 临时放行并注明「隔离不可用」。
- 会话不存在:write/resize/kill/attach 一律返回 false / `{ exists: false }`,渲染侧回退重新创建。
- 缓冲上限 1MB:trim 在半段处按换行对齐裁剪,保证回放不以半行开始。
- 尺寸变化:ResizeObserver + 100ms 防抖,fit 失败静默忽略(面板未布局)。
- 回放:64KB 分块 + 事件循环让步,组件卸载/会话切换即中止,不重复写入。
- 退出清理:main.ts:419 应用退出时 disposeAllTerminals,逐会话 kill 且忽略单点失败。
- 渲染侧所有 IPC 调用均 `.catch(() => {})` 降级,不因 IPC 失败阻塞 UI。

## 6. 相关接口与文档

- IPC 通道:`TERMINAL_CREATE/WRITE/RESIZE/KILL/ATTACH`(渲染→主)、`TERMINAL_DATA/TERMINAL_EXIT`(主→渲染),常量见 `electron/ipc/channels.ts` L111-117。
- 关联文档:03 §3.2/§3.4(ensureWorktree、allowRoot、lookupProjectRoot、stripWtSuffix 语义)、07(main.ts 退出清理链)。

## 7. 关键文件索引

- `electron/codex/terminal-manager.ts`(115 行)
- `electron/ipc/handlers/terminal.ts`(83 行)
- `src/components/IntegratedTerminal.vue`(321 行)
- `src/stores/panelStore/terminal.ts`(38 行)
