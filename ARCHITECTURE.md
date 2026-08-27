# MindOS Desktop — 功能与优势总览

> 给专业读者的产品功能与架构概览。实现细节与源码行号见 [`docs/features/`](features/)、[`PROTOCOL-COMPAT.md`](PROTOCOL-COMPAT.md)；维护规则见 [`AGENTS.md`](../AGENTS.md)；分发许可见 [`LICENSE.md`](../LICENSE.md)。

---

## 1. 产品定位

**MindOS Desktop 是一个"本地优先"的桌面 AI 编程助手：内置开源 Codex CLI 引擎，把「AI 编程能力」做成开箱即用的 Windows 桌面产品 —— 不用装环境、不用配终端、不用理解协议，打开就说需求。**

```
┌──────────────────────────────────────────────────────────┐
│ Renderer (Vue 3 + Pinia) — 对话/面板/设置/PR 视图         │
├──────────────────────────────────────────────────────────┤
│ Main Process (Electron) — IPC · 白名单 · 审计 · 沙箱      │
├──────────────────────────────────────────────────────────┤
│ Codex AppServer(codex-cli 0.149.0,官方二进制,外部进程)    │
├──────────────────────────────────────────────────────────┤
│ 本机能力 — PowerShell 7 · 终端 · Git 工作树 · 浏览器      │
└──────────────────────────────────────────────────────────┘
```

---

## 2. 功能全景

### 2.1 对话与协作
- 多线程会话（并行任务互不阻塞）、子代理派发、快照持续化与恢复（下次启动回来继续）
- 全量事件驱动 UI：变更卡片、原型预览、审批弹窗、控制台日志、token 用量与费用估算
- 自动化任务链：设置开关驱动 —— 提交自动审查（8 维度代码审查）、完成后自动测试（检测编译失败/测试失败/命令并自动重跑）
- 引用上下文：@提及 归档文件、粘贴图片、快捷键体系（`src/utils/shortcuts.ts`）

### 2.2 文件与变更（可视化是核心体验）
- **变更双源对账**：AI 工具事件（apply_patch / shell 写文件探测）与 git status 合并；每文件给出 状态、行数 diff、已应用标记
- **无 git 也能看 diff**：非 git 项目走「磁盘 vs 快照」生成差异，空态也有兜底（`utils/nonGitDiff.ts`）
- 文件树 / 目录浏览 / 原型卡片 / 图片预览，一处对话全链路呈现

### 2.3 Git 与拉取请求（写隔离工作流）
- **写隔离工作树**：AI 在独立 worktree 干活，主目录零污染；环境信息按钮实时显示「写隔离 / 未隔离」
- 环境面板一站式：变更统计、**创建分支**、**提交**（detached HEAD 防呆：无分支先建分支）、**推送**（主树执行 + 绕过 codex 保护钩子）、**打开/创建 PR**（GitHub API + 浏览器直达）
- PR 全生命周期：待合并 → 打开 GitHub → 合并后「已完成」

### 2.4 权限与安全
- **审批三档**：标准审批 / AI 代审（信任模式）/ 完全信任（沙箱内直接执行）
- Windows 沙箱指令：受限 token 执行、越界写入直接拒绝、`require_escalated` 提权需用户批准
- **路径白名单**（ipc 全链路校验 + 持久化）、**凭据库**（站点密码/SSH 加密存储）、GitHub token 存储
- 审计通知链（关键动作留痕）、app:// 私有协议与 webContents 加固

### 2.5 本地引擎（开箱即用）
- 内置 `codex-cli 0.149.0` 全家桶（core + command-runner + sandbox-setup + code-mode-host + skills），**用户零安装**
- 打包后强制走 `resources/` 内部二进制（不依赖 PATH）；开发模式完整探测链
- PowerShell 7 一等执行（沙箱指令引导 AI 使用 pwsh.exe，回退 powershell 5.1 / cmd /c）

### 2.6 上下文与任务路由（确定性智能）
- **DTR 任务路由**：7 类任务确定性分类（bug_fix / feature / architecture / code_review / prototype / data_query / generic），关键词规则零 AI 判断
- **能力域映射**：11 个能力域 → 精确工具名（文件/git/搜索/浏览器/桌面/数据…），随分类注入 `<context>` 提示块
- **混合路由**：按 模型视觉能力 × 平台 动态裁剪 —— 非视觉模型桌面操作降级为结构化 uia 路径；非 Windows 剔除浏览器/桌面域
- 分层上下文：意图链 / 关键决策 / Skills / 项目上下文（AGENTS.md + 文件树 + 近期改动），按任务类型重建（bug 拉 git 改动、feature 深树）

### 2.7 插件 · 技能 · 浏览器 · 终端
- 插件市场 21+：联网检索（Context7/Fetch/Brave/Tavily）、数据库（SQLite）、浏览器自动化（Playwright）、图表（Mermaid）、Git、Memory、文件系统、Notion、Open Design 等，一键开关
- 技能工作流库：创意/设计/产品模板（moodboard、design→code、code-review、polish、offer…）
- 内置浏览器（webview）+ computer-use（browser_open/get_ax/click/type + uia 桌面操作 + screenshot 视觉定位）
- 原生终端（xterm + node-pty）随会话联动

### 2.8 系统与工程
- 27 套主题（Catppuccin 等）、壁纸、字体/字号、自定义强调色
- 模型连接/校验/多 Provider（DeepSeek / OpenAI 兼容 / 本地 Ollama），token 成本实时估算
- 窗口状态记忆、就绪状态提示、日志（`%AppData%\mindos-desktop\logs\`）
- **自动更新**：每次启动检查 → 提示 → 点击下载 → 重启生效（无遥测、无强制）
- 打包体系：产物与源码分离（`D:/MindOS-Dist`）、NSIS 安装版 + 免安装 zip/文件夹、镜像网络、产物安全检查

---

## 3. 核心优势（为什么值得用）

| # | 优势 | 说明 |
|---|---|---|
| 1 | **开箱即用，零环境** | AI 引擎内置；没有 Node/Python/环境变量问题，装完就能聊 |
| 2 | **写隔离，主目录零污染** | AI 在隔离树干活；出问题不影响你手边的代码；一条龙提交/推送/PR |
| 3 | **变更看得见（无 git 也有）** | 万行改动可视化到行级 diff；不是 git 项目也照样给差异 |
| 4 | **确定性任务路由** | 分类用预设规则，不靠 AI 猜 —— 可控、可测、可审计（区别于"全让模型自己判断"的方案） |
| 5 | **混合路由适配模型** | 视觉/非视觉模型各走各的路径，桌面操作永远可跑（截图协议或 uia 结构化） |
| 6 | **本地优先，绝对隐私** | 无账号、无遥测；数据只发你自己配置的 API；闭源许可 + 加密凭据 |
| 7 | **审批与沙箱** | 三档审批 + Windows 沙箱 + 白名单 + 审计，给"AI 替你干活"上保险 |
| 8 | **PowerShell 7 一等公民** | 专为 Windows 打磨的执行链路（pwsh 优先、回退链明确） |
| 9 | **插件+技能生态** | 检索/数据库/浏览器自动化/设计工作流一键开，扩展即插即用 |
| 10 | **工程化安心** | 内置 codex 合规署名、自动更新、协议兼容契约、抗反编译打包（minify+安全检查）、镜像分发 |

---

## 4. 兼容性与升级纪律

- `docs/PROTOCOL-COMPAT.md` 绑定 `codex-cli 0.149.0`、上游基线 `rust-v0.149.0(openai/codex)`，含回归表与升级 SOP
- 内核升级只在 `electron/codex/event-converter.ts` 事件白名单一处收敛（协议适配唯一改动面）
- 质量门：vitest 2855+（单元 + 主进程 handler + 集成）、双 typecheck、覆盖率红线 ≥85.42%、UI 设计令牌规范

---

## 5. 许可证与署名声明

> **本软件为闭源免费软件，分发条款以 [`LICENSE.md`](../LICENSE.md) 为准**（禁止商业用途 / 逆向工程 / 二次分发 / 去除版权标识）。

**第三方组件归属**：MindOS Desktop 使用 [openai/codex](https://github.com/openai/codex)（Apache-2.0）官方构建产物 `codex-cli 0.149.0`，作为**独立外部进程**经 app-server 协议集成；**未修改、未分叉、未派生其任何源码**；适配层（协议/IPC/渲染层）为 MindOS 自有原创代码。所有权与署名归 OpenAI 及其 codex 项目贡献者所有，第三方的 MIT/Apache-2.0 组件仍受各自原始许可约束。
