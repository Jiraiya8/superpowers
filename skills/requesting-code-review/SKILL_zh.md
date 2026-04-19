---
name: requesting-code-review
description: Use when completing tasks, implementing major features, or before merging to verify work meets requirements
---

# 请求代码审查

分发 superpowers:code-reviewer 子代理以在问题扩散前捕获它们。审查员获得精确构建的上下文用于评估——绝非你会话的历史。这使审查员专注于工作产物而非你的思维过程，并为你自己的上下文保留空间用于继续工作。

**核心原则：及早审查，经常审查。**

## 何时请求审查

**强制性：**
- After each task in subagent-driven development
- After completing major feature
- Before merge to main

**可选但有价值：**
- When stuck (fresh perspective)
- Before refactoring (baseline check)
- After fixing complex bug

## 如何请求

**1. 获取 git SHAs：**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # or origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**2. 分发 code-reviewer 子代理：**

使用任务工具，类型为 superpowers:code-reviewer，填写 `code-reviewer.md` 中的模板

**占位符：**
- `{WHAT_WAS_IMPLEMENTED}` - What you just built
- `{PLAN_OR_REQUIREMENTS}` - What it should do
- `{BASE_SHA}` - Starting commit
- `{HEAD_SHA}` - Ending commit
- `{DESCRIPTION}` - Brief summary

**3. 处理反馈：**
- Fix Critical issues immediately
- Fix Important issues before proceeding
- Note Minor issues for later
- Push back if reviewer is wrong (with reasoning)

## 示例

```
[刚完成 Task 2: 添加验证函数]

你： 让我在继续前请求代码审查。

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[分发 superpowers:code-reviewer 子代理]
  WHAT_WAS_IMPLEMENTED: 对话索引的验证和修复函数
  PLAN_OR_REQUIREMENTS: docs/superpowers/plans/deployment-plan.md 中的 Task 2
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661
  DESCRIPTION: 添加了 verifyIndex() 和 repairIndex()，含 4 种问题类型

[子代理返回]:
  Strengths: 清晰架构，真实测试
  Issues:
    Important: 缺失进度指示器
    Minor: 报告间隔的魔法数字 (100)
  Assessment: 准备继续

你： [修复进度指示器]
[继续到 Task 3]
```

## 与工作流集成

**子代理驱动开发：**
- Review after EACH task
- Catch issues before they compound
- Fix before moving to next task

**执行计划：**
- Review after each batch (3 tasks)
- Get feedback, apply, continue

**临时开发：**
- Review before merge
- Review when stuck

## 警示信号

**切勿：**
- Skip review because "it's simple"
- Ignore Critical issues
- Proceed with unfixed Important issues
- Argue with valid technical feedback

**如果审查员错误：**
- Push back with technical reasoning
- Show code/tests that prove it works
- Request clarification

See template at: requesting-code-review/code-reviewer.md