---
name: executing-plans
description: 有书面实施计划需在独立会话中执行时使用，带审查检查点
---

# 执行计划

## 概述

加载计划，批判性审查，执行所有任务，完成时报告。

**开始时声明："我正在使用 executing-plans 技能来实现此计划。"**

**注意：** 告知 your human partner Superpowers 在有子代理访问时效果更好。在有子代理支持的平台（如 Claude Code 或 Codex）上运行时，其工作质量将显著更高。如果子代理可用，使用 superpowers:subagent-driven-development 而非此技能。

## 流程

### 步骤 1：加载并审查计划
1. Read plan file
2. Review critically - identify any questions or concerns about the plan
3. If concerns: Raise them with your human partner before starting
4. If no concerns: Create TodoWrite and proceed

### 步骤 2：执行任务

对于每个任务：
1. Mark as in_progress
2. Follow each step exactly (plan has bite-sized steps)
3. Run verifications as specified
4. Mark as completed

### 步骤 3：完成开发

所有任务完成并验证后：
- Announce: "I'm using the finishing-a-development-branch skill to complete this work."
- **必需子技能：** 使用 superpowers:finishing-a-development-branch
- 遵循该技能验证测试、呈现选项、执行选择

## 何时应停止并请求帮助

**立即停止执行当：**
- Hit a blocker (missing dependency, test fails, instruction unclear)
- Plan has critical gaps preventing starting
- You don't understand an instruction
- Verification fails repeatedly

**请求澄清而非猜测。**

## 何时应回到早期步骤

**返回审查（步骤 1）当：**
- Partner updates the plan based on your feedback
- Fundamental approach needs rethinking

**不要强行通过阻塞**— 停止并询问。

## 记住
- Review plan critically first
- Follow plan steps exactly
- Don't skip verifications
- Reference skills when plan says to
- Stop when blocked, don't guess
- Never start implementation on main/master branch without explicit user consent

## 集成

**必需工作流技能：**
- **superpowers:using-git-worktrees** — 必需：开始前设置隔离工作空间
- **superpowers:writing-plans** — 创建此技能执行的计划
- **superpowers:finishing-a-development-branch** — 所有任务完成后完成开发