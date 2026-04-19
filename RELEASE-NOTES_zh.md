# Superpowers 发布说明

## v5.0.7 (2026-03-31)

### GitHub Copilot CLI 支持

- **SessionStart 上下文注入** — Copilot CLI v1.0.11 在 sessionStart hook 输出中添加了 `additionalContext` 支持。session-start hook 现在会检测 `COPILOT_CLI` 环境变量并输出 SDK 标准的 `{ "additionalContext": "..." }` 格式，让 Copilot CLI 用户在会话启动时获得完整的 superpowers 引导。（原始修复由 @culinablaz 在 PR #910 中完成）
- **工具映射** — 添加了 `references/copilot-tools.md`，包含完整的 Claude Code 到 Copilot CLI 工具对照表
- **技能和 README 更新** — 在 `using-superpowers` 技能的平台说明和 README 安装部分添加了 Copilot CLI

### OpenCode 修复

- **技能路径一致性** — 引导文本不再宣传误导性的 `configDir/skills/superpowers/` 路径，该路径与运行时路径不匹配。Agent 应使用原生 `skill` 工具，而不是通过路径导航到文件。测试现在使用从单一真相源派生的一致路径。（#847, #916）
- **引导作为用户消息** — 将引导注入从 `experimental.chat.system.transform` 移至 `experimental.chat.messages.transform`，添加到第一条用户消息之前而不是添加系统消息。避免系统消息每轮重复导致的 token 膨胀（#750），并修复与 Qwen 和其他不支持多个系统消息的模型的兼容性（#894）。

## v5.0.6 (2026-03-24)

### 内联自审替代子代理审核循环

子代理审核循环（派遣新的代理来审核计划/规格）使执行时间翻倍（约 25 分钟开销），但并未显著提高计划质量。跨越 5 个版本、每个版本 5 次试验的回归测试显示，无论审核循环是否运行，质量评分相同。

- **brainstorming** — 将规格审核循环（子代理派遣 + 3 次迭代上限）替换为内联规格自审检查清单：占位符扫描、内部一致性、范围检查、歧义检查
- **writing-plans** — 将计划审核循环（子代理派遣 + 3 次迭代上限）替换为内联自审检查清单：规格覆盖、占位符扫描、类型一致性
- **writing-plans** — 添加了明确的"无占位符"部分，定义计划失败情况（TBD、模糊描述、未定义引用、"类似于任务 N"）
- 自审在约 30 秒内捕获 3-5 个真实 bug，而不是约 25 分钟，缺陷率与子代理方法相当

### Brainstorm 服务器

- **会话目录重构** — brainstorm 服务器会话目录现在包含两个并列子目录：`content/`（向浏览器提供的 HTML 文件）和 `state/`（事件、server-info、pid、日志）。此前，服务器状态和用户交互数据与提供的内容一起存储，可通过 HTTP 访问。`screen_dir` 和 `state_dir` 路径都包含在 server-started JSON 中。（由吉田仁报告）

### Bug 修复

- **Owner-PID 生命周期修复** — brainstorm 服务器 的 owner-PID 监控有两个 bug 导致 60 秒内错误关闭：（1）跨用户 PID 的 EPERM（Tailscale SSH 等）被视为"进程已死亡"，（2）在 WSL 上，祖父 PID 解析为在第一次生命周期检查之前退出的短生命子进程。修复方法是将 EPERM 视为"存活"，并在启动时验证 owner PID — 如果已死亡，则禁用监控，服务器依赖 30 分钟空闲超时。这也移除了 `start-server.sh` 中 Windows/MSYS2 的特殊处理，因为服务器现在通用处理。（#879）
- **writing-skills** — 修正了 SKILL.md frontmatter 只支持"两个字段"的错误说法；现在说"两个必需字段"并链接到 agentskills.io 规范以获取所有支持的字段（PR #882 by @arittr）

### Codex App 兼容性

- **codex-tools** — 添加了命名代理派遣映射，记录如何将 Claude Code 的命名代理类型转换为 Codex 的带 worker roles 的 `spawn_agent`（PR #647 by @arittr）
- **codex-tools** — 为 worktree-aware 技能添加了环境检测和 Codex App 完成部分（by @arittr）
- **设计规范** — 添加了 Codex App 兼容性设计规范（PRI-823），涵盖只读环境检测、worktree-safe 技能行为和沙箱回退模式（by @arittr）

## v5.0.5 (2026-03-17)

### Bug 修复

- **Brainstorm 服务器 ESM 修复** — 将 `server.js` 重命名为 `server.cjs`，以便 brainstorm 服务器在 Node.js 22+ 上正确启动，因为根目录 `package.json` 的 `"type": "module"` 导致 `require()` 失败。（PR #784 by @sarbojitrana，修复 #774, #780, #783）
- **Windows 上的 Brainstorm owner-PID** — 在 Windows/MSYS2 上跳过 PID 生命周期监控，因为 PID namespace 对 Node.js 不可见，防止服务器在 60 秒后自终止。（#770，文档来自 PR #768 by @lucasyhzlu-debug）
- **stop-server.sh 可靠性** — 在报告成功之前验证服务器进程确实已死亡。SIGTERM + 2s 等待 + SIGKILL 回退。（#723）

### 变更

- **执行移交** — 在计划编写后恢复用户在子代理驱动和内联执行之间的选择。推荐子代理驱动但不再强制。

## v5.0.4 (2026-03-16)

### 审核循环优化

通过消除不必要的审核轮次和收紧审核焦点，显著减少 token 使用并加快规格和计划审核。

- **单次全计划审核** — 计划审核者现在一次性审核完整计划，而不是逐块审核。移除所有块相关概念（`## Chunk N:` 标题、1000 行块限制、每块派遣）。
- **提高阻塞问题的门槛** — 规格和计划审核者提示现在都包含"校准"部分：只标记会在实现过程中造成真正问题的问题。轻微措辞、风格偏好和格式细节不应阻止批准。
- **减少最大审核迭代次数** — 从 5 次减少到 3 次，适用于规格和计划审核循环。如果审核者校准正确，3 轮足够。
- **精简审核检查清单** — 规格审核者从 7 个类别精简到 5 个；计划审核者从 7 个精简到 4 个。移除格式相关的检查（任务语法、块大小），关注实质内容（可构建性、规格对齐）。

### OpenCode

- **单行插件安装** — OpenCode 插件现在通过 `config` hook 自动注册技能目录。无需符号链接或 `skills.paths` 配置。安装只需在 `opencode.json` 中添加一行。（PR #753）
- **添加了 `package.json`**，使 OpenCode 可以从 git 安装 superpowers 作为 npm 包。

### Bug 修复

- **验证服务器确实已停止** — `stop-server.sh` 现在在报告成功之前确认进程已死亡。SIGTERM + 2s 等待 + SIGKILL 回退。如果进程存活则报告失败。（PR #751）
- **通用代理语言** — brainstorm companion 等待页面现在说"the agent"而不是"Claude"。

## v5.0.3 (2026-03-15)

### Cursor 支持

- **Cursor hooks** — 添加了 `hooks/hooks-cursor.json`，使用 Cursor 的 camelCase 格式（`sessionStart`, `version: 1`），并更新 `.cursor-plugin/plugin.json` 引用它。修复 `session-start` 中的平台检测，优先检查 `CURSOR_PLUGIN_ROOT`（Cursor 也可能设置 `CLAUDE_PLUGIN_ROOT`）。（基于 PR #709）

### Bug 修复

- **停止在 `--resume` 时触发 SessionStart hook** — 启动 hook 在恢复的会话上重新注入上下文，这些会话的历史记录中已有上下文。Hook 现在只在 `startup`、`clear` 和 `compact` 时触发。
- **Bash 5.3+ hook 挂起** — 在 `hooks/session-start` 中将 heredoc（`cat <<EOF`）替换为 `printf`。修复 macOS 上 Homebrew bash 5.3+ 因 bash 在 heredoc 中处理大变量展开的回归导致的无限挂起。（#572, #571）
- **POSIX-safe hook script** — 在 `hooks/session-start` 中将 `${BASH_SOURCE[0]:-$0}` 替换为 `$0`。修复 Ubuntu/Debian 上 `/bin/sh` 是 dash 时出现的"Bad substitution"错误。（#553）
- **Portable shebangs** — 在所有 shell 脚本中将 `#!/bin/bash` 替换为 `#!/usr/bin/env bash`。修复 NixOS、FreeBSD 和 macOS 上 Homebrew bash 中 `/bin/bash` 过时或缺失时的执行问题。（#700）
- **Windows 上的 Brainstorm 服务器** — 自动检测 Windows/Git Bash（`OSTYPE=msys*`, `MSYSTEM`）并切换到前台模式，修复 `nohup`/`disown` 进程回收导致的静默服务器失败。（#737）
- **Codex 文档修复** — 在 Codex 文档中将已弃用的 `collab` 标志替换为 `multi_agent`。（PR #749）

## v5.0.2 (2026-03-11)

### 零依赖 Brainstorm 服务器

**移除所有 vendored node_modules — server.js 现在完全自包含**

- 用零依赖 Node.js 服务器替换 Express/Chokidar/WebSocket 依赖，使用内置 `http`、`fs` 和 `crypto` 模块
- 移除了约 1,200 行 vendored `node_modules/`、`package.json` 和 `package-lock.json`
- 自定义 WebSocket 协议实现（RFC 6455 framing, ping/pong, proper close handshake）
- 原生 `fs.watch()` 文件监控替换 Chokidar
- 完整测试套件：HTTP 服务、WebSocket 协议、文件监控和集成测试

### Brainstorm 服务器可靠性

- **空闲 30 分钟后自动退出** — 当无客户端连接时服务器关闭，防止孤儿进程
- **Owner 进程追踪** — 服务器监控父 harness PID，当拥有会话死亡时退出
- **存活检查** — 技能验证服务器在重用现有实例之前有响应
- **编码修复** — 提供的 HTML 页面上正确的 `<meta charset="utf-8">`

### 子代理上下文隔离

- 所有委托技能（brainstorming、dispatching-parallel-agents、requesting-code-review、subagent-driven-development、writing-plans）现在包含上下文隔离原则
- 子代理只接收所需的上下文，防止上下文窗口污染

## v5.0.1 (2026-03-10)

### Agentskills 合规

**Brainstorm-server 移入技能目录**

- 将 `lib/brainstorm-server/` 移至 `skills/brainstorming/scripts/`，符合 [agentskills.io](https://agentskills.io) 规范
- 所有 `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/` 引用替换为相对 `scripts/` 路径
- 技能现在跨平台完全可移植 — 无需平台特定环境变量来定位脚本
- 移除了 `lib/` 目录（曾是最后一个残留内容）

### 新功能

**Gemini CLI 扩展**

- 通过 `gemini-extension.json` 和根目录 `GEMINI.md` 支持原生 Gemini CLI 扩展
- `GEMINI.md` 在会话启动时 @import `using-superpowers` 技能和工具映射表
- Gemini CLI 工具映射参考（`skills/using-superpowers/references/gemini-tools.md`）— 将 Claude Code 工具名（Read, Write, Edit, Bash 等）翻译为 Gemini CLI 等价物（read_file, write_file, replace 等）
- 文档 Gemini CLI 限制：无子代理支持，技能回退到 `executing-plans`
- 扩展根在 repo 根目录以实现跨平台兼容性（避免 Windows 符号链接问题）
- README 中添加了安装说明

### 改进

**多平台 brainstorm 服务器启动**

- visual-companion.md 中的每平台启动说明：Claude Code（默认模式）、Codex（通过 `CODEX_CI` 自动前台）、Gemini CLI（带 `is_background` 的 `--foreground`）和其他环境的回退
- 服务器现在将启动 JSON 写入 `$SCREEN_DIR/.server-info`，以便 agent 即使 stdout 被后台执行隐藏也能找到 URL 和端口

**Brainstorm 服务器依赖捆绑**

- `node_modules` vendored 到 repo，以便 brainstorm 服务器在全新插件安装时无需运行时 `npm` 即可工作
- 从捆绑依赖中移除 `fsevents`（macOS 专用原生二进制；chokidar 在没有它时优雅回退）
- 如果 `node_modules` 缺失，通过 `npm install` 回退自动安装

**OpenCode 工具映射修复**

- `TodoWrite` → `todowrite`（之前错误映射到 `update_plan`）；已对照 OpenCode 源码验证

### Bug 修复

**Windows/Linux: 单引号破坏 SessionStart hook** (#577, #529, #644, PR #585)

- hooks.json 中 `${CLAUDE_PLUGIN_ROOT}` 周围的单引号在 Windows 上失败（cmd.exe 不将单引号识别为路径分隔符）和在 Linux 上失败（单引号阻止变量展开）
- 修复：将单引号替换为转义双引号 — 在 macOS bash、Windows cmd.exe、Windows Git Bash 和 Linux 上都能工作，包括路径中有空格和无空格的情况
- 已在 Windows 11（NT 10.0.26200.0）上用 Claude Code 2.1.72 和 Git for Windows 验证

**Brainstorming 规格审核循环被跳过** (#677)

- 规格审核循环（派遣 spec-document-reviewer 子代理，迭代直到批准）存在于"After the Design"文本部分，但在检查清单和流程图中缺失
- 由于 agent 更可靠地遵循图和检查清单而不是文本，规格审核步骤被完全跳过
- 在检查清单中添加了步骤 7（规格审核循环）和在 dot graph 中添加了对应节点
- 用 `claude --plugin-dir` 和 `claude-session-driver` 测试：worker 现在正确派遣审核者

**Cursor 安装命令** (PR #676)

- 修复 README 中的 Cursor 安装命令：`/plugin-add` → `/add-plugin`（通过 Cursor 2.5 发布公告确认）

**Brainstorming 中的用户审核门** (#565)

- 在规格完成和 writing-plans 移交之间添加了明确的用户审核步骤
- 用户必须在实现计划开始之前批准规格
- 检查清单、流程图和文本已用新门更新

**Session-start hook 每平台只输出一次上下文**

- Hook 现在检测它是在 Claude Code 还是其他平台运行
- 为 Claude Code 输出 `hookSpecificOutput`，为其他平台输出 `additional_context` — 防止双重上下文注入

**Token 分析脚本中的 linting 修复**

- `tests/claude-code/analyze-token-usage.py` 中 `except:` → `except Exception:`

### 维护

**移除死代码**

- 删除了 `lib/skills-core.js` 及其测试（`tests/opencode/test-skills-core.js`）— 自 2026 年 2 月起未使用
- 从 `tests/opencode/test-plugin-loading.sh` 移除了 skills-core 存在检查

### 社区

- @karuturi — Claude Code 官方市场安装说明（PR #610）
- @mvanhorn — session-start hook 双重输出修复，OpenCode 工具映射修复
- @daniel-graham — bare except 的 linting 修复
- PR #585 作者 — Windows/Linux hooks 引号修复

---

## v5.0.0 (2026-03-09)

### 破坏性变更

**规格和计划目录重构**

- 规格（brainstorming 输出）现在保存到 `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
- 计划（writing-plans 输出）现在保存到 `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- 用户对规格/计划位置的偏好覆盖这些默认值
- 所有内部技能引用、测试文件和示例路径已更新匹配
- 迁移：如果需要，将现有文件从 `docs/plans/` 移至新位置

**在有能力的 harness 上强制子代理驱动开发**

Writing-plans 不再提供子代理驱动和 executing-plans 之间的选择。在有子代理支持的 harness（Claude Code, Codex）上，subagent-driven-development 是必需的。Executing-plans 保留给无子代理能力的 harness，现在告诉用户 Superpowers 在有子代理能力的平台上工作更好。

**Executing-plans 不再批量**

移除了"执行 3 个任务然后停下来审核"的模式。计划现在持续执行，只在遇到阻塞时停止。

**Slash 命令已弃用**

`/brainstorm`、`/write-plan` 和 `/execute-plan` 现在显示弃用通知，指向用户对应技能。命令将在下一个主要版本中移除。

### 新功能

**可视化 brainstorming companion**

Brainstorming 会话的可选浏览器 companion。当主题受益于可视化时，brainstorming 技能提议在浏览器窗口中显示 mockup、图表、比较和其他内容，与终端对话并行。

- `lib/brainstorm-server/` — WebSocket 服务器，带有浏览器 helper 库、会话管理脚本和深色/浅色主题框架模板（带 GitHub 链接的"Superpowers Brainstorming")
- `skills/brainstorming/visual-companion.md` — 服务器工作流、screen 创建和反馈收集的渐进披露指南
- Brainstorming 技能在流程图中添加了可视化 companion 决策点：在探索项目上下文后，技能评估即将到来的问题是否涉及可视化内容，并在自己的消息中提议 companion
- 每问题决策：即使接受后，每个问题都评估浏览器或终端更合适
- `tests/brainstorm-server/` 中的集成测试

**文档审核系统**

使用子代理派遣的规格和计划文档自动审核循环：

- `skills/brainstorming/spec-document-reviewer-prompt.md` — 审核者检查完整性、一致性、架构和 YAGNI
- `skills/writing-plans/plan-document-reviewer-prompt.md` — 审核者检查规格对齐、任务分解、文件结构和文件大小
- Brainstorming 在编写设计文档后派遣规格审核者
- Writing-plans 包含每个部分后的块级计划审核循环
- 审核循环重复直到批准或在 5 次迭代后升级
- `tests/claude-code/test-document-review-system.sh` 中的端到端测试
- `docs/superpowers/` 中的设计规范和实现计划

**技能管道中的架构指导**

在 brainstorming、writing-plans 和 subagent-driven-development 中添加了设计隔离和文件大小意识指导：

- **Brainstorming** — 新部分："设计隔离和清晰性"（清晰边界、定义良好的接口、独立可测试单元）和"在现有代码库中工作"（遵循现有模式、只做针对性改进）
- **Writing-plans** — 新"文件结构"部分：在定义任务之前规划文件和职责。新"范围检查"兜底：捕获应在 brainstorming 期间分解的多子系统规格
- **SDD implementer** — 新"代码组织"部分（遵循计划的文件结构、报告文件增长担忧）和"When You're in Over Your Head"升级指导
- **SDD 代码质量审核者** — 现在检查架构、单元分解、计划一致性和文件增长
- **规格/计划审核者** — 架构和文件大小添加到审核标准
- **范围评估** — Brainstorming 现在评估项目是否太大以至于单个规格无法处理。多子系统请求被早期标记并分解为子项目，每个有自己的规格 → 计划 → 实现循环

**子代理驱动开发改进**

- **模型选择** — 按任务类型选择模型能力的指导：便宜模型用于机械实现、标准模型用于集成、能力模型用于架构和审核
- **实现者状态协议** — 子代理现在报告 DONE、DONE_WITH_CONCERNS、BLOCKED 或 NEEDS_CONTEXT。Controller 适当处理每种状态：重新派遣带更多上下文、升级模型能力、分解任务或升级到人类

### 改进

**指令优先级层次**

在 using-superpowers 中添加了明确的优先级排序：

1. 用户明确指令（CLAUDE.md, AGENTS.md, 直接请求）— 最高优先级
2. Superpowers 技能 — 覆盖默认系统行为
3. 默认系统提示 — 最低优先级

如果 CLAUDE.md 或 AGENTS.md 说"不要使用 TDD"而技能说"总是使用 TDD"，用户指令优先。

**SUBAGENT-STOP 门**

在 using-superpowers 中添加了 `<SUBAGENT-STOP>` 块。为特定任务派遣的子代理现在跳过技能，而不是激活 1% 规则并调用完整技能工作流。

**多平台改进**

- Codex 工具映射移至渐进披露参考文件（`references/codex-tools.md`）
- 添加了平台适应指针，以便非 Claude Code 平台能找到工具等价物
- 计划标题现在称呼"agentic workers"而不是具体"Claude"
- `docs/README.codex.md` 中文档了 Collab 功能要求

**Writing-plans 模板更新**

- 计划步骤现在使用 checkbox 语法（`- [ ] **Step N:**`）进行进度追踪
- 计划标题引用 subagent-driven-development 和 executing-plans，带平台感知路由

---

## v4.3.1 (2026-02-21)

### 添加

**Cursor 支持**

Superpowers 现在通过 Cursor 的插件系统工作。包含 `.cursor-plugin/plugin.json` manifest 和 README 中 Cursor 专用的安装说明。SessionStart hook 输出现在包含 `additional_context` 字段，与现有 `hookSpecificOutput.additionalContext` 并列，以兼容 Cursor hook。

### 修复

**Windows: 恢复 polyglot wrapper 以实现可靠 hook 执行** (#518, #504, #491, #487, #466, #440)

Claude Code 在 Windows 上的 `.sh` 自动检测会在 hook 命令前添加 `bash`，破坏执行。修复：

- 将 `session-start.sh` 重命名为 `session-start`（无扩展名）以便自动检测不干扰
- 恢复带多位置 bash 发现（标准 Git for Windows 路径，然后 PATH 回退）的 `run-hook.cmd` polyglot wrapper
- 如果未找到 bash 则静默退出而不是报错
- 在 Unix 上，wrapper 通过 `exec bash` 直接运行脚本
- 使用 POSIX-safe 的 `dirname "$0"` 路径解析（在 dash/sh 上工作，不只是 bash）

这修复了 Windows 上带空格路径、缺少 WSL、MSYS 上 `set -euo pipefail` 脆弱性和反斜杠混乱的 SessionStart 失败。

## v4.3.0 (2026-02-12)

此修复应显著提高 superpowers 技能合规性并减少 Claude 无意中进入原生计划模式的几率。

### 变更

**Brainstorming 技能现在强制其工作流而不是描述它**

模型正在跳过设计阶段并直接跳到实现技能如 frontend-design，或将整个 brainstorming 过程折叠为单个文本块。技能现在使用硬门、强制检查清单和 graphviz 流程图来强制合规：

- `<HARD-GATE>`：在呈现设计并用户批准之前，无实现技能、代码或脚手架
- 必须作为任务创建并按顺序完成的明确检查清单（6 项）
- Graphviz 流程图，以 `writing-plans` 作为唯一有效终端状态
- "这太简单不需要设计"的反模式提醒 — 模型用来跳过过程的合理化方式
- 基于部分复杂度而不是项目复杂度的设计部分大小

**Using-superpowers 工作流图拦截 EnterPlanMode**

在技能流程图中添加了 `EnterPlanMode` 拦截。当模型即将进入 Claude 原生计划模式时，它检查 brainstorming 是否已发生并路由到 brainstorming 技能。计划模式从不进入。

### 修复

**SessionStart hook 现在同步运行**

在 hooks.json 中将 `async: true` 改为 `async: false`。当 async 时，hook 可能无法在模型第一轮之前完成，意味着 using-superpowers 指令不在第一条消息的上下文中。

## v4.2.0 (2026-02-05)

### 破坏性变更

**Codex: 用原生技能发现替换 bootstrap CLI**

移除了 `superpowers-codex` bootstrap CLI、Windows `.cmd` wrapper 和相关 bootstrap 内容文件。Codex 现在通过 `~/.agents/skills/superpowers/` 符号链接使用原生技能发现，所以旧的 `use_skill`/`find_skills` CLI 工具不再需要。

安装现在只是 clone + symlink（在 INSTALL.md 中文档）。无需 Node.js 依赖。旧的 `~/.codex/skills/` 路径已弃用。

### 修复

**Windows: 修复 Claude Code 2.1.x hook 执行** (#331)

Claude Code 2.1.x 改变了 hooks 在 Windows 上执行的方式：它现在自动检测命令中的 `.sh` 文件并添加 `bash` 前缀。这破坏了 polyglot wrapper 模式，因为 `bash "run-hook.cmd" session-start.sh` 尝试将 `.cmd` 文件作为 bash 脚本执行。

修复：hooks.json 现在直接调用 session-start.sh。Claude Code 2.1.x 自动处理 bash 调用。还添加了 .gitattributes 以强制 shell 脚本使用 LF 行尾（修复 Windows checkout 上的 CRLF 问题）。

**Windows: SessionStart hook 异步运行以防止终端冻结** (#404, #413, #414, #419)

同步 SessionStart hook 阻止 TUI 在 Windows 上进入 raw mode，冻结所有键盘输入。异步运行 hook 防止冻结，同时仍注入 superpowers 上下文。

**Windows: 修复 O(n^2) `escape_for_json` 性能**

使用 `${input:$i:1}` 的逐字符循环由于子字符串复制开销在 bash 中是 O(n^2)。在 Windows Git Bash 上这需要 60+ 秒。用 bash 参数替换（`${s//old/new}`）替换，每个模式作为单个 C 级别 pass 运行 — macOS 上快 7x，Windows 上显著更快。

**Codex: 修复 Windows/PowerShell 调用** (#285, #243)

- Windows 不尊重 shebangs，所以直接调用无扩展名的 `superpowers-codex` 脚本触发"打开方式"对话框。所有调用现在添加 `node` 前缀。
- 修复 Windows 上的 `~/` 路径展开 — PowerShell 在作为参数传递给 `node` 时不展开 `~`。改为 `$HOME`，在 bash 和 PowerShell 中都正确展开。

**Codex: 修复安装器中的路径解析**

用 `fileURLToPath()` 而不是手动 URL pathname 解析，正确处理所有平台上带空格和特殊字符的路径。

**Codex: 修复 writing-skills 中的陈旧技能路径**

更新 `~/.codex/skills/` 引用（已弃用）为 `~/.agents/skills/` 以原生发现。

### 改进

**实现前现在需要 worktree 隔离**

为 `subagent-driven-development` 和 `executing-plans` 都添加了 `using-git-worktrees` 作为必需技能。实现工作流现在明确要求在开始工作之前设置隔离 worktree，防止意外直接在 main 上工作。

**主分支保护软化以要求明确同意**

技能现在允许带用户明确同意的主分支工作。更灵活，同时仍确保用户意识到影响。

**简化安装验证**

从验证步骤中移除了 `/help` 命令检查和特定 slash 命令列表。技能主要通过描述你想做什么来调用，而不是运行特定命令。

**Codex: 在 bootstrap 中澄清子代理工具映射**

改进了 Codex 工具如何映射到 Claude Code 等价物用于子代理工作流的文档。

### 测试

- 添加了 subagent-driven-development 的 worktree 要求测试
- 添加了主分支红旗警告测试
- 修复了技能识别测试断言中的大小写敏感性

---

## v4.1.1 (2026-01-23)

### 修复

**OpenCode: 按官方文档标准化为 `plugins/` 目录** (#343)

OpenCode 官方文档使用 `~/.config/opencode/plugins/`（复数）。我们的文档之前使用 `plugin/`（单数）。虽然 OpenCode 接受两种形式，我们已标准化为官方惯例以避免混淆。

变更：
- 在 repo 结构中将 `.opencode/plugin/` 重命名为 `.opencode/plugins/`
- 更新所有安装文档（INSTALL.md, README.opencode.md）跨所有平台
- 更新测试脚本匹配

**OpenCode: 修复符号链接说明** (#339, #342)

- 在 `ln -s` 之前添加明确 `rm`（修复重新安装时的"file already exists"错误）
- 添加了 INSTALL.md 中缺失的技能符号链接步骤
- 从已弃用的 `use_skill`/`find_skills` 更新为原生 `skill` 工具引用

---

## v4.1.0 (2026-01-23)

### 破坏性变更

**OpenCode: 切换到原生技能系统**

Superpowers for OpenCode 现在使用 OpenCode 原生 `skill` 工具而不是自定义 `use_skill`/`find_skills` 工具。这是与 OpenCode 内置技能发现配合工作的更干净集成。

**需要迁移：** 技能必须符号链接到 `~/.config/opencode/skills/superpowers/`（见更新的安装文档）。

### 修复

**OpenCode: 修复会话启动时的 agent 重置** (#226)

之前使用 `session.prompt({ noReply: true })` 的 bootstrap 注入方法导致 OpenCode 在第一条消息时将选定的 agent 重置为"build"。现在使用 `experimental.chat.system.transform` hook，直接修改系统提示而无副作用。

**OpenCode: 修复 Windows 安装** (#232)

- 移除了对 `skills-core.js` 的依赖（当文件被复制而不是符号链接时消除损坏的相对导入）
- 为 cmd.exe、PowerShell 和 Git Bash 添加了全面的 Windows 安装文档
- 为每个平台文档了正确的 symlink vs junction 用法

**Claude Code: 修复 Claude Code 2.1.x 的 Windows hook 执行**

Claude Code 2.1.x 改变了 hooks 在 Windows 上执行的方式：它现在自动检测命令中的 `.sh` 文件并添加 `bash` 前缀。这破坏了 polyglot wrapper 模式，因为 `bash "run-hook.cmd" session-start.sh` 尝试将 .cmd 文件作为 bash 脚本执行。

修复：hooks.json 现在直接调用 session-start.sh。Claude Code 2.1.x 自动处理 bash 调用。还添加了 .gitattributes 以强制 shell 脚本使用 LF 行尾（修复 Windows checkout 上的 CRLF 问题）。

---

## v4.0.3 (2025-12-26)

### 改进

**为明确技能请求加强 using-superpowers 技能**

解决了 Claude 即使在用户明确按名称请求技能时也会跳过调用的失败模式（例如 "subagent-driven-development, please"）。Claude 会认为"我知道那是什么意思"并直接开始工作而不是加载技能。

变更：
- 更新"The Rule"说"调用相关或请求的技能"而不是"检查技能" - 强调主动调用而不是被动检查
- 添加了"在任何响应或动作之前" - 原始措辞只提到"响应"但 Claude 有时会先执行动作而不响应
- 添加了调用错误技能也没问题的安慰 - 减少犹豫
- 添加了新红旗："我知道那是什么意思" → 知道概念 ≠ 使用技能

**添加了明确技能请求测试**

`tests/explicit-skill-requests/` 中的新测试套件验证 Claude 在用户按名称请求时正确调用技能。包含单轮和多轮测试场景。

## v4.0.2 (2025-12-23)

### 修复

**Slash 命令现在只限用户**

为所有三个 slash 命令（`/brainstorm`、`/execute-plan`、`/write-plan`）添加了 `disable-model-invocation: true`。Claude 不能再通过 Skill tool 调用这些命令 — 它们只限于手动用户调用。

底层技能（`superpowers:brainstorming`、`superpowers:executing-plans`、`superpowers:writing-plans`）仍可供 Claude 自主调用。此更改防止混淆，当 Claude 调用一个只是重定向到技能的命令时。

## v4.0.1 (2025-12-23)

### 修复

**澄清如何在 Claude Code 中访问技能**

修复了一个混乱模式，Claude 会通过 Skill tool 调用技能，然后尝试单独 Read 技能文件。`using-superpowers` 技能现在明确声明 Skill tool 直接加载技能内容 — 无需读取文件。

- 在 `using-superpowers` 中添加了"How to Access Skills"部分
- 在指令中改"read the skill" → "invoke the skill"
- 更新 slash 命令使用完全限定技能名（例如 `superpowers:brainstorming`）

**在 receiving-code-review 中添加了 GitHub thread 回复指导** (h/t @ralphbean)

添加了关于在原始 thread 中回复 inline review comments 而不是作为顶级 PR comments 的说明。

**在 writing-skills 中添加了 automation-over-documentation 指导** (h/t @EthanJStark)

添加了机械约束应自动化而不是文档化的指导 — 将技能留给判断性决策。

## v4.0.0 (2025-12-17)

### 新功能

**子代理驱动开发中的两阶段代码审核**

子代理工作流现在在每个任务后使用两个分开的审核阶段：

1. **规格合规审核** - 挑剔的审核者验证实现完全匹配规格。捕获缺失需求和过度构建。不信任实现者的报告 — 读取实际代码。

2. **代码质量审核** - 只在规格合规通过后运行。审核整洁代码、测试覆盖率、可维护性。

这捕获代码写得好但不匹配请求内容的常见失败模式。审核是循环，不是一次性：如果审核者发现问题，实现者修复，然后审核者再检查。

其他子代理工作流改进：
- Controller 向 workers 提供完整任务文本（不是文件引用）
- Workers 可以在工作前和工作期间提问澄清
- 完成报告前的自审检查清单
- 开始时读取计划一次，提取到 TodoWrite

`skills/subagent-driven-development/` 中的新提示模板：
- `implementer-prompt.md` - 包含自审检查清单，鼓励提问
- `spec-reviewer-prompt.md` - 对需求的挑剔验证
- `code-quality-reviewer-prompt.md` - 标准代码审核

**调试技术与工具整合**

`systematic-debugging` 现在捆绑支持技术和工具：
- `root-cause-tracing.md` - 通过调用栈向后追踪 bug
- `defense-in-depth.md` - 在多个层次添加验证
- `condition-based-waiting.md` - 用条件轮询替换任意超时
- `find-polluter.sh` - 二分脚本找出哪个测试创建污染
- `condition-based-waiting-example.ts` - 来自真实调试会话的完整实现

**测试反模式参考**

`test-driven-development` 现在包含 `testing-anti-patterns.md`，涵盖：
- 测试 mock 行为而不是真实行为
- 向生产类添加仅测试方法
- 不理解依赖而 mock
- 不完整 mock 隐藏结构性假设

**技能测试基础设施**

三个验证技能行为的新测试框架：

`tests/skill-triggering/` - 验证技能从朴素提示触发而无明确命名。测试 6 个技能以确保描述足够。

`tests/claude-code/` - 使用 `claude -p` 进行无头测试的集成测试。通过会话 transcript（JSONL）分析验证技能使用。包含 `analyze-token-usage.py` 用于成本追踪。

`tests/subagent-driven-dev/` - 带两个完整测试项目的端到端工作流验证：
- `go-fractals/` - 带 Sierpinski/Mandelbrot 的 CLI 工具（10 个任务）
- `svelte-todo/` - 带 localStorage 和 Playwright 的 CRUD app（12 个任务）

### 主要变更

**DOT 流程图作为可执行规范**

使用 DOT/GraphViz 流程图作为权威流程定义重写了关键技能。文本变为支持内容。

**The Description Trap**（在 `writing-skills` 中文档）：发现当描述包含工作流摘要时技能描述会覆盖流程图内容。Claude 遵循短描述而不是读取详细流程图。修复：描述必须是触发专用（"Use when X"）而无流程细节。

**using-superpowers 中的技能优先级**

当多个技能适用时，流程技能（brainstorming, debugging）现在明确在实现技能之前。"构建 X"先触发 brainstorming，然后领域技能。

**brainstorming 触发加强**

描述改为命令式："任何创造性工作前你必须使用此技能 — 创建功能、构建组件、添加功能或修改行为。"

### 破坏性变更

**技能整合** - 六个独立技能合并：
- `root-cause-tracing`、`defense-in-depth`、`condition-based-waiting` → 捆绑在 `systematic-debugging/`
- `testing-skills-with-subagents` → 捆绑在 `writing-skills/`
- `testing-anti-patterns` → 捆绑在 `test-driven-development/`
- `sharing-skills` 已移除（过时）

### 其他改进

- **render-graphs.js** - 从技能提取 DOT 图并渲染为 SVG 的工具
- **using-superpowers 中的合理化表** - 可扫描格式，包含新条目："我需要更多上下文"、"让我先探索"、"这感觉 productive"
- **docs/testing.md** - 用 Claude Code 集成测试测试技能的指南

---

## v3.6.2 (2025-12-03)

### 修复

- **Linux 兼容性**：修复 polyglot hook wrapper（`run-hook.cmd`）使用 POSIX 兼容语法
  - 在第 16 行将 bash 专用的 `${BASH_SOURCE[0]:-$0}` 替换为标准 `$0`
  - 解决 Ubuntu/Debian 系统上 `/bin/sh` 是 dash 时出现的"Bad substitution"错误
  - 修复 #141

---

## v3.5.1 (2025-11-24)

### 变更

- **OpenCode Bootstrap 重构**：从 `chat.message` hook 切换到 `session.created` 事件用于 bootstrap 注入
  - Bootstrap 现在通过 `session.prompt()` 带 `noReply: true` 在会话创建时注入
  - 明确告诉模型 using-superpowers 已加载以防止冗余技能加载
  - 将 bootstrap 内容生成整合到共享 `getBootstrapContent()` helper
  - 更干净的单实现方法（移除了回退模式）

---

## v3.5.0 (2025-11-23)

### 添加

- **OpenCode 支持**：OpenCode.ai 的原生 JavaScript 插件
  - 自定义工具：`use_skill` 和 `find_skills`
  - 技能在上下文压缩时持久化的消息插入模式
  - 通过 chat.message hook 自动上下文注入
  - session.compacted 事件时自动重新注入
  - 三层技能优先级：project > personal > superpowers
  - 项目本地技能支持（`.opencode/skills/`）
  - 与 Codex 共享的核心模块（`lib/skills-core.js`）用于代码复用
  - 带适当隔离的自动测试套件（`tests/opencode/`）
  - 平台特定文档（`docs/README.opencode.md`, `docs/README.codex.md`）

### 变更

- **重构 Codex 实现**：现在使用共享 `lib/skills-core.js` ES 模块
  - 消除 Codex 和 OpenCode 之间的代码重复
  - 技能发现和解析的单一真相源
  - Codex 通过 Node.js interop 成功加载 ES 模块

- **改进文档**：重写 README 清晰解释问题/解决方案
  - 移除重复部分和冲突信息
  - 添加完整工作流描述（brainstorm → plan → execute → finish）
  - 简化平台安装说明
  - 强调技能检查协议而非自动激活声明

---

## v3.4.1 (2025-10-31)

### 改进

- 优化 superpowers bootstrap 以消除冗余技能执行。`using-superpowers` 技能内容现在直接在会话上下文中提供，明确指导只对其他技能使用 Skill tool。这减少了开销并防止混乱循环，agent 会手动执行 `using-superpowers` 尽管会话启动时已有内容。

## v3.4.0 (2025-10-30)

### 改进

- 简化 `brainstorming` 技能回归原始对话愿景。移除带正式检查清单的重型 6 阶段流程，改为自然对话：逐一提问，然后以 200-300 词部分呈现设计并验证。保留文档和实现移交功能。

## v3.3.1 (2025-10-28)

### 改进

- 更新 `brainstorming` 技能要求提问前自主侦察，鼓励推荐驱动的决策，并防止 agent 将优先级委派回人类。
- 应用 Strunk's "Elements of Style" 原则对 `brainstorming` 技能进行写作清晰性改进（删除多余词、将否定转为肯定形式、改进平行结构）。

### Bug 修复

- 澄清 `writing-skills` 指导指向正确的 agent 专用个人技能目录（Claude Code 用 `~/.claude/skills`，Codex 用 `~/.codex/skills`）。

## v3.3.0 (2025-10-28)

### 新功能

**实验性 Codex 支持**
- 添加带 bootstrap/use-skill/find-skills 命令的统一 `superpowers-codex` 脚本
- 跨平台 Node.js 实现（在 Windows、macOS、Linux 上工作）
- 命名空间技能：`superpowers:skill-name` 用于 superpowers 技能，`skill-name` 用于个人技能
- 个人技能在名称匹配时覆盖 superpowers 技能
- 清洁技能显示：显示名称/描述而无原始 frontmatter
- 有用上下文：为每个技能显示支持文件目录
- Codex 工具映射：TodoWrite→update_plan、subagents→手动回退等
- 带 Codex 专用最小 AGENTS.md 的 Bootstrap 集成用于自动启动
- 完整安装指南和 Codex 专用的 bootstrap 指示

**与 Claude Code 集成的关键区别：**
- 单一统一脚本而不是分开的工具
- Codex 专用等价物的工具替换系统
- 简化的子代理处理（手动工作而不是委派）
- 更新的术语："Superpowers skills" 而不是 "Core skills"

### 添加的文件
- `.codex/INSTALL.md` - Codex 用户安装指南
- `.codex/superpowers-bootstrap.md` - 带 Codex 适配的 Bootstrap 指示
- `.codex/superpowers-codex` - 带所有功能的统一 Node.js 可执行文件

**注意：** Codex 支持是实验性的。集成提供核心 superpowers 功能但可能需要根据用户反馈完善。

## v3.2.3 (2025-10-23)

### 改进

**更新 using-superpowers 技能使用 Skill tool 而不是 Read tool**
- 将技能调用指令从 Read tool 改为 Skill tool
- 更新描述："using Read tool" → "using Skill tool"
- 更新步骤 3："Use the Read tool" → "Use the Skill tool to read and run"
- 更新合理化列表："Read the current version" → "Run the current version"

Skill tool 是 Claude Code 中调用技能的正确机制。此更新修正 bootstrap 指示以引导 agent 使用正确的工具。

### 变更的文件
- 更新：`skills/using-superpowers/SKILL.md` - 将工具引用从 Read 改为 Skill

## v3.2.2 (2025-10-21)

### 改进

**针对 agent 合理化加强 using-superpowers 技能**
- 添加带关于强制技能检查绝对语言的 EXTREMELY-IMPORTANT 块
  - "即使有 1% 可能性技能适用，你必须读取它"
  - "你没有选择。你不能合理化绕过。"
- 添加 MANDATORY FIRST RESPONSE PROTOCOL 检查清单
  - agent 在任何响应前必须完成的 5 步过程
  - 明确"无此响应 = 失败"后果
- 添加 Common Rationalizations 部分，包含 8 个特定规避模式
  - "这只是简单问题" → 错
  - "我可以快速检查文件" → 错
  - "让我先收集信息" → 错
  - 还有观察到的 agent 行为中的 5 个更多模式

这些更改解决观察到的 agent 行为，他们尽管有清晰指令仍合理化绕过技能使用。强力语言和预先反驳旨在使不合规更难。

### 变更的文件
- 更新：`skills/using-superpowers/SKILL.md` - 添加三层强制以防止技能跳过合理化

## v3.2.1 (2025-10-20)

### 新功能

**代码审核 agent 现包含在插件中**
- 向插件的 `agents/` 目录添加了 `superpowers:code-reviewer` agent
- Agent 提供对计划和编码标准的系统性代码审核
- 之前需要用户有个人 agent 配置
- 所有技能引用更新为使用命名空间 `superpowers:code-reviewer`
- 修复 #55

### 变更的文件
- 新增：`agents/code-reviewer.md` - 带审核检查清单和输出格式的 Agent 定义
- 更新：`skills/requesting-code-review/SKILL.md` - 对 `superpowers:code-reviewer` 的引用
- 更新：`skills/subagent-driven-development/SKILL.md` - 对 `superpowers:code-reviewer` 的引用

## v3.2.0 (2025-10-18)

### 新功能

**Brainstorming 工作流中的设计文档**
- 向 brainstorming 技能添加了 Phase 4: Design Documentation
- 设计文档现在在实现前写入 `docs/plans/YYYY-MM-DD-<topic>-design.md`
- 恢复在技能转换期间丢失的原始 brainstorming 命令功能
- 文档在 worktree 设置和实现计划前写入
- 用子代理测试以验证时间压力下的合规

### 破坏性变更

**技能引用命名空间标准化**
- 所有内部技能引用现在使用 `superpowers:` 命名空间前缀
- 更新格式：`superpowers:test-driven-development`（之前只是 `test-driven-development`）
- 影响 REQUIRED SUB-SKILL、RECOMMENDED SUB-SKILL 和 REQUIRED BACKGROUND 引用
- 与使用 Skill tool 调用技能的方式对齐
- 更新文件：brainstorming、executing-plans、subagent-driven-development、systematic-debugging、testing-skills-with-subagents、writing-plans、writing-skills

### 改进

**设计与实现计划命名**
- 设计文档使用 `-design.md` 后缀防止文件名冲突
- 实现计划继续使用现有 `YYYY-MM-DD-<feature-name>.md` 格式
- 都存储在 `docs/plans/` 目录，命名清晰区分

## v3.1.1 (2025-10-17)

### Bug 修复

- **修复 README 中的命令语法** (#44) - 更新所有命令引用使用正确命名空间语法（`/superpowers:brainstorm` 而不是 `/brainstorm`）。插件提供的命令由 Claude Code 自动命名空间化以避免插件间冲突。

## v3.1.0 (2025-10-17)

### 破坏性变更

**技能名称标准化为小写**
- 所有技能 frontmatter `name:` 字段现在使用匹配目录名的小写 kebab-case
- 示例：`brainstorming`、`test-driven-development`、`using-git-worktrees`
- 所有技能公告和交叉引用更新为小写格式
- 这确保目录名、frontmatter 和文档中的命名一致

### 新功能

**增强 brainstorming 技能**
- 添加显示阶段、活动和工具使用的快速参考表
- 添加可复制的工作流检查清单用于追踪进度
- 添加何时回访早期阶段的决策流程图
- 添加带具体示例的全面 AskUserQuestion 工具指导
- 添加"问题模式"部分解释何时使用结构化 vs 开放式问题
- 重构关键原则为可扫描表

**Anthropic 最佳实践整合**
- 添加 `skills/writing-skills/anthropic-best-practices.md` - 官方 Anthropic 技能编写指南
- 在 writing-skills SKILL.md 中引用以获取全面指导
- 提供渐进披露、工作流和评估模式

### 改进

**技能交叉引用清晰性**
- 所有技能引用现在使用明确要求标记：
  - `**REQUIRED BACKGROUND:**` - 必须理解的前提
  - `**REQUIRED SUB-SKILL:**` - 工作流中必须使用的技能
  - `**Complementary skills:**` - 可选但有帮助的相关技能
- 移除旧路径格式（`skills/collaboration/X` → 只是 `X`）
- 更新集成部分带分类关系（必需 vs 补充）
- 更新交叉引用文档带最佳实践

**与 Anthropic 最佳实践对齐**
- 修复描述语法和语态（完全第三人称）
- 添加用于扫描的快速参考表
- 添加 Claude 可以复制和追踪的工作流检查清单
- 对不明显决策点适当使用流程图
- 改进可扫描表格式
- 所有技能低于 500 行推荐

### Bug 修复

- **重新添加缺失的命令重定向** - 恢复在 v3.0 迁移中意外删除的 `commands/brainstorm.md` 和 `commands/write-plan.md`
- 修复 `defense-in-depth` 名称不匹配（曾是 `Defense-in-Depth-Validation`）
- 修复 `receiving-code-review` 名称不匹配（曾是 `Code-Review-Reception`）
- 修复 `commands/brainstorm.md` 引用正确的技能名
- 移除对不存在相关技能的引用

### 文档

**writing-skills 改进**
- 更新交叉引用指导带明确要求标记
- 添加 Anthropic 官方最佳实践引用
- 改进示例显示正确技能引用格式

## v3.0.1 (2025-10-16)

### 变更

我们现在使用 Anthropic 的第一方技能系统！

## v2.0.2 (2025-10-12)

### Bug 修复

- **修复本地技能 repo 领先于 upstream 时的错误警告** - 初始化脚本在本地 repo 有 commits 领先于 upstream 时错误警告"New skills available from upstream"。逻辑现在正确区分三种 git 状态：本地落后（应更新）、本地领先（无警告）、分歧（应警告）。

## v2.0.1 (2025-10-12)

### Bug 修复

- **修复插件上下文中的 session-start hook 执行** (#8, PR #9) - Hook 在"Plugin hook error"下静默失败，阻止技能上下文加载。修复：
  - 当 BASH_SOURCE 在 Claude Code 执行上下文中未绑定时使用 `${BASH_SOURCE[0]:-$0}` 回退
  - 添加 `|| true` 以优雅处理空 grep 结果当过滤状态标志

---

# Superpowers v2.0.0 发布说明

## 概述

Superpowers v2.0 通过重大架构转变使技能更易访问、可维护和社区驱动。

头条变更 是 **技能仓库分离**：所有技能、脚本和文档已从插件移到专用仓库（[obra/superpowers-skills](https://github.com/obra/superpowers-skills））。这将 superpowers 从单体插件转变为管理技能仓库本地克隆的轻量 shim。技能在会话启动时自动更新。用户 fork 并通过标准 git workflow 贡献改进。技能库与插件独立版本化。

除了基础设施，此版本添加了九个聚焦问题解决、研究和架构的新技能。我们用命令式语调和更清晰结构重写了核心 **using-skills** 文档，使 Claude 更易理解何时和如何使用技能。**find-skills** 现输出可直接粘贴到 Read tool 的路径，消除技能发现工作流中的摩擦。

用户体验无缝操作：插件自动处理克隆、fork 和更新。贡献者发现新架构使改进和共享技能变得 trivial。此版本为技能作为社区资源快速演进奠定了基础。

## 破坏性变更

### 技能仓库分离

**最大变更：** 技能不再存在于插件中。它们已移到 [obra/superpowers-skills](https://github.com/obra/superpowers-skills) 的单独仓库。

**这意味着：**

- **首次安装：** 插件自动克隆技能到 `~/.config/superpowers/skills/`
- **Forking：** 设置期间，如果安装了 `gh`，你将被提议 fork 技能 repo 的选项
- **更新：** 技能在会话启动时自动更新（尽可能快进）