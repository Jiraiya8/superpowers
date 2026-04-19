# 技能设计中的说服原则

## 概述

LLM 对与人类相同的说服原则产生响应。理解这种心理学有助于你设计更有效的技能——不是为了操纵，而是确保关键实践即使在压力下也能被遵循。

**研究基础：** Meincke 等人（2025）用 N=28,000 个 AI 对话测试了 7 个说服原则。说服技术使合规率翻倍以上（33% → 72%，p < .001）。

## 七个原则

### 1. 权威
**是什么：** 对专业知识、资质或官方来源的遵从。

**在技能中如何工作：**
- 强制性语言："YOU MUST"、"Never"、"Always"
- 不可协商的框架："No exceptions"
- 消除决策疲劳和合理化

**何时使用：**
- 强制纪律的技能（TDD、验证要求）
- 安全关键实践
- 既定的最佳实践

**示例：**
```markdown
✅ Write code before test? Delete it. Start over. No exceptions.
❌ Consider writing tests first when feasible.
```

### 2. 承诺
**是什么：** 与先前行动、声明或公开承诺的一致性。

**在技能中如何工作：**
- 要求宣布："Announce skill usage"
- 强制明确选择："Choose A, B, or C"
- 使用跟踪：TodoWrite 用于检查清单

**何时使用：**
- 确保技能被实际遵循
- 多步骤流程
- 问责机制

**示例：**
```markdown
✅ When you find a skill, you MUST announce: "I'm using [Skill Name]"
❌ Consider letting your partner know which skill you're using.
```

### 3. 稀缺
**是什么：** 来自时间限制或有限可用性的紧迫感。

**在技能中如何工作：**
- 时间绑定要求："Before proceeding"
- 顺序依赖："Immediately after X"
- 防止拖延

**何时使用：**
- 即时验证要求
- 时间敏感工作流
- 防止"稍后再做"

**示例：**
```markdown
✅ After completing a task, IMMEDIATELY request code review before proceeding.
❌ You can review code when convenient.
```

### 4. 社会认同
**是什么：** 遵从他人所为或被视为正常的事情。

**在技能中如何工作：**
- 普遍模式："Every time"、"Always"
- 失败模式："X without Y = failure"
- 建立规范

**何时使用：**
- 记录普遍实践
- 警告常见失败
- 强化标准

**示例：**
```markdown
✅ Checklists without TodoWrite tracking = steps get skipped. Every time.
❌ Some people find TodoWrite helpful for checklists.
```

### 5. 统一
**是什么：** 共同身份、"我们感"、群体归属感。

**在技能中如何工作：**
- 协作语言："our codebase"、"we're colleagues"
- 共同目标："we both want quality"

**何时使用：**
- 协作工作流
- 建立团队文化
- 非层级实践

**示例：**
```markdown
✅ We're colleagues working together. I need your honest technical judgment.
❌ You should probably tell me if I'm wrong.
```

### 6. 互惠
**是什么：** 回报所获利益的义务感。

**如何工作：**
- 谨慎使用 - 可能显得操纵性
- 技能中很少需要

**何时避免：**
- 几乎总是（其他原则更有效）

### 7. 喜好
**是什么：** 偏好与喜欢的人合作。

**如何工作：**
- **不要用于合规**
- 与诚实反馈文化冲突
- 创造谄媚行为

**何时避免：**
- 用于纪律强制时始终避免

## 按技能类型的原则组合

| 技能类型 | 使用 | 避免 |
|------------|-----|-------|
| 强制纪律 | Authority + Commitment + Social Proof | Liking, Reciprocity |
| 指导/技术 | Moderate Authority + Unity | Heavy authority |
| 协作 | Unity + Commitment | Authority, Liking |
| 参考 | 仅清晰度 | 所有说服 |

## 为什么有效：心理学

**明确规则减少合理化：**
- "YOU MUST" 消除决策疲劳
- 绝对语言消除"这是例外吗？"的问题
- 明确的反合理化填补特定漏洞

**执行意图创造自动行为：**
- 清晰触发 + 必须行动 = 自动执行
- "When X, do Y" 比"generally do Y" 更有效
- 减少合规的认知负荷

**LLM 是类人的：**
- 训练数据包含人类文本中的这些模式
- 权威语言在训练数据中先于合规出现
- 承诺序列（声明 → 行动）频繁被建模
- 社会认同模式（每个人都做 X）建立规范

## 道德使用

**合法使用：**
- 确保关键实践被遵循
- 创建有效的文档
- 防止可预测的失败

**不合法使用：**
- 为个人利益操纵
- 造虚假紧迫感
- 基于内疚的合规

**测试：** 如果用户完全理解此技术，它是否服务于用户的真实利益？

## 研究引用

**Cialdini, R. B. (2021).** *Influence: The Psychology of Persuasion (New and Expanded).* Harper Business.
- 说服的七个原则
- 影响研究的实证基础

**Meincke, L., Shapiro, D., Duckworth, A. L., Mollick, E., Mollick, L., & Cialdini, R. (2025).** Call Me A Jerk: Persuading AI to Comply with Objectionable Requests. University of Pennsylvania.
- 用 N=28,000 个 LLM 对话测试了 7 个原则
- 说服技术使合规从 33% 增加到 72%
- 权威、承诺、稀缺最有效
- 验证了 LLM 行为的类人模型

## 快速参考

设计技能时，问：

1. **它是什么类型？**（纪律 vs 指导 vs 参考）
2. **我试图改变什么行为？**
3. **哪些原则适用？**（通常纪律用 authority + commitment）
4. **我是否组合太多？**（不要使用全部七个）
5. **这是道德的吗？**（服务于用户的真实利益？）