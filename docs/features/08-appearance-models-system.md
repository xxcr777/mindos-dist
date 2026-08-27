# 08 外观 · 模型 · 系统集成实现原理

渲染层设置中心(主题/壁纸/模型连接/自动化开关)与主进程系统集成(托盘、单实例、窗口快照退出链、app:// 协议、安全加固)的完整实现说明。

> 行号基于当前工作区 main 分支,若与本地不同,以 `rg -n` 实际结果为准。

## §1 功能概述

- **设置持久化**:全部用户设置序列化到渲染层 `localStorage` 的 `mindos-settings`(`settingsStore.ts:10`),由 `load()`(L309)解析恢复、`save()`(L358)落盘并立即 `apply()`。
- **外观**:27 个 Codex 主题(`CODEX_THEMES` L37)、字体/字号/强调色、壁纸(mask/blur/亮度自适应压暗)、主题导入导出(`WALLPAPER_*` IPC 通道,channels.ts L101-107)。
- **模型管理**:内置模型目录 + 后端 `models:list` 合并;连接凭证(apiKey/apiBase)本地存储;`model:verify` 校验;默认模型与推理强度。
- **自动化与 Hook**:自动测试/审查/分支同步等开关;三类 Git Hook(beforeCommit / afterSave / onReview)的种子与用户数据读写。
- **系统集成**:单实例锁、系统托盘、窗口控制、退出快照链、`app://` 私有协议、webContents 安全加固、`codex:check` 环境探测。

## §2 架构与数据流

```text
┌────────────────────────── 渲染进程 ──────────────────────────┐
│  App.vue / 设置面板                                          │
│    └── useSettingsStore (src/stores/settingsStore.ts)        │
│           │  load() / save()  → localStorage 'mindos-settings'│
│           │  apply()          → <html> class + CSS 变量       │
│           │  fetchAvailableModels() / connectModel()          │
│           └── window.mindos.*  (preload 桥)                   │
└──────────────────────────────┬───────────────────────────────┘
                               │ ipcRenderer.invoke
┌──────────────────────────────▼───────────────────────────────┐
│  preload.ts:setLogLevel L75 / listModels L101 / readHooks     │
│  L108 / writeHooks L109 / getWallpaperConfig L243 / exportTheme│
│  L246                                                         │
└──────────────────────────────┬───────────────────────────────┘
                               │ ipcMain.handle
┌──────────────────────────────▼───────────────────────────────┐
│  handlers/models.ts  MODELS_LIST L39 / MODEL_VERIFY L12       │
│  handlers/hooks.ts   HOOKS_READ L88 / HOOKS_WRITE L104        │
│  handlers/system.ts  WINDOW_* L33-47 / LOG_SET_LEVEL L77      │
│                       CODEX_CHECK L49                         │
│  └── appServer.modelList() / appServer.getCodexPath()         │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│  main.ts 系统集成:单实例锁 L374 / 托盘 L147 / 窗口 L159 /      │
│  退出快照链 L211-226、L404-414 / app:// 协议 L109 / 加固 L98   │
│  通知 L35-48 / 启动编排 L385-393                              │
└──────────────────────────────────────────────────────────────┘
```

## §3 实现分解

### 3.1 设置状态与持久化

| 文件:行号 | 函数/定义 | 说明 |
|---|---|---|
| `settingsStore.ts:10` | `STORAGE_KEY` | `'mindos-settings'`,localStorage 键 |
| `settingsStore.ts:88` | `parseStoredNumber` | 兼容 number/数字字符串,非法回退 |
| `settingsStore.ts:97` | `parseStoredBool` | 仅接受布尔,否则回退 |
| `settingsStore.ts:102` | `parseStoredString` | 非空字符串才采纳 |
| `settingsStore.ts:114` | `isStoredConnected` | 连接模型存储条目类型守卫 |
| `settingsStore.ts:125` | `parseConnectedModels` | 数组过滤 + provider 推导(`deriveProvider` L76) |
| `settingsStore.ts:309` | `load()` | 解析 JSON;任何异常静默忽略(保留默认值),末尾 `apply()` + `applyLogLevel()` |
| `settingsStore.ts:358` | `save()` | 组装字段表写回 localStorage,`apply()` + `syncWallpaperToBackend()` |
| `settingsStore.ts:554` | `apply()` | 主题/字体/强调色/壁纸落地到 DOM |
| `settingsStore.ts:608` | 模块初始化 | 构造时即 `load()` + `loadWallpaperFromBackend()` + `loadHooks()` |

状态字段完整清单见 §4。所有变更路径统一走 `ref → save()`,不存在分散写入。

### 3.2 外观:主题/字体/字号/强调色

| 文件:行号 | 说明 |
|---|---|
| `settingsStore.ts:150-161` | 外观状态:`theme`(默认 `'dark'`)、`fontSize`(14)、`fontFamily`、`accentColor`(`#89b4fa`)、`customThemeVars` |
| `settingsStore.ts:288` | `setTheme()`:写状态 → save → apply |
| `settingsStore.ts:503` | `ALL_THEME_CLASSES`:`theme-*` 类名全集 |
| `settingsStore.ts:554-578` | `apply()`:字号 → CSS 变量 `--font-size-*`;强调色 → `--accent`;字族 → `--font-sans-applied` + `body`;先移除全部 `theme-*` 类,非 dark 时再挂 `theme-<value>`;随后覆盖 `customThemeVars` |
| `settingsStore.ts:604` | `applyLive()`:预览用,立即重放 apply |

`dark` 为默认主题且不带类名;其余 26 个主题(含 4 个 Catppuccin 变体、Coldark、Gruvbox、Solarized 等)通过类名激活,具体配色由全局 CSS 定义。

### 3.3 壁纸与主题导入导出

| 文件:行号 | 说明 |
|---|---|
| `settingsStore.ts:139-140` | `WALLPAPER_BRIGHT_THRESHOLD`(0.55)、`WALLPAPER_AUTO_DIM_MAX`(0.22) |
| `settingsStore.ts:390` | `loadWallpaperFromBackend()`:启动时经 `window.mindos.getWallpaperConfig()` 拉取主进程持久化的壁纸配置(路径等,localStorage 存不了图片) |
| `settingsStore.ts:404` | `syncWallpaperToBackend()`:每次 save 时把壁纸四要素同步到主进程 |
| `settingsStore.ts:417` | `importMindosTheme()`:JSON 解析 → 写 `customThemeVars`/主题/壁纸开关/图片(base64 经 `saveWallpaperBase64` 落盘)/mask/blur,容错回退 |
| `settingsStore.ts:464` | `exportMindosTheme()`:从 `getComputedStyle` 采集 17 个 CSS 变量 + 壁纸 base64(`readWallpaperBase64`)组装 JSON,经 `exportTheme` 保存 |
| `settingsStore.ts:508` | `updateAutoDim()`:解码壁纸 → 64px 缩略图 → 亮度加权平均(Luminance Rec.709 系数 L540)→ 超过 0.55 时叠加压暗 `--wallpaper-auto-dim`,结果按路径缓存 |
| `settingsStore.ts:580-601` | `apply()` 壁纸分支:启用时 `#wallpaper-layer` 以 `app://file?path=` 加载图片、设置 `--wallpaper-mask/blur/auto-dim`;否则清除全部壁纸样式 |

IPC 通道:`WALLPAPER_PICK_IMAGE` L101、`GET_CONFIG` L102、`SET_CONFIG` L103、`EXPORT_THEME` L104、`IMPORT_THEME` L105、`READ_BASE64` L106、`SAVE_BASE64` L107(channels.ts)。

### 3.4 模型管理

| 文件:行号 | 说明 |
|---|---|
| `settingsStore.ts:12-19` | `AVAILABLE_MODELS`:6 个内置模型(qwen3.8 / kimi-k3 / glm5.2 / deepseek 三款),前端保底目录 |
| `settingsStore.ts:213` | `availableModels`:运行时模型目录(初始拷贝内置) |
| `settingsStore.ts:217` | `fetchAvailableModels()`:调 `window.mindos.listModels()` → 后端 `{id, displayName}` 按 id 合并:同名覆盖 label、后端独有追加(provider 由 `deriveProvider` 推断 L237)、后端失败时静默保留现状 |
| `settingsStore.ts:67-72` | `ConnectedModel`:`modelId / modelProvider / apiKey / apiBase` |
| `settingsStore.ts:165` | `connectedModels`:已连接模型(含 apiKey,随 localStorage 持久化) |
| `settingsStore.ts:205` | `connectedModelOptions`:computed 的 `ModelOption[]`,label 用 `modelLabel()` L82 解析 |
| `settingsStore.ts:256` | `connectModel()`:按 modelId 覆盖或追加,`save()` |
| `settingsStore.ts:267` | `disconnectModel()`:过滤移除,`save()` |
| `settingsStore.ts:248/252` | `isModelConnected()` / `getModelApiKey()`:供聊天链路查询 |
| `handlers/models.ts:6-9` | `DEFAULT_BASE_URLS`:deepseek → `api.deepseek.com/v1`,qwen → 阿里云 compatible-mode |
| `handlers/models.ts:12-37` | `MODEL_VERIFY`:拼 `${base}/chat/completions`,带 `Authorization: Bearer`,`max_tokens: 1` 的 `ping` 请求,2xx 即通过;网络异常返回 false |
| `handlers/models.ts:39-41` | `MODELS_LIST`:直通 `appServer.modelList()`(服务端缓存逻辑见 07 文档) |
| `chatStore.ts:297` | 首次进入用 `connectedModels[0].modelId` 初始化 `currentModel` |
| `chatStore.ts:389` | 恢复会话时校验快照模型仍已连接,否则回退 |
| `chatStore.ts:536-538` | 默认模型解析:优先 `defaultModelId`,无则取 `connectedModels[0]` |
| `chatStore.ts:542` | 当前模型 provider 查询 |

安全边界:apiKey 仅存渲染层 localStorage(明文、非隔离存储),主进程不落盘密钥;`MODEL_VERIFY` 由主进程发起,密钥不进渲染网络层之外的任何日志。

### 3.5 自动化开关与 Git Hook

| 文件:行号 | 说明 |
|---|---|
| `settingsStore.ts:175-179` | 自动化开关:`autoTestOnAgentComplete` / `autoReviewOnCommit` / `autoUpdateContextOnBranch` / `autoDetectNpmInstall` / `autoSystemTrayNotify` |
| `settingsStore.ts:171-173` | 默认执行模式(`build`/`plan`)、默认模型、默认推理强度(`medium`/`high`/`max`,`REASONING_LEVELS` L21) |
| `settingsStore.ts:181` | `hooks`:`HookItem[]`,仅存渲染内存,持久化走主进程 `hooks.json` |
| `settingsStore.ts:183/191/200` | `loadHooks()` / `saveHooks()` / `getHookContent()`(按事件取启用项内容,未启用返回 null) |
| `handlers/hooks.ts:11-60` | `HOOK_SEED`:beforeCommit / afterSave / onReview 三条内置模板,均默认禁用 |
| `handlers/hooks.ts:63` | `isValidHookId`:`^[a-z0-9][a-z0-9-]{0,63}$`,防路径穿越 |
| `handlers/hooks.ts:68` | `sanitizeHooks`:白名单化字段(id/event 校验、name≤120、description≤500、content 字符串化) |
| `handlers/hooks.ts:88-102` | `HOOKS_READ`:读 `userData/hooks.json`,不存在或解析失败回退种子 |
| `handlers/hooks.ts:104-114` | `HOOKS_WRITE`:同步写回 sanitize 后的 JSON |

事件类型 `HookEvent`:`'beforeCommit' | 'afterSave' | 'onReview'`(channels.ts:254),`HookItem` 定义在 channels.ts:256。

### 3.6 日志级别与渲染日志

| 文件:行号 | 说明 |
|---|---|
| `settingsStore.ts:27-33` | `LOG_LEVELS`:error → verbose 五档 |
| `settingsStore.ts:274` | `applyLogLevel()`:加载/启动时同步主进程 |
| `settingsStore.ts:282` | `setLogLevel()`:保存 + 同步 |
| `handlers/system.ts:77-84` | `LOG_SET_LEVEL`:校验后调 `setLogLevel()`(log.js),非法值返回 `fail` |
| `handlers/system.ts:67-75` | `LOG_WRITE`:渲染日志转发到主进程 logger,消息截断 2000 字符 |
| `handlers/system.ts:63` | `PROBE_LOG`:渲染侧探针行写入 probe 管线 |

### 3.7 系统集成(主进程)

| 文件:行号 | 说明 |
|---|---|
| `main.ts:26-28` | 图标路径:win32 用 `.ico`,其余 `.png` |
| `main.ts:50-52` | `app` scheme 注册为 privileged(standard/secure/fetch/stream) |
| `main.ts:54-65` | `MIME_BY_EXT` + `mimeFromPath`:app:// 响应 MIME 映射 |
| `main.ts:67-96` | `fileUrlToPath` / `isAllowedFileUrl` / `isAllowedNavigationUrl`:导航白名单(dev server / app:// / allowlist 内 file://) |
| `main.ts:98-107` | `hardenWebContents`:全局拒绝 window.open、拦截非法 will-navigate |
| `main.ts:109-135` | `registerAppProtocol`:`app://artifact` 走 artifact handler;`app://file?path=` 需过 `isPathAllowed` + fs.access,再流式返回 |
| `main.ts:137-145` | `showMainWindow()`:重建/还原/显示/聚焦 |
| `main.ts:147-157` | `createTray()`:托盘图标 + 菜单(显示主窗口 / 退出),点击左键显示窗口 |
| `main.ts:159-178` | `createWindow()`:1200×800 无边框(`frame:false`)、`contextIsolation:true`、`nodeIntegration:false`、`webviewTag:true` |
| `main.ts:180-186` | `ready-to-show` 显示;5s 兜底强制显示,防白屏隐身 |
| `main.ts:190-202` | `will-attach-webview`:仅允许白名单 URL,剥离 preload、强制 sandbox |
| `main.ts:211-226` | 窗口 `close`:先 `preventDefault` + 发 `APP_BEFORE_QUIT_SNAPSHOT` 让渲染进程存会话快照,3s 未完成则 `destroy()` |
| `main.ts:374-383` | 单实例:`requestSingleInstanceLock()` 失败即退;`second-instance` 还原并聚焦 |
| `main.ts:385-393` | `whenReady`:win32 设 `AppUserModelId`、注册协议、加固、清旧日志、建窗口、建托盘、`bootstrapAppServer()` |
| `main.ts:395-402` | `window-all-closed`:dispose eventForwarder + `appServerManager.cleanup()`;非 darwin 直接 quit |
| `main.ts:404-414` | `before-quit`:防重入(`quitting` 标志),窗口存活时先发快照再 3s 后 quit |
| `main.ts:416-421` | `will-quit`:`computerUseSession.revoke()`、`disposeAllTerminals()`、`flushProbeSync()` |
| `main.ts:35-48` | 通知:`makeTurnNotifier` / `makeTokenUsageNotifier`,窗口聚焦时抑制(详情见 07 文档) |

窗口控制 IPC:`WINDOW_MINIMIZE` L2 / `WINDOW_MAXIMIZE` L3 / `WINDOW_CLOSE` L4(channels.ts),handler 在 `handlers/system.ts:33-47`。

### 3.8 codex 环境探测

| 文件:行号 | 说明 |
|---|---|
| `handlers/system.ts:49-61` | `CODEX_CHECK`:先取 `appServer.getCodexPath()`,未就绪则 `waitForReady(8000)`,再回退 `findCodexPath()`;返回 `{installed, path, version}` |
| `handlers/system.ts:13-30` | `getCodexVersion()`:`codex --version` 子进程探测,60s TTL 缓存,失败置 null |

## §4 状态与数据模型

### localStorage `mindos-settings` 字段(settingsStore.ts:359-384)

| 字段 | 默认值 | 说明 |
|---|---|---|
| `language` | `'zh-CN'` | i18n |
| `defaultWorkDir` | `''` | 新建项目默认目录 |
| `openLastProject` | `true` | 启动恢复上次项目 |
| `autoSaveInterval` | `30` | 秒 |
| `autoCompactTokenLimit` | `DEFAULT_AUTO_COMPACT_TOKEN_LIMIT`(channels 导入,settingsStore.ts:4) | 自动压缩阈值 |
| `logLevel` | `'info'` | 主进程日志级别 |
| `theme` / `fontSize` / `fontFamily` / `accentColor` | `'dark'` / `14` / `'system'` / `'#89b4fa'` | 外观 |
| `customThemeVars` | `{}` | 导入主题的 CSS 变量覆盖 |
| `apiBase` | `''` | 模型 API base 全局覆盖 |
| `connectedModels` | `[]` | `ConnectedModel[]`,含 apiKey |
| `mindosUrl` / `mindosPassword` / `autoReconnect` | `'http://120.27.192.154:8001'` / `''` / `true` | MindOS 服务端 |
| `defaultMode` / `defaultModelId` / `defaultReasoningEffort` | `'build'` / `''` / `'medium'` | 聊天默认 |
| `autoTestOnAgentComplete` 等 5 个 `auto*` | `false` | 自动化开关 |
| `hooks` | — | Hook 项仅内存持有,持久化在 `userData/hooks.json` |

### 类型定义

- `ConnectedModel`(settingsStore.ts:67)、`ModelOption`(L74)
- `ModelVerifyOptions`(channels.ts:237)、`HookEvent`(L254)、`HookItem`(L256)
- `LogLevelValue`(settingsStore.ts:35)

### 主题生效机制

`<html>` 上挂 `theme-<value>` 类(除默认 dark),外加 CSS 变量:字号三档、`--accent`、`--font-sans-applied`、壁纸 `--wallpaper-mask/blur/auto-dim`;`customThemeVars` 在类之后覆盖,优先级最高。

## §5 边界与异常

- **localStorage 损坏**:`load()` catch 静默保留默认值(settingsStore.ts:351-353)。
- **后端 models:list 失败/为空**:保留现有 `availableModels` 不覆盖(settingsStore.ts:217-246)。
- **密钥安全**:apiKey 明文存渲染层 localStorage,主进程不落盘;`MODEL_VERIFY` 请求由主进程发出。
- **MODEL_VERIFY 网络失败**:返回 `false` 不抛出(handlers/models.ts:34-36)。
- **壁纸采样失败**:auto-dim 归零,不阻断 UI(settingsStore.ts:548-551)。
- **主题导入异常**:返回 `{success:false, error}` 文案,渲染层展示,不崩溃(settingsStore.ts:459)。
- **hooks.json 损坏/缺失**:回退 `HOOK_SEED`(handlers/hooks.ts:90-101);写失败返回 false。
- **快照退出**:close/before-quit 均 3s 兜底强制 `destroy()`/`quit()`(main.ts:220-225、L412),渲染进程卡死不阻塞退出;快照请求由 `snapshotRequested` 标志防重入(L213、L410)。
- **窗口白屏**:`ready-to-show` 5s 兜底显示(main.ts:182-186)。
- **webview 安全**:非白名单 URL 一律 `preventDefault`,webview 强制去 preload + sandbox(main.ts:190-202)。
- **平台差异**:darwin 下关窗不退出应用(main.ts:399-401),`activate` 重建窗口(L423)。
- **托盘重复创建**:`createTray()` 仅在 `whenReady` 调用一次,`tray` 全局持有防 GC。

## §6 相关接口与文档

| 通道(channels.ts) | 行号 | 消费方 |
|---|---|---|
| `WINDOW_MINIMIZE` / `WINDOW_MAXIMIZE` / `WINDOW_CLOSE` | 2/3/4 | system.ts:33-47 |
| `CODEX_CHECK` | 19 | system.ts:49 |
| `PROBE_LOG` / `LOG_WRITE` / `LOG_SET_LEVEL` | 23/24/25 | system.ts:63/67/77 |
| `MODEL_VERIFY` / `MODELS_LIST` | 43/44 | models.ts:12/39 |
| `HOOKS_READ` / `HOOKS_WRITE` | 49/50 | hooks.ts:88/104 |
| `APP_BEFORE_QUIT_SNAPSHOT` | 99 | main.ts:219/411 |
| `WALLPAPER_PICK_IMAGE` ~ `WALLPAPER_SAVE_BASE64` | 101-107 | settingsStore 壁纸链路 |

- `handlers/system.ts:86` `APP_SERVER_STATUS_GET` 与 07 文档的 `AppServerStatusPayload` 联动。
- 模型服务端列表(`appServer.modelList`)与 30s 握手细节见 `07-lifecycle.md`。
- 通知判定 `isWindowFocused` 与任务完成链路见 `07-lifecycle.md` §3.8。
- 导航白名单依赖 03 文档的 fs allowlist(`isPathAllowed`)。

## §7 关键文件索引

| 文件 | 作用 |
|---|---|
| `src/stores/settingsStore.ts`(663 行) | 设置中心唯一 store:状态/持久化/外观落地/模型连接/壁纸/主题导入导出 |
| `src/stores/chat/chatStore.ts` | 模型选择消费方(默认模型回退 L536-538、会话恢复 L389) |
| `electron/main.ts`(428 行) | 窗口/托盘/单实例/协议/安全加固/退出快照链 |
| `electron/ipc/handlers/models.ts`(42 行) | 模型验证与列表 |
| `electron/ipc/handlers/hooks.ts`(115 行) | Git Hook 读写与消毒 |
| `electron/ipc/handlers/system.ts`(99 行) | 窗口控制/日志/环境探测 |
| `electron/preload.ts` | 渲染桥(`window.mindos`) |
| `electron/ipc/channels.ts` | IPC 常量与类型真源 |
