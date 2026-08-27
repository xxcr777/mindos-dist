# 03 文件系统与写隔离

> 说明:本文基于当前工作区源码(main 分支),行号均为编写时核对的真实位置;代码重构后需重新核对。

职责:管理渲染进程对本地文件系统的全部读写能力,并通过「白名单 + 写隔离工作区 + 写前快照」三层机制,把 agent 对项目文件的修改限制在可控、可回滚的范围内。

## 1. 功能概述

文件系统能力由 `electron/ipc/handlers/fs.ts` 的 IPC handler 提供,所有路径先过 `isPathAllowed` 白名单校验,渲染进程无任何绕过路径。

写隔离:线程/终端启动时(见 `handlers/threads.ts`、`handlers/terminal.ts`),主进程在项目仓库中创建 **detached worktree**(`git worktree add --detach`),agent 的写入全部落在 worktree 目录,与用户主分支工作区隔离;通过 manifest 登记 + pre-push hook 兜底,防止 agent 把代码推到远端。`$CODEX_HOME/worktrees` 为 worktree 根目录。

写前快照:agent 执行写文件类工具调用前,主进程对目标文件(≤2MB)做内存快照,渲染侧可随时通过 `FS_READ_SNAPSHOT` 取回(未命中不报错,只返回空)。

变更通知:主进程递归 watch 已登记目录,500ms 防抖 + 500 条缓冲上限,`FS_CHANGED` 推送渲染侧 `panelStore/fileChanges`。

## 2. 架构与数据流

```text
渲染层
  fileStore / panelStore/fileChanges.ts ──┐
                                          │ FS_READ_*/FS_WRITE_*/FS_CHANGED(推)
主进程 handlers/fs.ts ────────────────────┤
  │ 全部入口: isPathAllowed(allowlist.ts) 守卫
  ├─ 目录枚举/大文件截断/PS 读取/写/删(回收站)/改名/打开
  └─ ensureDirWatcher: recursive watch → 500ms 防抖 → FS_CHANGED

写隔离
threads.ts / terminal.ts ── ensureWorktree(worktree.ts) ── git worktree add --detach
                                 │  $CODEX_HOME/worktrees/<proj>-<key>
                                 └─ manifest(userData/worktrees-manifest.json) + allowRoot + pre-push hook

写前快照
ws-client(agent tool_call) ── fileSnapshotOnToolCall(snapshot.ts) ── Map<path, snapshot>
                                                                    └─ FS_READ_SNAPSHOT 查询(仅内存)
```

## 3. 实现分解

| 模块 | 行号 | 函数/常量 | 说明 |
|---|---|---|---|
| handlers/fs.ts | L14-18 | dirWatchers / fsNotifyDebounce / fsChangeBuffer | watch 句柄、防抖定时器、变更缓冲(每目录一个 Map) |
| | L17-21 | FS_NOTIFY_DEBOUNCE_MS=500 / FS_CHANGE_BUFFER_LIMIT=500 / MAX_PREVIEW_BYTES=10MB / MAX_WRITE_BYTES=10MB / MAX_TREE_DEPTH=8 | 关键阈值 |
| | L23-24 | PS7_PATH / PS5_PATH | PowerShell 读取通道:优先 PS7,回退 PS5.1 |
| | L26-27 | PS_READ_MAX_LINES=20000 / FS_WATCH_IGNORE_RE | PS 读取行数上限;watch 噪音正则(node_modules/.git/dist 等) |
| | L28-48 | FS_TREE_SKIP_DIRS | 目录树枚举跳过 19 个目录(node_modules/.git/.venv/target 等) |
| | L55-70 | classifyFsEvent | change→`update`;stat 存在→`add`;stat 失败→`delete`;噪音路径返回 null |
| | L72-108 | ensureDirWatcher | 每目录单 watcher;`watch(dir,{recursive:true})`;变更入缓冲,超限丢最旧;500ms 防抖后 send `IPC.FS_CHANGED`;error 时关闭并移除 |
| | L110-122 | cleanupAllWatchers | 窗口关闭时清理全部 watcher/定时器/缓冲 |
| | L133-166 | readDirEntries | 深度受限递归枚举,目录优先 + 名称排序,不可访问项静默跳过 |
| | L168-182 | resolvePowerShellExecutable | PS7 → PS5 → PATH 兜底 |
| | L184-226 | readFileViaPowerShell | `Get-Content -TotalCount 20001`,UTF-8 输出,单引号转义,超 20000 行截断标记 |
| | L231-238 | FS_READ_DIR | 白名单 → 枚举(默认深度 2)→ 启动 watcher |
| | L240-262 | FS_READ_FILE | >10MB 只读前 10MB(truncated:true);否则全量 utf-8 |
| | L264-276 | FS_READ_FILE_VIA_POWERSHELL | 与 agent 侧 Get-Content 视角对齐,失败映射 NOT_FOUND |
| | L278-283 | FS_READ_SNAPSHOT | 仅查内存写前快照,未命中返回 ok()(无错误) |
| | L285-301 | FS_WRITE_FILE | 内容必须 string、≤10MB,utf-8 写入 |
| | L303-313 | FS_DELETE | `shell.trashItem` → 进系统回收站,非永久删除 |
| | L315-325 | FS_RENAME | 新旧路径双端白名单校验 |
| | L327-337 / L339-349 | FS_OPEN_EXTERNAL / FS_OPEN_URL | shell.openPath / openExternal;URL 仅 http/https |
| | L351-364 / L366-388 | FILE_READ_PREVIEW / FILE_READ_PREVIEW_TEXT | base64 全量 / 前 600KB 文本截断 |
| fs/allowlist.ts | L79 | isPathAllowed | 唯一路径守门:仅允许已登记项目根、worktree 根与数据目录(详见 02 §2.2) |
| codex/snapshot.ts | L51-77 | extractTargetPaths / toAbsolute | 从工具输入提取写目标路径;cwd 为 null 时返回 null(不快照) |
| | L87-107 | fileSnapshotOnToolCall | agent tool_call 回调:提取路径 → ≤2MB 读入内存缓存 |
| | L108-113 / L115 / L120 | getFileSnapshot / resetFileSnapshotsForTest / snapshotSizeForTest | 查询(供 FS_READ_SNAPSHOT)/ 测试重置 / 测试统计 |
| codex/worktree.ts | L19-23 | WORKTREES_DIR_NAME='worktrees' / WORKTREES_MANIFEST_NAME / LEGACY_WT_ROOT='.wt' / MAX_PATH=260 | 目录与 manifest 命名、遗留兼容、路径上限 |
| | L35 | stripWtSuffix | 旧 `.wt/main` 形式 cwd 兼容剥离(终端调用) |
| | L43-45 | WorktreeFailureReason | `notGit` / `gitUnavailable` / `tooLong` / `failed` |
| | L91-97 | getCodexHome / getWorktreeRoot | worktree 根默认 `~/.codex/worktrees`(CODEX_HOME 可覆盖) |
| | L112-115 | sanitizeDirName | 目录名净化:仅保留 `[A-Za-z0-9-]`,防路径穿越 |
| | L117-124 | isGitRepo | 探测 `.git` 目录 |
| | L134-139 | isGitAvailable | `git --version` 一次探测,进程内缓存 |
| | L141-163 | readManifest / writeManifest | 字段校验后读入;写入走 `.tmp` + rename 原子替换 |
| | L169-200 | ensurePrePushHook | 仅当 `hooks/pre-push` 不存在时安装 exit 1 禁推 hook(尊重用户 hook);失败不阻断 |
| | L206-256 | ensureWorktree | 见下方关键流程 |
| | L259-280 | disposeWorktree | `git worktree remove --force` → 失败 fs.rm 兜底 → manifest 移除 |
| | L292-327 | pruneWorktrees | 启动清理:activeDirs(全部线程 cwd)命中的保留;逐仓库 `git worktree prune`;遗留 `.wt/main` 迁移清理 |
| | L330-339 | removeProjectWorktrees | 项目移除时销毁名下全部 worktree + prune |
| | L342-357 | renameWorktreeKey | threadStart 先以临时 key 建树,拿到真实 threadId 后改名 |
| | L360-369 | disposeWorktreeByKey | 按 key 销毁(同 key 多记录逐一处理) |
| | L372-387 | lookupProjectRoot | 目录或其祖先属于登记 worktree → 返回项目根(最长前缀匹配);终端回退用 |
| 渲染侧 panelStore/fileChanges.ts | L9-82 | createFileChangesSlice | `fileChangeHistory: Map<threadId, FileChangeEntry[]>`;按 id 合并更新、状态推进(stream.ts 事件路由驱动,见 01 §3.3 `item/fileChange/patchUpdated`)、`getPendingFileChangeCount` 供审批角标 |

### 3.1 关键流程:ensureWorktree(worktree.ts:206)

1. `projectRoot` 非 git 仓库 → `notGit`;`git` 不可用 → `gitUnavailable`;返回 `{ dir: null, reason }`,调用方回退项目根并提示「写隔离不可用」。
2. 目录名净化:项目名 + 净化后的 key(`线程 id` / `终端临时 key`),拼到 worktree 根;长度超 260 → `tooLong`。
3. 幂等复用:目录已存在 + manifest 命中同 `dir`/`projectRoot` + `git worktree list --porcelain` 认可 → 直接复用(并确保 pre-push hook)。
4. 否则 `git worktree add --detach --force --quiet <dir> HEAD`;失败则 fs 兜底删目录 → `failed`。
5. 登记 manifest(`key/projectRoot/dir/createdAt`)→ `allowRoot(dir)` 登记白名单(可关闭)→ 安装 pre-push hook → 返回 `{ dir }`。

## 4. 状态与数据模型

- `FsChangeInfo = { path: string; kind: 'add' | 'update' | 'delete' }`(channels.ts 类型)。
- `WorktreeManifestEntry = { key: string; projectRoot: string; dir: string; createdAt: number }`,持久化于 `userData/worktrees-manifest.json`。
- `EnsureWorktreeResult = { dir: string | null; reason: WorktreeFailureReason | null }`。
- 写前快照:主进程内存 `Map<path, { content, size, mtimeMs }>`,≤2MB,进程退出即清空(不落盘)。

## 5. 边界与异常

- 所有读/写/删/改名/打开入口先过白名单,越权返回 `PERMISSION_DENIED`(错误消息统一「路径不在允许范围内」)。
- 大文件:预览/PS 读取均截断并带 `truncated` 标记;写入上限 10MB。
- 删除走回收站(`shell.trashItem` 失败才报错),不做永久删除。
- 非 git 仓库 / git 未安装 / 路径超长 → 写隔离降级为项目根直写,同时推送中文提示(见 06 §3.1 终端联动的 `workspaceNote`)。
- watch 噪音目录(node_modules/.git/dist 等)与不可访问目录静默跳过,不影响枚举与通知。
- 变更缓冲超 500 条丢最旧,防抖动合并后一次推送,避免事件风暴。
- 快照仅内存、仅 ≤2MB,且 `FS_READ_SNAPSHOT` 未命中返回成功(渲染侧视为「无快照可恢复」,不弹错误)。
- manifest 写入原子化(tmp+rename),损坏时按空 manifest 处理(不崩溃)。

## 6. 相关接口与文档

- IPC 通道(常量见 `electron/ipc/channels.ts`):`FS_READ_DIR`、`FS_READ_FILE`、`FS_READ_FILE_VIA_POWERSHELL`、`FS_READ_SNAPSHOT`、`FS_WRITE_FILE`、`FS_DELETE`、`FS_RENAME`、`FS_OPEN_EXTERNAL`、`FS_OPEN_URL`、`FILE_READ_PREVIEW`、`FILE_READ_PREVIEW_TEXT`、`FS_CHANGED`(主→渲染)。
- 关联文档:01 §3.3(事件路由与文件变更面板)、02 §2.2(白名单与双闸)、04(写隔离与 Git 的关系)、06(终端写隔离联动)、07(启动 prune 时机)。

## 7. 关键文件索引

- `electron/ipc/handlers/fs.ts`(389 行)
- `electron/fs/allowlist.ts`
- `electron/codex/worktree.ts`(387 行)
- `electron/codex/snapshot.ts`(122 行)
- `src/stores/panelStore/fileChanges.ts`(83 行)
