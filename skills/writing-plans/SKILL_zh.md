---
name: writing-plans
description: 有多步骤任务的规格或需求时使用，在触碰代码前
---

# 编写计划

## 概述

编写全面的实施计划，假设工程师对我们的代码库零背景知识且品味存疑。记录他们需要知道的一切：每个任务需要修改哪些文件、代码、测试、可能需要查阅的文档，以及如何测试。将整个计划分解为小粒度任务。DRY。YAGNI。TDD。频繁提交。

假设他们是有经验的开发者，但对我们工具集或问题领域几乎一无所知。假设他们不太了解良好的测试设计。

**开始时宣布：** "我正在使用 writing-plans 技能来创建实施计划。"

**上下文：** 这应该在一个独立的工作树中运行（由 brainstorming 技能创建）。

**计划保存至：** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- （用户对计划位置的首选项覆盖此默认值）

## 范围检查

如果规范涵盖多个独立子系统，应该在做计划时将其拆分为子项目规范。如果没有拆分，建议将此拆分为单独的计划——每个子系统一个计划。每个计划应该能够独立产出可工作、可测试的软件。

## 文件结构

在定义任务之前，规划出将要创建或修改哪些文件，以及每个文件负责什么。这是分解决策确定的地方。

- 设计具有清晰边界和明确定义接口的单元。每个文件应该有一个明确的职责。
- 你最能理解一次性能容纳在上下文中的代码，当文件聚焦时你的编辑也更可靠。优先选择小型、聚焦的文件，而不是做太多事情的大型文件。
- 一起变化的文件应该放在一起。按职责拆分，而不是按技术层拆分。
- 在现有代码库中，遵循既定模式。如果代码库使用大型文件，不要单方面重构——但如果你正在修改的文件已经变得难以控制，在计划中包含拆分是合理的。

此结构指导任务分解。每个任务应该产出独立合理的自包含变更。

## 小粒度任务粒度

**每个步骤是一个动作（2-5 分钟）：**
- "编写失败的测试" - 步骤
- "运行它以确保它失败" - 步骤
- "实现使测试通过的最小代码" - 步骤
- "运行测试并确保它们通过" - 步骤
- "提交" - 步骤

## 计划文档头部

**每个计划必须以此头部开始：**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## 任务结构

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## 禁止占位符

每个步骤必须包含工程师需要的实际内容。这些都是**计划失败**——永远不要写这些：
- "TBD"、"TODO"、"稍后实现"、"填写细节"
- "添加适当的错误处理"/"添加验证"/"处理边缘情况"
- "为上述编写测试"（没有实际测试代码）
- "类似于任务 N"（重复代码——工程师可能不按顺序阅读任务）
- 描述做什么但不展示如何做的步骤（代码步骤需要代码块）
- 引用在任何任务中都未定义的类型、函数或方法

## 记住
- 始终使用精确的文件路径
- 每个步骤中包含完整代码——如果步骤修改代码，展示代码
- 精确的命令及预期输出
- DRY、YAGNI、TDD、频繁提交

## 自我审查

编写完整计划后，用新眼光查看规范并对照检查计划。这是你自己运行的检查清单——不是分派子代理。

**1. 规范覆盖：** 浏览规范中的每个部分/需求。你能指出实现它的任务吗？列出任何缺口。

**2. 占位符扫描：** 在计划中搜索危险信号——上面"禁止占位符"部分中的任何模式。修复它们。

**3. 类型一致性：** 你在后续任务中使用的类型、方法签名和属性名称是否与早期任务中定义的匹配？任务 3 中调用 `clearLayers()` 但任务 7 中调用 `clearFullLayers()` 是一个 bug。

如果发现问题，在线修复。不需要重新审查——只需修复并继续。如果发现没有任务对应规范需求，添加任务。

## 执行交接

保存计划后，提供执行选择：

**"计划已完成并保存至 `docs/superpowers/plans/<filename>.md`。两种执行选项：**

**1. 子代理驱动（推荐）** - 我为每个任务分派一个新子代理，任务间审查，快速迭代

**2. 内联执行** - 使用 executing-plans 在此会话中执行任务，批量执行并设检查点

**选择哪种方式？"**

**如果选择子代理驱动：**
- **必需子技能：** 使用 superpowers:subagent-driven-development
- 每个任务一个新子代理 + 两阶段审查

**如果选择内联执行：**
- **必需子技能：** 使用 superpowers:executing-plans
- 批量执行并设检查点供审查