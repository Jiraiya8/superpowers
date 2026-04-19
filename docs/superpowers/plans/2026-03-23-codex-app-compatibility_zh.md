# Codex App 兼容性实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用 checkbox（`- [ ]`）语法进行追踪。

**目标：** 使 `using-git-worktrees`、`finishing-a-development-branch` 和相关技能在 Codex App 的沙盒 worktree 环境中工作，不破坏现有行为。

**架构：** 在两个技能开始时进行只读环境检测（`git-dir` vs `git-common-dir`）。如果已在链接 worktree，跳过创建。如果是分离 HEAD，输出移交 payload 而不是 4 选项菜单。沙盒回退捕获 worktree 创建时的权限错误。

**技术栈：** Git、Markdown（技能文件是说明文档，不是可执行代码）

**规范：** `docs/superpowers/specs/2026-03-23-codex-app-compatibility-design.md`

---

## 文件结构

| 文件 | 职责 | 动作 |
|---|---|---|
| `skills/using-git-worktrees/SKILL.md` | Worktree 创建 + 隔离 | 添加 Step 0 检测 + 沙盒回退 |
| `skills/finishing-a-development-branch/SKILL.md` | Branch 完成工作流 | 添加 Step 1.5 检测 + 清理守卫 |
| `skills/subagent-driven-development/SKILL.md` | 带子代理的计划执行 | 更新 Integration 描述 |
| `skills/executing-plans/SKILL.md` | 内联计划执行 | 更新 Integration 描述 |
| `skills/using-superpowers/references/codex-tools.md` | Codex 平台参考 | 添加检测 + 完成文档 |

---

### Task 1: 向 `using-git-worktrees` 添加 Step 0

**文件：**
- 修改：`skills/using-git-worktrees/SKILL.md:14-15`（在 Overview 后、Directory Selection Process 前插入）

- [ ] **Step 1: 读取当前技能文件**

完整读取 `skills/using-git-worktrees/SKILL.md`。确定精确插入点：在 "Announce at start" 行（line 14）后、"## Directory Selection Process"（line 16）前。

- [ ] **Step 2: 插入 Step 0 部分**

在 Overview 部分和 "## Directory Selection Process" 之间插入：

```markdown
## Step 0: 检查是否已在隔离工作空间中

在创建 worktree 之前，检查是否已存在：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**如果 `GIT_DIR` 与 `GIT_COMMON` 不同：** 你已在一个链接的 worktree 中（由 Codex App、Claude Code 的 Agent 工具、之前的技能运行或用户创建）。不要创建另一个 worktree。而是：

1. 运行项目设置（如下面"运行项目设置"自动检测包管理器）
2. 验证干净基线（如下面"验证干净基线"运行测试）
3. 报告分支状态：
   - 在分支上："已在 `<path>` 的隔离工作空间中，分支 `<name>`。测试通过。准备实现。"
   - 分离 HEAD："已在 `<path>` 的隔离工作空间中（分离 HEAD，外部管理）。测试通过。注意：完成时需要创建分支。准备实现。"

报告后，停止。不要继续到目录选择或创建步骤。

**如果 `GIT_DIR` 等于 `GIT_COMMON`：** 继续下面完整的 worktree 创建流程。

**沙箱回退：** 如果你继续到创建步骤但 `git worktree add -b` 因权限错误失败（例如，"Operation not permitted"），将此视为后期检测的受限环境。回退到上面的行为 — 在当前目录运行设置和基线测试，相应报告，并停止。
```

- [ ] **Step 3: 验证插入**

再次读取文件。确认：
- Step 0 出现在 Overview 和 Directory Selection Process 之间
- 文件其余部分（Directory Selection、Safety Verification、Creation Steps 等）不变
- 无重复部分或损坏的 markdown

- [ ] **Step 4: 提交**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "feat(using-git-worktrees): add Step 0 environment detection（PRI-823）

Skip worktree creation when already in a linked worktree。Includes
sandbox fallback for permission errors on git worktree add。"
```

---

### Task 2: 更新 `using-git-worktrees` Integration 部分

**文件：**
- 修改：`skills/using-git-worktrees/SKILL.md:211-215`（Integration > Called by）

- [ ] **Step 1: 更新三个 "Called by" 条目**

将 lines 212-214 从：

```markdown
- **brainstorming**（Phase 4） - REQUIRED when design is approved and implementation follows
- **subagent-driven-development** - REQUIRED before executing any tasks
- **executing-plans** - REQUIRED before executing any tasks
```

改为：

```markdown
- **brainstorming** - REQUIRED: Ensures isolated workspace（creates one or verifies existing）
- **subagent-driven-development** - REQUIRED: Ensures isolated workspace（creates one or verifies existing）
- **executing-plans** - REQUIRED: Ensures isolated workspace（creates one or verifies existing）
```

- [ ] **Step 2: 验证 Integration 部分**

读取 Integration 部分。确认所有三个条目已更新、"Pairs with" 不变。

- [ ] **Step 3: 提交**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "docs(using-git-worktrees): update Integration descriptions（PRI-823）

Clarify that skill ensures a workspace exists，not that it always creates one。"
```

---

### Task 3: 向 `finishing-a-development-branch` 添加 Step 1.5

**文件：**
- 修改：`skills/finishing-a-development-branch/SKILL.md:38`（在 Step 1 后、Step 2 前插入）

- [ ] **Step 1: 读取当前技能文件**

完整读取 `skills/finishing-a-development-branch/SKILL.md`。确定精确插入点：在 "**If tests pass:** Continue to Step 2。"（line 38）后、"### Step 2: Determine Base Branch"（line 40）前。

- [ ] **Step 2: 插入 Step 1.5 部分**

在 Step 1 和 Step 2 之间插入：

```markdown
### Step 1.5: 检测环境

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**路径 A — `GIT_DIR` 与 `GIT_COMMON` 不同 且 `BRANCH` 为空（外部管理的 worktree，分离 HEAD）:**

首先，确保所有工作已暂存并提交（`git add` + `git commit`）。

然后向用户呈现此内容（不要呈现 4 选项菜单）:

```
实现完成。所有测试通过。
当前 HEAD: <完整提交 sha>

此工作空间是外部管理的（分离 HEAD）。
我无法从这里创建分支、推送或打开 PR。

⚠ 这些提交位于分离 HEAD 上。如果你不创建分支，
当此工作空间被清理时它们可能会丢失。

如果你的宿主应用提供这些控件：
- "创建分支" — 命名分支，然后提交/推送/创建 PR
- "移交到本地" — 将更改移动到你的本地检出

建议的分支名：<ticket-id/简短描述>
建议的提交消息：<工作摘要>
```

分支名：如果有 ticket ID 则使用（例如 `pri-823/codex-compat`），否则将计划标题的前 5 个词转为 slug，否则省略。避免在分支名中使用敏感内容。

跳到 Step 5（清理是无操作 — 见下面的守卫）。

**路径 B — `GIT_DIR` 与 `GIT_COMMON` 不同 且 `BRANCH` 存在（外部管理的 worktree，命名分支）:**

继续到 Step 2 并正常呈现 4 选项菜单。

**路径 C — `GIT_DIR` 等于 `GIT_COMMON`（正常环境）:**

继续到 Step 2 并正常呈现 4 选项菜单。
```

- [ ] **Step 3: 验证插入**

再次读取文件。确认：
- Step 1.5 出现在 Step 1 和 Step 2 之间
- Steps 2-5 不变
- Path A 移交包含 commit SHA 和数据丢失警告
- Paths B 和 C 正常继续到 Step 2

- [ ] **Step 4: 提交**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat(finishing-a-development-branch): add Step 1.5 environment detection（PRI-823）

Detect externally managed worktrees with detached HEAD and emit handoff
payload instead of 4-option menu。Includes commit SHA and data loss warning。"
```

---

### Task 4: 向 `finishing-a-development-branch` 添加 Step 5 清理守卫

**文件：**
- 修改：`skills/finishing-a-development-branch/SKILL.md`（Step 5: Cleanup Worktree — 按部分标题查找，Task 3 后行号已偏移）

- [ ] **Step 1: 读取当前 Step 5 部分**

在 `skills/finishing-a-development-branch/SKILL.md` 中找到 "### Step 5: Cleanup Worktree" 部分（Task 3 插入后行号已偏移）。当前 Step 5 为：

```markdown
### Step 5: 清理 Worktree

**对于选项 1、2、4：**

检查是否在 worktree 中：
```bash
git worktree list | grep $(git branch --show-current)
```

如果是：
```bash
git worktree remove <worktree-path>
```

**对于选项 3：** 保留 worktree。
```

- [ ] **Step 2: 在现有逻辑前添加清理守卫**

替换 Step 5 部分：

```markdown
### Step 5: 清理 Worktree

**首先，检查 worktree 是否为外部管理：**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

如果 `GIT_DIR` 与 `GIT_COMMON` 不同：跳过 worktree 移除 — 宿主环境拥有此工作空间。

**否则，对于选项 1 和 4：**

检查是否在 worktree 中：
```bash
git worktree list | grep $(git branch --show-current)
```

如果是：
```bash
git worktree remove <worktree-path>
```

**对于选项 3：** 保留 worktree。
```

注意：原文本说 "For Options 1，2，4" 但 Quick Reference 表格和 Common Mistakes 部分说 "Options 1 & 4 only。" 此编辑使 Step 5 与那些部分对齐。

- [ ] **Step 3: 验证替换**

读取 Step 5。确认：
- 清理守卫（重新检测）首先出现
- 非外部管理 worktree 的现有移除逻辑保留
- "Options 1 and 4"（不是 "1，2，4"）匹配 Quick Reference 和 Common Mistakes

- [ ] **Step 4: 提交**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat(finishing-a-development-branch): add Step 5 cleanup guard（PRI-823）

Re-detect externally managed worktree at cleanup time and skip removal。
Also fixes pre-existing inconsistency: cleanup now correctly says
Options 1 and 4 only，matching Quick Reference and Common Mistakes。"
```

---

### Task 5: 更新 `subagent-driven-development` 和 `executing-plans` 的 Integration 行

**文件：**
- 修改：`skills/subagent-driven-development/SKILL.md:268`
- 修改：`skills/executing-plans/SKILL.md:68`

- [ ] **Step 1: 更新 `subagent-driven-development`**

将 line 268 从：
```
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```
改为：
```
- **superpowers:using-git-worktrees** - REQUIRED: Ensures isolated workspace（creates one or verifies existing）
```

- [ ] **Step 2: 更新 `executing-plans`**

将 line 68 从：
```
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```
改为：
```
- **superpowers:using-git-worktrees** - REQUIRED: Ensures isolated workspace（creates one or verifies existing）
```

- [ ] **Step 3: 验证两个文件**

读取 `skills/subagent-driven-development/SKILL.md` line 268 和 `skills/executing-plans/SKILL.md` line 68。确认两者都说 "Ensures isolated workspace（creates one or verifies existing）"。

- [ ] **Step 4: 提交**

```bash
git add skills/subagent-driven-development/SKILL.md skills/executing-plans/SKILL.md
git commit -m "docs(sdd，executing-plans): update worktree Integration descriptions（PRI-823）

Clarify that using-git-worktrees ensures a workspace exists rather than
always creating one。"
```

---

### Task 6: 向 `codex-tools.md` 添加环境检测文档

**文件：**
- 修改：`skills/using-superpowers/references/codex-tools.md:25`（末尾追加）

- [ ] **Step 1: 读取当前文件**

完整读取 `skills/using-superpowers/references/codex-tools.md`。确认它在 multi_agent 部分后 line 25-26 结束。

- [ ] **Step 2: 追加两个新部分**

在文件末尾添加：

```markdown

## 环境检测

创建 worktrees 或完成分支的技能应在继续之前使用只读 git 命令检测其环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → 已在链接的 worktree 中（跳过创建）
- `BRANCH` 空 → 分离 HEAD（无法从沙箱分支/推送/创建 PR）

参见 `using-git-worktrees` Step 0 和 `finishing-a-development-branch`
Step 1.5 了解每个技能如何使用这些信号。

## Codex App 完成

当沙箱阻止分支/推送操作（在外部管理的 worktree 中的分离 HEAD）时，
代理提交所有工作并通知用户使用 App 的原生控件：

- **"创建分支"** — 命名分支，然后通过 App UI 提交/推送/创建 PR
- **"移交到本地"** — 将工作转移到用户的本地检出

代理仍可运行测试、暂存文件，并输出建议的分支名、提交消息和 PR 描述供用户复制。
names，commit messages，and PR descriptions for the user to copy。
```

- [ ] **Step 3: 验证添加**

读取完整文件。确认：
- 两个新部分出现在现有内容后
- Bash 代码块正确渲染（未转义）
- Step 0 和 Step 1.5 的交叉引用存在

- [ ] **Step 4: 提交**

```bash
git add skills/using-superpowers/references/codex-tools.md
git commit -m "docs(codex-tools): add environment detection and App finishing docs（PRI-823）

Document the git-dir vs git-common-dir detection pattern and the Codex
App's native finishing flow for skills that need to adapt。"
```

---

### Task 7: 自动化测试 — 环境检测

**文件：**
- 创建：`tests/codex-app-compat/test-environment-detection.sh`

- [ ] **Step 1: 创建测试目录**

```bash
mkdir -p tests/codex-app-compat
```

- [ ] **Step 2: 编写检测测试脚本**

创建 `tests/codex-app-compat/test-environment-detection.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

# Test environment detection logic from PRI-823
# Tests the git-dir vs git-common-dir comparison used by
# using-git-worktrees Step 0 and finishing-a-development-branch Step 1.5

PASS=0
FAIL=0
TEMP_DIR=$(mktemp -d)
trap "rm -rf $TEMP_DIR" EXIT

log_pass() { echo "  PASS: $1"; PASS=$((PASS + 1)); }
log_fail() { echo "  FAIL: $1"; FAIL=$((FAIL + 1)); }

# Helper: run detection and return "linked" or "normal"
detect_worktree() {
  local git_dir git_common
  git_dir=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
  git_common=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
  if [ "$git_dir" != "$git_common" ]; then
    echo "linked"
  else
    echo "normal"
  fi
}

echo "=== Test 1: Normal repo detection ==="
cd "$TEMP_DIR"
git init test-repo > /dev/null 2>&1
cd test-repo
git commit --allow-empty -m "init" > /dev/null 2>&1
result=$(detect_worktree)
if [ "$result" = "normal" ]; then
  log_pass "Normal repo detected as normal"
else
  log_fail "Normal repo detected as '$result'（expected 'normal'）"
fi

echo "=== Test 2: Linked worktree detection ==="
git worktree add "$TEMP_DIR/test-wt" -b test-branch > /dev/null 2>&1
cd "$TEMP_DIR/test-wt"
result=$(detect_worktree)
if [ "$result" = "linked" ]; then
  log_pass "Linked worktree detected as linked"
else
  log_fail "Linked worktree detected as '$result'（expected 'linked'）"
fi

echo "=== Test 3: Detached HEAD detection ==="
git checkout --detach HEAD > /dev/null 2>&1
branch=$(git branch --show-current)
if [ -z "$branch" ]; then
  log_pass "Detached HEAD: branch is empty"
else
  log_fail "Detached HEAD: branch is '$branch'（expected empty）"
fi

echo "=== Test 4: Linked worktree + detached HEAD（Codex App simulation） ==="
result=$(detect_worktree)
branch=$(git branch --show-current)
if [ "$result" = "linked" ] && [ -z "$branch" ]; then
  log_pass "Codex App simulation: linked + detached HEAD"
else
  log_fail "Codex App simulation: result='$result'，branch='$branch'"
fi

echo "=== Test 5: Cleanup guard — linked worktree should NOT remove ==="
cd "$TEMP_DIR/test-wt"
result=$(detect_worktree)
if [ "$result" = "linked" ]; then
  log_pass "Cleanup guard: linked worktree correctly detected（would skip removal）"
else
  log_fail "Cleanup guard: expected 'linked'，got '$result'"
fi

echo "=== Test 6: Cleanup guard — main repo SHOULD remove ==="
cd "$TEMP_DIR/test-repo"
result=$(detect_worktree)
if [ "$result" = "normal" ]; then
  log_pass "Cleanup guard: main repo correctly detected（would proceed with removal）"
else
  log_fail "Cleanup guard: expected 'normal'，got '$result'"
fi

# Cleanup worktree before temp dir removal
cd "$TEMP_DIR/test-repo"
git worktree remove "$TEMP_DIR/test-wt" > /dev/null 2>&1 || true

echo ""
echo "=== Results: $PASS passed，$FAIL failed ==="
if [ "$FAIL" -gt 0 ]; then
  exit 1
fi
```

- [ ] **Step 3: 设置可执行并运行**

```bash
chmod +x tests/codex-app-compat/test-environment-detection.sh
./tests/codex-app-compat/test-environment-detection.sh
```

预期输出：6 passed，0 failed。

- [ ] **Step 4: 提交**

```bash
git add tests/codex-app-compat/test-environment-detection.sh
git commit -m "test: add environment detection tests for Codex App compat（PRI-823）

Tests git-dir vs git-common-dir comparison in normal repo，linked
worktree，detached HEAD，and cleanup guard scenarios。"
```

---

### Task 8: 最终验证

**文件：**
- 读取：所有 5 个修改的技能文件

- [ ] **Step 1: 运行自动化检测测试**

```bash
./tests/codex-app-compat/test-environment-detection.sh
```

预期：6 passed，0 failed。

- [ ] **Step 2: 读取每个修改文件并验证变更**

完整读取每个文件：
- `skills/using-git-worktrees/SKILL.md` — Step 0 存在，其余不变
- `skills/finishing-a-development-branch/SKILL.md` — Step 1.5 存在，清理守卫存在，其余不变
- `skills/subagent-driven-development/SKILL.md` — line 268 已更新
- `skills/executing-plans/SKILL.md` — line 68 已更新
- `skills/using-superpowers/references/codex-tools.md` — 末尾两个新部分

- [ ] **Step 3: 验证无意外变更**

```bash
git diff --stat HEAD~7
```

应显示确切 6 个文件变更（5 个技能文件 + 1 个测试文件）。无其他文件修改。

- [ ] **Step 4: 运行现有测试套件**

如有测试运行器：
```bash
# Run skill-triggering tests
./tests/skill-triggering/run-all.sh 2>/dev/null || echo "Skill triggering tests not available in this environment"

# Run SDD integration test
./tests/claude-code/test-subagent-driven-development-integration.sh 2>/dev/null || echo "SDD integration test not available in this environment"
```

注意：这些测试需要带 `--dangerously-skip-permissions` 的 Claude Code。如不可用，文档说明回归测试应手动运行。