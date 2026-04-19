---
name: verification-before-completion
description: 声称工作完成、修复或通过前使用，在提交或创建PR前 — 要求运行验证命令并确认输出后再做任何成功声称；先有证据再做断言
---

# 完成前验证

## 概述

声称工作完成而未经验证是 dishonesty，而非 efficiency。

**核心原则：证据先于声明，始终如此。**

**违反这条规则的字面含义即是违反其精神。**

## 铁律

```
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

如果你没有在本次消息中运行验证命令，你就不能声称它通过了。

## 门禁机制

```
BEFORE claiming any status or expressing satisfaction:

1. IDENTIFY: What command proves this claim?
2. RUN: Execute the FULL command (fresh, complete)
3. READ: Full output, check exit code, count failures
4. VERIFY: Does output confirm the claim?
   - If NO: State actual status with evidence
   - If YES: State claim WITH evidence
5. ONLY THEN: Make the claim

Skip any step = lying, not verifying
```

## 常见失败

| 声明 | 需要 | 不充分 |
|-------|----------|----------------|
| Tests pass | Test command output: 0 failures | Previous run, "should pass" |
| Linter clean | Linter output: 0 errors | Partial check, extrapolation |
| Build succeeds | Build command: exit 0 | Linter passing, logs look good |
| Bug fixed | Test original symptom: passes | Code changed, assumed fixed |
| Regression test works | Red-green cycle verified | Test passes once |
| Agent completed | VCS diff shows changes | Agent reports "success" |
| Requirements met | Line-by-line checklist | Tests passing |

## 警示信号 - 停止

- 使用 "should"、"probably"、"seems to"
- 在验证前表示满意（"Great!"、"Perfect!"、"Done!" 等）
- 即将 commit/push/PR 却未验证
- 相信 agent success reports
- 依赖部分验证
- 认为 "just this once"
- 疲倦且想结束工作
- **任何暗示成功但未运行验证的措辞**

## 辩解预防

| 辩解 | 事实 |
|--------|---------|
| "Should work now" | 运行验证 |
| "I'm confident" | 信心 ≠ 证据 |
| "Just this once" | 无例外 |
| "Linter passed" | Linter ≠ compiler |
| "Agent said success" | 独立验证 |
| "I'm tired" | 精疲力竭 ≠ 辩解 |
| "Partial check is enough" | 部分验证什么也证明不了 |
| "Different words so rule doesn't apply" | 精神胜于字面 |

## 关键模式

**测试：**
```
✅ [Run test command] [See: 34/34 pass] "All tests pass"
❌ "Should pass now" / "Looks correct"
```

**回归测试（TDD Red-Green）：**
```
✅ Write → Run (pass) → Revert fix → Run (MUST FAIL) → Restore → Run (pass)
❌ "I've written a regression test" (without red-green verification)
```

**构建：**
```
✅ [Run build] [See: exit 0] "Build passes"
❌ "Linter passed" (linter doesn't check compilation)
```

**需求：**
```
✅ Re-read plan → Create checklist → Verify each → Report gaps or completion
❌ "Tests pass, phase complete"
```

**Agent delegation:**
```
✅ Agent reports success → Check VCS diff → Verify changes → Report actual state
❌ Trust agent report
```

## 为什么这很重要

来自 24 次失败记忆：
- your human partner said "I don't believe you" - trust broken
- Undefined functions shipped - would crash
- Missing requirements shipped - incomplete features
- Time wasted on false completion → redirect → rework
- Violates: "Honesty is a core value. If you lie, you'll be replaced."

## 适用时机

**始终在以下之前：**
- 任何成功/完成声明的变体
- 任何满意表达
- 任何关于工作状态的正面陈述
- Committing, PR creation, task completion
- 转向下一个任务
- Delegating to agents

**规则适用于：**
- 确切短语
- Paraphrases and synonyms
- Implications of success
- 任何暗示完成/正确性的沟通

## 底线

**验证无捷径。**

运行命令。阅读输出。然后声明结果。

这是不可协商的。