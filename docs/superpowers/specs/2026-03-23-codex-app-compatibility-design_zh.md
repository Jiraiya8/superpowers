# Codex App 兼容性：Worktree 和完成技能适配

使 superpowers 技能在 Codex App 的沙盒 worktree 环境中工作，同时不破坏现有的 Claude Code 或 Codex CLI 行为。

**工单：** PRI-823

## 动机

Codex App 在其管理的 git worktree 内运行代理 — 分离的 HEAD、位于 `$CODEX_HOME/worktrees/` 下、带有 Seatbelt 沙盒阻止 `git checkout -b`、`git push` 和网络访问。三个 superpowers 技能假设不受限制的 git 访问：`using-git-worktrees` 创建带有命名分支的手动 worktree，`finishing-a-development-branch` 通过分支名合并/推送/创建 PR，`subagent-driven-development` 需要这两者。

Codex CLI（开源终端工具）不存在此冲突 — 它没有内置的 worktree 管理。我们的手动 worktree 方法填补了那里的隔离缺口。问题仅存在于 Codex App。

## 实验发现

于 2026-03-23 在 Codex App 中测试：

| 操作 | workspace-write 沙盒 | Full access 沙盒 |
|---|---|---|
| `git add` | 正常 | 正常 |
| `git commit` | 正常 | 正常 |
| `git checkout -b` | **被阻止**（无法写入 `.git/refs/heads/`） | 正常 |
| `git push` | **被阻止**（网络 + `.git/refs/remotes/`） | 正常 |
| `gh pr create` | **被阻止**（网络） | 正常 |
| `git status/diff/log` | 正常 | 正常 |

其他发现：
- `spawn_agent` 子代理 **共享** 父线程的文件系统（通过标记文件测试确认）
- "Create branch" 按钮在 App 头部显示，无论 worktree 从哪个分支启动
- App 的原生完成流程：Create branch → Commit modal → Commit and push / Commit and create PR
- `network_access = true` 配置在 macOS 上静默失效（问题 #10390）

## 设计：只读环境检测

三个只读 git 命令检测环境，无副作用：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

得出两个信号：

- **IN_LINKED_WORKTREE：** `GIT_DIR != GIT_COMMON` — 代理在由其他东西创建的 worktree 中（Codex App、Claude Code Agent 工具、之前的技能运行，或用户）
- **ON_DETACHED_HEAD：** `BRANCH` 为空 — 不存在命名分支

为什么用 `git-dir != git-common-dir` 而不是检查 `show-toplevel`：
- 在普通仓库中，两者解析到同一个 `.git` 目录
- 在链接的 worktree 中，`git-dir` 是 `.git/worktrees/<name>` 而 `git-common-dir` 是 `.git`
- 在子模块中，两者相等 — 避免 `show-toplevel` 会产生的误报
- 通过 `cd && pwd -P` 解析处理相对路径问题（`git-common-dir` 在普通仓库返回相对 `.git` 但在 worktree 返回绝对路径）和符号链接（macOS `/tmp` → `/private/tmp`）

### 决策矩阵

| 链接 Worktree? | 分离 HEAD? | 环境 | 动作 |
|---|---|---|---|
| 否 | 否 | Claude Code / Codex CLI / 普通 git | 完整技能行为（不变） |
| 是 | 是 | Codex App worktree（workspace-write） | 跳过 worktree 创建；完成时移交 payload |
| 是 | 否 | Codex App（Full access）或手动 worktree | 跳过 worktree 创建；完整完成流程 |
| 否 | 是 | 异常（手动分离 HEAD） | 正常创建 worktree；完成时警告 |

## 变更

### 1. `using-git-worktrees/SKILL.md` — 添加 Step 0（约 12 行）

在 "Overview" 和 "Directory Selection Process" 之间新增部分：

**Step 0: 检查是否已在隔离工作空间**

运行检测命令。如果 `GIT_DIR != GIT_COMMON`，完全跳过 worktree 创建。而是：
1. 跳到 Creation Steps 下的 "Run Project Setup" 子部分 — `npm install` 等是幂等的，值得运行以确保安全
2. 然后 "Verify Clean Baseline" — 运行测试
3. 报告分支状态：
   - 在分支上："Already in an isolated workspace at `<path>` on branch `<name>`。Tests passing。Ready to implement。"
   - 分离 HEAD："Already in an isolated workspace at `<path>`（detached HEAD，externally managed）。Tests passing。Note: branch creation needed at finish time。Ready to implement。"

如果 `GIT_DIR == GIT_COMMON`，继续完整的 worktree 创建流程（不变）。

安全验证（.gitignore 检查）在 Step 0 触发时跳过 — 与外部创建的 worktree 无关。

更新 Integration 部分的 "Called by" 条目。将每个描述从上下文特定的文本改为："Ensures isolated workspace（creates one or verifies existing）"。例如，`subagent-driven-development` 条目从 "REQUIRED: Set up isolated workspace before starting" 改为 "REQUIRED: Ensures isolated workspace（creates one or verifies existing）"。

**沙盒回退：** 如果 `GIT_DIR == GIT_COMMON` 且技能继续 Creation Steps，但 `git worktree add -b` 因权限错误失败（例如 Seatbelt 沙盒拒绝），将其视为延迟检测到的受限环境。回退到 Step 0 "already in workspace" 行为 — 跳过创建，在当前目录运行设置和基线测试，相应报告。

在 Step 0 报告后，停止。不要继续到 Directory Selection 或 Creation Steps。

**其他内容不变：** Directory Selection、Safety Verification、Creation Steps、Project Setup、Baseline Tests、Quick Reference、Common Mistakes、Red Flags。

### 2. `finishing-a-development-branch/SKILL.md` — 添加 Step 1.5 + 清理守卫（约 20 行）

**Step 1.5: 检测环境**（在 Step 1 "Verify Tests" 之后，Step 2 "Determine Base Branch" 之前）

运行检测命令。三条路径：

- **Path A** 完全跳过 Step 2 和 Step 3（不需要基线分支或选项）。
- **Path B 和 C** 正常继续 Step 2（Determine Base Branch）和 Step 3（Present Options）。

**Path A — 外部管理的 worktree + 分离 HEAD**（`GIT_DIR != GIT_COMMON` 且 `BRANCH` 为空）：

首先，确保所有工作已暂存并提交（`git add` + `git commit`）。Codex App 的完成控件作用于已提交的工作。

然后向用户展示（不要展示 4 选项菜单）：

```
Implementation complete。All tests passing。
Current HEAD: <full-commit-sha>

This workspace is externally managed（detached HEAD）。
I cannot create branches，push，or open PRs from here。

⚠ These commits are on a detached HEAD。If you do not create a branch，
they may be lost when this workspace is cleaned up。

If your host application provides these controls:
- "Create branch" — to name a branch，then commit/push/PR
- "Hand off to local" — to move changes to your local checkout

Suggested branch name: <ticket-id/short-description>
Suggested commit message: <summary-of-work>
```

分支名推导：如果有 ticket ID 则使用（例如 `pri-823/codex-compat`），否则 slugify 计划标题的前 5 个词，否则省略建议。避免在分支名中包含敏感内容（漏洞描述、客户名）。

跳到 Step 5（清理对外部管理的 worktree 是空操作）。

**Path B — 外部管理的 worktree + 命名分支**（`GIT_DIR != GIT_COMMON` 且 `BRANCH` 存在）：

正常展示 4 选项菜单。（Step 5 清理守卫会独立重新检测外部管理的状态。）

**Path C — 正常环境**（`GIT_DIR == GIT_COMMON`）：

今天一样正常展示 4 选项菜单（不变）。

**Step 5 清理守卫：**

在清理时重新运行 `GIT_DIR` vs `GIT_COMMON` 检测（不要依赖之前技能输出 — 完成技能可能在不同会话中运行）。如果 `GIT_DIR != GIT_COMMON`，跳过 `git worktree remove` — 主环境拥有此工作空间。

否则，照今天检查和移除。注意：现有 Step 5 文本说 "For Options 1, 2, 4" 但 Quick Reference 表格和 Common Mistakes 部分说 "Options 1 & 4 only。" 新守卫添加在现有逻辑之前，不改变哪些选项触发清理。

**其他内容不变：** Options 1-4 逻辑、Quick Reference、Common Mistakes、Red Flags。

### 3. `subagent-driven-development/SKILL.md` 和 `executing-plans/SKILL.md` — 各 1 行编辑

两个技能有相同的 Integration 部分行。从：
```
- superpowers:using-git-worktrees - REQUIRED: Set up isolated workspace before starting
```
改为：
```
- superpowers:using-git-worktrees - REQUIRED: Ensures isolated workspace（creates one or verifies existing）
```

**其他内容不变：** Dispatch/review loop、prompt templates、model selection、status handling、red flags。

### 4. `codex-tools.md` — 添加环境检测文档（约 15 行）

末尾两个新部分：

**环境检测：**

```markdown
## Environment Detection

Skills that create worktrees or finish branches should detect their
environment with read-only git commands before proceeding:

\```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
\```

- `GIT_DIR != GIT_COMMON` → already in a linked worktree（skip creation）
- `BRANCH` empty → detached HEAD（cannot branch/push/PR from sandbox）

See `using-git-worktrees` Step 0 and `finishing-a-development-branch`
Step 1.5 for how each skill uses these signals。
```

**Codex App 完成：**

```markdown
## Codex App Finishing

When the sandbox blocks branch/push operations（detached HEAD in an
externally managed worktree），the agent commits all work and informs
the user to use the App's native controls:

- **"Create branch"** — names the branch，then commit/push/PR via App UI
- **"Hand off to local"** — transfers work to the user's local checkout

The agent can still run tests，stage files，and output suggested branch
names，commit messages，and PR descriptions for the user to copy。
```

## 不变的内容

- `implementer-prompt.md`、`spec-reviewer-prompt.md`、`code-quality-reviewer-prompt.md` — 子代理 prompt 不变
- `executing-plans/SKILL.md` — 只有 1 行 Integration 描述变更（与 `subagent-driven-development` 相同）；所有运行时行为不变
- `dispatching-parallel-agents/SKILL.md` — 无 worktree 或完成操作
- `.codex/INSTALL.md` — 安装流程不变
- 4 选项完成菜单 — 对 Claude Code 和 Codex CLI 完全保留
- 完整 worktree 创建流程 — 对非 worktree 环境完全保留
- 子代理 dispatch/review/iterate 循环 — 不变（文件系统共享已确认）

## 范围摘要

| 文件 | 变更 |
|---|---|
| `skills/using-git-worktrees/SKILL.md` | +12 行（Step 0） |
| `skills/finishing-a-development-branch/SKILL.md` | +20 行（Step 1.5 + 清理守卫） |
| `skills/subagent-driven-development/SKILL.md` | 1 行编辑 |
| `skills/executing-plans/SKILL.md` | 1 行编辑 |
| `skills/using-superpowers/references/codex-tools.md` | +15 行 |

约 50 行添加/变更，分布在 5 个文件。零新文件。零破坏性变更。

## 未来考虑

如果有第三个技能需要相同的检测模式，将其提取到共享的 `references/environment-detection.md` 文件（Approach B）。现在不需要 — 只有 2 个技能使用它。

## 测试计划

### 自动化（实现后在 Claude Code 中运行）

1. 普通仓库检测 — 断言 IN_LINKED_WORKTREE=false
2. 链接 worktree 检测 — `git worktree add` 测试 worktree，断言 IN_LINKED_WORKTREE=true
3. 分离 HEAD 检测 — `git checkout --detach`，断言 ON_DETACHED_HEAD=true
4. 完成技能移交输出 — 在受限环境验证移交消息（不是 4 选项菜单）
5. **Step 5 清理守卫** — 创建链接 worktree（`git worktree add /tmp/test-cleanup -b test-cleanup`），`cd` 进入，运行 Step 5 清理检测（`GIT_DIR` vs `GIT_COMMON`），断言它不会调用 `git worktree remove`。然后 `cd` 回主仓库，运行相同检测，断言它会调用 `git worktree remove`。之后清理测试 worktree。

### 手动 Codex App 测试（5 个测试）

1. Worktree 线程检测（workspace-write） — 验证 GIT_DIR != GIT_COMMON，空分支
2. Worktree 线程检测（Full access） — 相同检测，不同沙盒行为
3. 完成技能移交格式 — 验证代理输出移交 payload，不是 4 选项菜单
4. 完整生命周期 — 检测 → commit → 完成检测 → 正确行为 → 清理
5. **Local 线程中的沙盒回退** — 启动 Codex App **Local thread**（workspace-write 沙盒）。Prompt："Use the superpowers skill `using-git-worktrees` to set up an isolated workspace for implementing a small change。" 预检查：`git checkout -b test-sandbox-check` 应因 `Operation not permitted` 失败。预期：技能检测到 `GIT_DIR == GIT_COMMON`（普通仓库），尝试 `git worktree add -b`，遇到 Seatbelt 拒绝，回退到 Step 0 "already in workspace" 行为 — 运行设置、基线测试，从当前目录报告就绪。通过：代理优雅恢复，无晦涩错误消息。失败：代理打印原始 Seatbelt 错误、重试，或放弃并输出混乱内容。

### 回归

- 现有 Claude Code skill-triggering 测试仍通过
- 现有 subagent-driven-development 集成测试仍通过
- 普通 Claude Code 会话：完整 worktree 创建 + 4 选项完成仍工作