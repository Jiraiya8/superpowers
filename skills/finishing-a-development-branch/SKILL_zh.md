---
name: finishing-a-development-branch
description: 实现完成、所有测试通过、需决定如何集成工作时使用 — 提供合并、PR或清理的结构化选项指导
---

# 完成开发分支

## 概述

通过呈现清晰选项和处理选定工作流来指导开发工作的完成。

**核心原则：验证测试 → 呈现选项 → 执行选择 → 清理。**

**开始时声明："我正在使用 finishing-a-development-branch 技能来完成此工作。"**

## 流程

### 步骤 1：验证测试

**在呈现选项前，验证测试通过：**

```bash
# Run project's test suite
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

停止。不继续到步骤 2。

**如果测试通过：** 继续到步骤 2。

### 步骤 2：确定基分支

```bash
# Try common base branches
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或询问："This branch split from main - is that correct?"

### 步骤 3：呈现选项

准确呈现这 4 个选项：

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**不要添加解释**— 保持选项简洁。

### 步骤 4：执行选择

#### 选项 1：本地合并

```bash
# Switch to base branch
git checkout <base-branch>

# Pull latest
git pull

# Merge feature branch
git merge <feature-branch>

# Verify tests on merged result
<test command>

# If tests pass
git branch -d <feature-branch>
```

然后：清理 worktree（步骤 5）

#### 选项 2：推送并创建 PR

```bash
# Push branch
git push -u origin <feature-branch>

# Create PR
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

然后：清理 worktree（步骤 5）

#### 选项 3：保持原样

报告："Keeping branch <name>. Worktree preserved at <path>."

**不清理 worktree。**

#### 选项 4：丢弃

**先确认：**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

等待确切确认。

如果确认：
```bash
git checkout <base-branch>
git branch -D <feature-branch>
```

然后：清理 worktree（步骤 5）

### 步骤 5：清理 Worktree

**对于选项 1、2、4：**

检查是否在 worktree 中：
```bash
git worktree list | grep $(git branch --show-current)
```

如果是：
```bash
git worktree remove <worktree-path>
```

**对于选项 3：** 保持 worktree。

## 快速参考

| Option | Merge | Push | Keep Worktree | Cleanup Branch |
|--------|-------|------|---------------|----------------|
| 1. Merge locally | ✓ | - | - | ✓ |
| 2. Create PR | - | ✓ | ✓ | - |
| 3. Keep as-is | - | - | ✓ | - |
| 4. Discard | - | - | - | ✓ (force) |

## 常见错误

**跳过测试验证**
- **问题：** 合并损坏代码，创建失败 PR
- **修复：** 始终在提供选项前验证测试

**开放式问题**
- **问题：** "What should I do next?" → 模糊
- **修复：** 准确呈现 4 个结构化选项

**自动 worktree 清理**
- **问题：** 在可能需要时删除 worktree（选项 2、3）
- **修复：** 仅对选项 1 和 4 清理

**丢弃无确认**
- **问题：** 意外删除工作
- **修复：** 要求输入 "discard" 确认

## 警示信号

**切勿：**
- Proceed with failing tests
- Merge without verifying tests on result
- Delete work without confirmation
- Force-push without explicit request

**始终：**
- Verify tests before offering options
- Present exactly 4 options
- Get typed confirmation for Option 4
- Clean up worktree for Options 1 & 4 only

## 成成

**被调用者：**
- **subagent-driven-development**（步骤 7）— 所有任务完成后
- **executing-plans**（步骤 5）— 所有批次完成后

**配对者：**
- **using-git-worktrees**— 清理该技能创建的 worktree