# 05 插件与 MCP 生态

> 说明:本文基于当前工作区源码(main 分支),行号均为编写时核对的真实位置;代码重构后需重新核对。

职责:管理远程 MCP 服务器插件的生命周期——插件定义的持久化、市场索引拉取与安装(含运行时环境检查、预热、审计)、`~/.codex/config.toml` 配置导出与合并、MCP 运行状态跟踪、以及连接测试。

## 1. 功能概述

- **插件 CRUD**:`userData/plugins.json` 持久化插件列表,渲染侧 `pluginStore` 负责本地状态(含内置插件模板),主进程 handler 负责读写与导出。
- **市场安装**:`market-index.ts` 提供市场索引拉取(1h 缓存、`MINDO_MARKET_INDEX_URL` 可覆盖)、条目校验(类型/黑名单/命令/URL 正则)、运行时环境检查(npm/docker/uvx 缺失时中文提示)、包预热/镜像拉取(进度流式推送 `INSTALL_PROGRESS`)、PyPI/Docker 补偿清理与 JSONL 审计。
- **配置导出**:`mcp-export.ts` 把受管插件合并进 `~/.codex/config.toml`,缺失启动命令或凭证的服务器被跳过并汇报;`handlers/mcp.ts` 读取 TOML 时过滤 `plugin_` 前缀段(视为孤儿段),把未受管服务器(如 searxng)作为「外部服务器」展示。
- **运行状态**:`mcp-manager.ts` 订阅 app-server 推送的 `mcpServer/startupStatus/updated`,并支持主动查询 `mcpServerStatus/list` 与重载 `config/mcpServer/reload`;启动失败(failed)会清空工具数,避免「已可用」假象。

## 2. 架构与数据流

```text
渲染侧 pluginStore / marketStore(registry.ts)
        │ PLUGINS_*/MARKET_*(preload → IPC)          │ INSTALL_PROGRESS(推)
        ▼                                             ▼
handlers/plugins.ts registerPluginsHandlers(window, mcpManager)
  ├─ PLUGINS_READ/WRITE → userData/plugins.json
  ├─ PLUGINS_EXPORT_MCP ──┐
  ├─ PLUGINS_MARKET_LIST ─┤
  ├─ PLUGINS_MARKET_INSTALL ── market-index.ts: validateInstall → checkRuntimeEnv
  │                              → warmupNpmPackage / pullDockerImage / verifyPypiPackage / verifyRemoteUrl
  │                              → auditInstall(JSONL) → exportMcpServersToCodexConfig
  ├─ PLUGINS_INSTALL_PACKAGE → npm exec 预热(activeInstalls 可取消)
  └─ PLUGINS_TEST_CONNECTION → testStdioConnection / testUrlConnection(mcp-client.ts)

handlers/mcp.ts ── readPlugins / configTomlPath(~/.codex/config.toml) / readExternalServerIds
mcp-export.ts ─── parseMcpServersFromToml / parseMcpServerDefs / removeMcpServerSections
                  mergeConfigToml / exportMcpServersToCodexConfig / managedFeatures(7 项)

app-server ←── mcp-manager.ts: mcpServer/startupStatus/updated(订阅) / mcpServerStatus/list(查询) / config/mcpServer/reload(重载)
```

## 3. 实现分解

| 模块 | 行号 | 函数/常量 | 说明 |
|---|---|---|---|
| handlers/plugins.ts | L28 | registerPluginsHandlers({ window, mcpManager }) | 全部插件 IPC 入口 |
| | L29-45 | PLUGINS_READ | 读 `userData/plugins.json`,缺失返回 `[]` |
| | L46-56 | PLUGINS_WRITE | 全量写回 plugins.json |
| | L58-68 | PLUGINS_EXPORT_MCP | 调用 exportMcpServersToCodexConfig 生成/更新 config.toml |
| | L70-96 | PLUGINS_CHECK_DEPS | 逐插件运行时依赖检查(checkRuntimeEnv 聚合) |
| | L97-152 | PLUGINS_INSTALL_PACKAGE | 包名正则校验(`@?[\w.-]+(/[\w.-]+)?`)→ `npm exec --yes --package=<pkg> -c "exit 0"`(win 下 npm.cmd + shell);stderr 逐行发 `INSTALL_PROGRESS`(phase=downloading),close 发 done/error;300s 超时;`activeInstalls` 登记可取消 |
| | L154-165 | PLUGINS_INSTALL_CANCEL | kill 进程 + 删登记 + 发「安装已取消」进度 |
| | L167-169 | PLUGINS_MARKET_LIST | fetchMarketIndex(force=true 绕过缓存) |
| | L171-236 | PLUGINS_MARKET_INSTALL | 见下方关键流程 |
| | L238-253 | PLUGINS_MARKET_CLEANUP | 镜像名正则校验 → `docker rmi --force` |
| | L255-273 | PLUGINS_TEST_CONNECTION | 参数校验;有 url → testUrlConnection,否则 testStdioConnection |
| | L276-339 | testStdioConnection | `buildInitializeRequest()` 写 JSON-RPC initialize 到 stdin,读 stdout 经 `parseMcpStdioResponse` 解析,受 `MCP_TEST_TIMEOUT_MS` 约束 |
| | L340-391 | testUrlConnection | `parseMcpHttpResponse` 解析 HTTP 响应 |
| handlers/mcp.ts | L12-30 | pluginsFilePath / readPlugins / writePlugins | plugins.json 读写 |
| | L32-38 | configTomlPath | `~/.codex/config.toml` |
| | L40+ | readExternalServerIds | 解析 TOML 中受管之外的全部段;`plugin_` 前缀视为孤儿段(残留)剔除,余下(如 searxng)列为外部服务器 |
| codex/market-index.ts | L18-22 | DEFAULT_MARKET_INDEX_URL / INDEX_TTL_MS=1h / FETCH_TIMEOUT_MS=10s / WARMUP_TIMEOUT_MS=300s / DEFAULT_AUDIT_DIR=~/.mindos/audit | 关键常量 |
| | L24-29 | COMMAND_ALLOWLIST / PACKAGE_NAME_RE / DOCKER_IMAGE_RE / URL_RE / PACKAGE_TYPES / INDEX_SOURCES | 校验字典:npx/npx.cmd/uvx/uv/uvx.exe/uv.exe/docker/docker.exe |
| | L31-35 | NETWORK_ERROR_RE / MISSING_CMD_RE / MARKET_BLACKLIST | 错误归类与黑名单 |
| | L40-43 | marketIndexUrl | 环境变量 `MINDO_MARKET_INDEX_URL` 覆盖默认地址 |
| | L45-51 / L53-89 | setMarketAuditDirForTest / resetMarketCache / auditDir / 缓存与 fetch | 缓存命中(1h TTL)与强制刷新 |
| | L91-115 | isValidEntry | 远端条目结构校验 |
| | L116-122 | isBlacklisted | 黑名单命中 |
| | L123-147 | validateInstall | 名称/命令/URL/包名/类型校验,返回中文错误串或 null |
| | L148-158 | commandExists | PATH 探测(含 .cmd/.exe) |
| | L159-173 | checkRuntimeEnv | npm/docker/uvx 缺失 → 中文安装提示 |
| | L174-191 | friendlyInstallError | MISSING_CMD_RE → 安装 Docker Desktop/uv/Node.js 引导文案;NETWORK_ERROR_RE → 「网络连接失败…」 |
| | L192-217 | warmupNpmPackage | npm exec 预热,stderr 逐行回调进度 |
| | L220-225 / L228-230 | pullDockerImage / cleanupDockerImage | docker pull / docker rmi --force(补偿清理) |
| | L237-244 | verifyRemoteUrl | HEAD 探测;任何 HTTP 响应(含 403/404/5xx)视为端点存在,仅网络层错误判不可达(OAuth 网关考虑) |
| | L247-260 | verifyPypiPackage | PyPI JSON API 查询包存在性 |
| | L269-299 | runSimpleSpawn | 统一 spawn 包装:stdout/stderr 逐行回调、300s 超时、windowsHide |
| | L301-322 | InstallAuditInfo / auditInstall | 按天轮转 JSONL(`plugin-install-YYYYMMDD.jsonl`),失败静默 |
| codex/mcp-client.ts | L1-14 / L16-27 | McpJsonRpcResponse / buildInitializeRequest | initialize 请求文本构建 |
| | L29-44 / L46-62 / L64+ | parseJsonResponse / parseMcpStdioResponse / parseMcpHttpResponse | stdio 缓冲切帧解析 / HTTP 文本解析 |
| codex/mcp-manager.ts | L5-10 | McpServerRuntimeInfo | `{ state, error, toolsCount, at }` |
| | L12-15 | STARTUP_STATUS_METHOD / LIST_STATUS_METHOD / RELOAD_METHOD / MAX_LOG_ENTRIES=20 | `mcpServer/startupStatus/updated` / `mcpServerStatus/list` / `config/mcpServer/reload` |
| | L33-53 | McpManager(ctor / onStatusChanged / recordLog / getLogs) | 构造时经 AppServerManager 订阅启动状态推送;日志滚动上限 20 条 |
| | L56-78 | list | `mcpServerStatus/list`(detail=toolsAndAuthOnly,15s 超时);未收到过推送的新项状态置 `unknown` 而非凭空 ready;ready 保留 toolsCount |
| | L80-90 | reload | `config/mcpServer/reload`(30s);app-server 未连接直接抛错 |
| | L92-117 | handleStartupStatus | unknown 状态记 warn 并跳过;仅 ready 保留 toolsCount,failed/cancelled/starting 清空(防残留旧工具数);变更推送 statusListener |
| codex/mcp-export.ts | L14-47 | toTomlLiteral / mcpServersToToml | TOML 字面量转义与服务器段生成 |
| | L49-62 / L63-258 | parseMcpServersFromToml / parseMcpServerDefs(解析链) | TOML 段名提取与定义解析(stripTomlComment / extractTomlStringValue / splitTopLevel / parseTomlStringArray / parseTomlInlineTable) |
| | L259-305 | removeMcpServerSections | 按 id 移除 TOML 段 |
| | L306-388 | mergeConfigToml | 合并:顶层键/注释保留、受管 `[mcp_servers.*]` 段替换、`[features]` 段受管键(key=value)重写、非受管内容原样保留、尾部空行清理 |
| | L390-394 / L395+ | SkippedMcpServer / exportMcpServersToCodexConfig | 导出主入口;缺启动命令 / 缺环境变量凭证的服务器标记 skipped |
| | L473 | managedFeatures | 受管 feature 键白名单(7 项,合并时专用) |
| 渲染侧 pluginStore.ts | L83-87 | STORAGE_KEY='mindos-plugins' / BUILTINS_VERSION='5' / BUILTINS_VERSION_KEY / WARMUP_RETRY_KEY | 本地持久化与内置版本迁移键 |
| | L89+ | BUILTIN_PLUGINS / CATEGORY_LABELS / remoteToDraft(698) / remoteCommandArgs(681) | 内置模板、分类标签、远端→草稿转换 |
| | L723 | usePluginStore | 插件 store(安装进度订阅、重试键等) |
| 渲染侧 registry.ts | L18-20 | SOURCE_PRIORITY=['builtin','registry','smithery','github'] | 市场来源优先级去重 |
| | L36-85 | mergeMarketEntries / searchMarketEntries | 市场条目合并(按来源优先级)与关键词搜索 |
| | L118-125 / L136 | hasMarketUpdate / useMarketStore | 更新提示判定与市场 store |

### 3.1 关键流程:PLUGINS_MARKET_INSTALL(plugins.ts:171)

1. 请求结构校验 + `confirmed !== true` 拒绝(`PERMISSION_DENIED`)。
2. `validateInstall` 校验名称/命令/URL/包名/类型,不过 → `INVALID_PARAMS`。
3. 按包类型选择安装路径:`checkRuntimeEnv` 缺失(npm/docker/uvx)→ 中文提示;docker → `pullDockerImage`(进度流式转发);pypi → `verifyPypiPackage`;npm/未指定 → `warmupNpmPackage`;仅 url → `verifyRemoteUrl`。
4. `auditInstall` 落 JSONL 审计;审计抛错且 docker 已成功时,补偿 `docker rmi --force` 并返回「安装失败,请重试」。
5. 失败返回友好错误;成功返回 `{ success, pluginId, phase: 'downloaded' }`,渲染侧随后落插件定义并触发 `PLUGINS_EXPORT_MCP` 更新 config.toml。

## 4. 状态与数据模型

- 插件定义持久化于 `userData/plugins.json`;运行时状态(`McpServerRuntimeInfo`)仅存在于主进程内存,推送渲染侧。
- `~/.codex/config.toml` 受管段 = 插件段(`plugin_` 前缀)+ 受管 feature 键;`SkippedMcpServer` 记录缺失命令/凭证被跳过的服务器。
- 安装审计:`~/.mindos/audit/plugin-install-YYYYMMDD.jsonl`(JSONL 按天轮转)。
- 市场缓存:内存 1h TTL,`force` 可绕过;来源优先级 `builtin > registry > smithery > github`(渲染侧去重)。

## 5. 边界与异常

- 输入校验:包名/镜像名/URL 正则前置校验,非法参数 `INVALID_PARAMS`;安装未确认拒绝。
- 运行时环境缺失:中文引导文案(npm/Docker/uvx),不发起无效安装。
- 网络失败:统一友好文案;PyPI 404 明确提示「包不存在」;URL 探测容忍 403/404/5xx(仅网络层错误判失败)。
- 安装取消:`activeInstalls` 登记进程可被 kill;进度通道 `INSTALL_PROGRESS` 推送 downloading/done/error 三态。
- 失败补偿:docker 镜像拉取成功后审计失败 → 强制 rmi 清理防残留。
- TOML 合并只动受管段与受管 feature 键,用户注释与自定义内容保留;孤儿 `plugin_` 段在读取侧被过滤,不展示为外部服务器。
- MCP 运行状态:未推送过的新项状态保持 `unknown`;failed/cancelled 清空 toolsCount;unknown 状态记 warn 不落库。
- 重载需 app-server 已连接,否则明确报错「app-server 未连接,无法重载 MCP 配置」。

## 6. 相关接口与文档

- IPC 通道:`PLUGINS_READ/WRITE/EXPORT_MCP/CHECK_DEPS/INSTALL_PACKAGE/INSTALL_CANCEL/MARKET_LIST/MARKET_INSTALL/MARKET_CLEANUP/TEST_CONNECTION`、`INSTALL_PROGRESS`(主→渲染),常量见 `electron/ipc/channels.ts`。
- 关联文档:07(app-server 连接与 ws 调用)、01(自动化事件/状态推送渲染侧),config.toml 的写隔离外另见 03(worktree 不覆盖本文件)。

## 7. 关键文件索引

- `electron/ipc/handlers/plugins.ts`(391 行)、`electron/ipc/handlers/mcp.ts`(179 行)
- `electron/codex/market-index.ts`(322 行)、`mcp-manager.ts`(118 行)、`mcp-client.ts`、`mcp-export.ts`(492 行)
- `src/stores/pluginStore.ts`、`src/stores/registry.ts`、`src/types/chat.ts`(RemotePlugin 等类型源)
