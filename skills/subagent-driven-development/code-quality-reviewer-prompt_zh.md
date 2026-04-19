# 代码质量审查员提示模板

当分发代码质量审查子代理时使用此模板。

**目的：** 验证实现构建良好（干净、已测试、可维护）

**仅在规范合规审查通过后分发。**

```
Task tool (superpowers:code-reviewer):
  Use template at requesting-code-review/code-reviewer.md

  WHAT_WAS_IMPLEMENTED: [来自实现者的报告]
  PLAN_OR_REQUIREMENTS: [计划文件]中的 Task N
  BASE_SHA: [任务前的提交]
  HEAD_SHA: [当前提交]
  DESCRIPTION: [任务摘要]
```

**除标准代码质量关注点外，审查员应检查：**
- 每个文件是否有单一明确的职责和定义良好的接口？
- 单元是否分解到可以独立理解和测试？
- 实现是否遵循计划中的文件结构？
- 此实现是否创建了已经很大的新文件，或显著增大了现有文件？（不要标记预先存在的文件大小 — 关注此变更带来的贡献。）

**代码审查员返回：** 优点、问题（关键/重要/次要）、评估