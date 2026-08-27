# 04 Git 集成

> 说明:本文基于当前工作区源码(main 分支),行号均为编写时核对的真实位置;代码重构后需重新核对。

职责:渲染侧 Git 面板(状态、暂存、提交、推送/拉取、分支、日志)通过 IPC 调用主进程,主进程用 `execFile`(非 shell)执行 git 命令,并对仓库路径与文件路径做白名单双闸校验,保证 git 操作只能在登记项目仓库内进行。

## 1. 功能概述

- 所有 git 命令统一走 `runGit`(`execFile('git', args, { cwd: repoPath, ... })`),参数数组直传、不做字符串拼接,天然规避 shell 注入;`maxBuffer` 64MB,默认超时 5s,push/pull 放宽到 60s,log 15s。
- 双闸校验:`GIT_DIFF`、`GIT_STAGE`、`GIT_UNSTAGE` 额外对文件路径做 `pathInRepo`(白名单 + 仓库内判定),其余入口校验 `repoPath` 白名单。
- 渲染侧 `gitStore` 驱动:仓库路径随「活动线程所属项目 → 项目本地路径」自动切换,开启自动刷新后监听 `FS_CHANGED`(500ms 防抖)刷新状态;提交/切换分支前后向自动化事件总线发事件。

## 2. 架构与数据流

```text
渲染侧 gitStore.ts
  watch(项目/线程) → repoPath
  refresh / loadLog / stage / commit / push ...
        │  window.mindos.git* (preload → IPC)
        ▼
handlers/git.ts registerGitHandlers
  ├─ isPathAllowed(repoPath) + pathInRepo(filePath) 双闸
  └─ runGit → execFile('git', args, { cwd: repoPath, timeout, maxBuffer: 64MB })
        │
        ▼
用户仓库(项目根)
  └─ 与 03 写隔离的关系:worktree 与主工作区共享同一 .git 目录,
     Git 面板操作对象是项目根工作区

自动化联动:commit → automationEventBus('git:before-commit' / 'git:commit')
              switchBranch 成功 → 'git:branch-switched'
```

## 3. 实现分解

| 模块 | 行号 | 函数/常量 | 说明 |
|---|---|---|---|
| handlers/git.ts | L21-37 | execFileAsync | execFile 回调 promise 化,err 上保留 stdout/stderr |
| | L39-54 | runGit | 统一执行入口;`windowsHide:true`;编码 utf-8;超时 5s(可传参) |
| | L56-58 | pathInRepo | 相对路径先 join repoPath 再走 isPathAllowed(绝对化白名单判定) |
| | L60-75 | statusError | 失败统一形状:`{success:false, isRepo:false, error}` |
| | L77-85 | findGitRepo | 自 dir 向上逐级找 `.git` 目录(目录监听/仓库探测用) |
| | L88-138 | GIT_DIFF | 双闸 → `diff HEAD -- <rel>`;空输出时 `ls-files --error-unmatch` 判定跟踪与否;未跟踪走 `diff --no-index /dev/null <file>`;文件不存在 → NOT_FOUND;catch 兜底同类逻辑 |
| | L140-233 | GIT_STATUS | `status --porcelain=v1 -b -uall`;解析 `##` 头(分支/上游/ahead/behind)、XY 状态码(`??`→`?`、U→冲突、R/C 取 ` -> ` 后路径、staged=`x!==' '&&x!=='?'`、引号路径 JSON.parse);`diff --cached --numstat` 统计 staged 文件数/增删行 |
| | L235-246 | GIT_STAGE | `git add -- <files>`(全路径白名单) |
| | L248-259 | GIT_UNSTAGE | `git restore --staged -- <files>` |
| | L261-272 | GIT_COMMIT | `git commit -m <message>` |
| | L274-285 | GIT_PUSH | `git push`,超时 60s |
| | L287-298 | GIT_PULL | `git pull`,超时 60s |
| | L300-322 | GIT_LOG | `log -n <count> --skip=<skip>`;格式 `%H\x1f%h\x1f%s\x1f%an\x1f%aI\x1e` 分隔解析;超时 15s |
| | L324-341 | GIT_BRANCH | `git branch`;`*` 前缀标记当前分支 |
| | L343-354 | GIT_CREATE_BRANCH | `git switch -c <name>` |
| | L356-367 | GIT_SWITCH_BRANCH | `git switch <name>` |
| 渲染侧 gitStore.ts | L8-9 | AUTO_REFRESH_KEY / LOG_PAGE_SIZE=20 | 自动刷新开关持久化键、日志分页大小 |
| | L15-39 | 状态 ref | repoPath/repoRoot/branch/upstream/ahead/behind/files/staged*/commits/branches/loading/busy/error/hasMoreLog/autoRefresh/isRepo/fileHistory 等 |
| | L51-102 | refresh | `gitStatus` 成功填充全部状态;非仓库时区分 `git.notRepoAuto`/`git.notRepo` 文案 |
| | L104-118 | loadLog | 分页加载(skip=当前长度),`hasMoreLog` 判定 |
| | L120-124 | resetLog | 清空后重载第一页 |
| | L126-146 | loadFileHistory | 单文件提交历史,`append` 分页 |
| | L148-160 | probeRepo | 仓库探测 + 结果缓存(`repoProbeCache`) |
| | L162-172 | isInRepo | 大小写归一化后判断路径是否位于 repoRoot 内 |
| | L190-208 | runAction | 统一动作包装:busy/lastActionError,成功后 refresh(+可选 resetLog) |
| | L210-242 | stage/unstage/stageAll/stageModified/stageDeleted/unstageAll | 暂存操作组合(stageAll 空时返回「无内容可暂存」文案) |
| | L244-250 | commit | 先发 `git:before-commit` → 执行 → 成功发 `git:commit` |
| | L252-258 | push / pull | resetLog 重载日志 |
| | L260-269 | createBranch / switchBranch | 切换成功发 `git:branch-switched` |
| | L271-290 | applyAutoRefresh | 开启后订阅 `onFsChanged`,仅当 `dirPath === repoPath` 且 500ms 防抖后 refresh;同时 `readDir(repoPath,1)` 拉起目录 watcher |
| | L292-325 | watch(autoRefresh) / watch(项目↔线程) | 自动刷新键持久化;repoPath 跟随「活动线程的项目 id → 项目 localPath」,切换时清缓存并 refresh + resetLog + loadBranches(immediate) |

### 3.1 关键流程:GIT_DIFF 的跟踪状态推导(git.ts:88)

1. `isPathAllowed(repoPath)` 与 `pathInRepo(repoPath, filePath)` 任一不过 → `PERMISSION_DENIED`。
2. `git diff HEAD -- <rel>`:有输出直接返回;无输出说明「已跟踪且无改动」或「未跟踪/不存在」。
3. `git ls-files --error-unmatch` 判定:失败说明未跟踪 → 检查文件是否存在;不存在 → `NOT_FOUND`;存在 → `git diff --no-index /dev/null <file>`(status 0/1 均视为成功,1 表示有差异)。
4. 任一步骤抛错时,外层 catch 重复上述「未跟踪文件」兜底,仍失败才报 `not a git repo or file not trackable`。

## 4. 状态与数据模型

- `GitFileChange = { path: string; status: '?'\|'A'\|'D'\|'M'\|'R'\|'C'\|'U'; staged: boolean }`(channels.ts)。
- `GitCommitInfo = { hash; shortHash; message; author; date }`;`GitBranchInfo = { name; current }`。
- 状态成功形状:分支信息 + `files` + staged 统计;失败统一 `statusError` 形状(见 3)。

## 5. 边界与异常

- 命令级防注入:全链路 `execFile` + 参数数组,不存在 shell 拼接路径。
- 路径级:文件级命令(stage/unstage/diff)双闸校验,越权一律 `PERMISSION_DENIED`(「路径不在允许范围内」)。
- 超时与内存:push/pull 60s、log 15s、默认 5s;64MB 输出上限。
- 未跟踪/已删除文件的 diff 有专门兜底路径;两者都失败返回明确错误而非空 diff。
- 非仓库目录:`GIT_STATUS` 返回 `isRepo:false` + 文案提示;渲染侧置空状态并展示 `notRepoAuto`(存在 repoRoot 时)或 `notRepo`。
- 自动刷新只在开启且 `dirPath` 精确等于 repoPath 时触发(防无关目录刷新)。
- gitStore 在项目切换/线程切换时主动刷新,`repoProbeCache` 避免重复探测。

## 6. 相关接口与文档

- IPC 通道:`GIT_DIFF`、`GIT_STATUS`、`GIT_STAGE`、`GIT_UNSTAGE`、`GIT_COMMIT`、`GIT_PUSH`、`GIT_PULL`、`GIT_LOG`、`GIT_BRANCH`、`GIT_CREATE_BRANCH`、`GIT_SWITCH_BRANCH`(常量见 `electron/ipc/channels.ts`)。
- 自动化事件:`git:before-commit` / `git:commit` / `git:branch-switched`(`src/utils/automation/eventBus.ts` 总线,见 01 §3.6)。
- 关联文档:03 §3(写隔离 worktree 与主仓库共享 .git;Git 面板操作项目根)、02 §2.2(白名单守卫)。

## 7. 关键文件索引

- `electron/ipc/handlers/git.ts`(368 行)
- `src/stores/gitStore.ts`(372 行)
- `src/stores/chat/gitSync.ts`(agent 变更与 git 状态同步辅助,`GIT_SYNC_MAX_FILES=500`)
