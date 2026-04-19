***

name: using-superpowers
description: Use when starting any conversation - establishes how to find and use skills, requiring Skill tool invocation before ANY response including clarifying questions
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------

<SUBAGENT-STOP>
If you were dispatched as a subagent to execute a specific task, skip this skill.
</SUBAGENT-STOP>

<EXTREMELY-IMPORTANT>
If you think there is even a 1% chance a skill might apply to what you are doing, you ABSOLUTELY MUST invoke the skill.

IF A SKILL APPLIES TO YOUR TASK, YOU DO NOT HAVE A CHOICE. YOU MUST USE IT.

This is not negotiable. This is not optional. You cannot rationalize your way out of this. </EXTREMELY-IMPORTANT>

## 指令优先级

Superpowers 技能覆盖默认系统提示行为，但**用户指令始终优先**：

1. **用户显式指令**（CLAUDE.md、GEMINI.md、AGENTS.md、直接请求）— 最高优先级
2. **Superpowers 技能**— 在冲突处覆盖默认系统行为
3. **默认系统提示**— 最低优先级

如果 CLAUDE.md、GEMINI.md 或 AGENTS.md 说 "don't use TDD" 而技能说 "always use TDD"，请遵循用户指令。用户掌控一切。

## 如何访问技能

**在 Claude Code：** 使用 `Skill` 工具。当你调用技能时，其内容被加载并呈现给你——直接遵循。切勿使用 Read 工具读取技能文件。

**在 Copilot CLI：** 使用 `skill` 工具。技能从已安装插件自动发现。`skill` 工具与 Claude Code 的 `Skill` 工具相同。

**在 Gemini CLI：** 技能通过 `activate_skill` 工具激活。Gemini 在会话开始时加载技能元数据并按需激活完整内容。

**在其他环境：** 查阅你的平台文档以了解技能如何加载。

## 平台适配

技能使用 Claude Code 工具名称。非 CC 平台：参阅 `references/copilot-tools.md`（Copilot CLI）、`references/codex-tools.md`（Codex）获取工具等效映射。Gemini CLI 用户通过 GEMINI.md 自动加载工具映射。

# 使用技能

## 规则

**在任何响应或行动之前调用相关或请求的技能。** 即使只有 1% 的可能性技能适用，你也应该调用技能来检查。如果调用的技能对情况不适用，你无需使用它。

```dot
digraph skill_flow {
    "User message received" [shape=doublecircle];
    "About to EnterPlanMode?" [shape=doublecircle];
    "Already brainstormed?" [shape=diamond];
    "Invoke brainstorming skill" [shape=box];
    "Might any skill apply?" [shape=diamond];
    "Invoke Skill tool" [shape=box];
    "Announce: 'Using [skill] to [purpose]'" [shape=box];
    "Has checklist?" [shape=diamond];
    "Create TodoWrite todo per item" [shape=box];
    "Follow skill exactly" [shape=box];
    "Respond (including clarifications)" [shape=doublecircle];

    "About to EnterPlanMode?" -> "Already brainstormed?";
    "Already brainstormed?" -> "Invoke brainstorming skill" [label="no"];
    "Already brainstormed?" -> "Might any skill apply?" [label="yes"];
    "Invoke brainstorming skill" -> "Might any skill apply?";

    "User message received" -> "Might any skill apply?";
    "Might any skill apply?" -> "Invoke Skill tool" [label="yes, even 1%"];
    "Might any skill apply?" -> "Respond (including clarifications)" [label="definitely not"];
    "Invoke Skill tool" -> "Announce: 'Using [skill] to [purpose]'";
    "Announce: 'Using [skill] to [purpose]'" -> "Has checklist?";
    "Has checklist?" -> "Create TodoWrite todo per item" [label="yes"];
    "Has checklist?" -> "Follow skill exactly" [label="no"];
    "Create TodoWrite todo per item" -> "Follow skill exactly";
}
```

## 警示信号

这些想法意味着停止——你在辩解：

| Thought                             | Reality                                                |
| ----------------------------------- | ------------------------------------------------------ |
| "This is just a simple question"    | Questions are tasks. Check for skills.                 |
| "I need more context first"         | Skill check comes BEFORE clarifying questions.         |
| "Let me explore the codebase first" | Skills tell you HOW to explore. Check first.           |
| "I can check git/files quickly"     | Files lack conversation context. Check for skills.     |
| "Let me gather information first"   | Skills tell you HOW to gather information.             |
| "This doesn't need a formal skill"  | If a skill exists, use it.                             |
| "I remember this skill"             | Skills evolve. Read current version.                   |
| "This doesn't count as a task"      | Action = task. Check for skills.                       |
| "The skill is overkill"             | Simple things become complex. Use it.                  |
| "I'll just do this one thing first" | Check BEFORE doing anything.                           |
| "This feels productive"             | Undisciplined action wastes time. Skills prevent this. |
| "I know what that means"            | Knowing the concept ≠ using the skill. Invoke it.      |

## 技能优先级

当多个技能可能适用时，按此顺序：

1. **流程技能优先**（brainstorming、debugging）— 这些决定如何处理任务
2. **实现技能其次**（frontend-design、mcp-builder）— 这些指导执行

"Let's build X" → brainstorming first, then implementation skills.
"Fix this bug" → debugging first, then domain-specific skills.

## 技能类型

**严格型**（TDD、debugging）：精确遵循。不要摆脱纪律。

**灵活型**（patterns）：将原则适配到上下文。

技能本身会告诉你属于哪种类型。

## 用户指令

指令说做什么，不说怎么做。"Add X" 或 "Fix Y" 不意味着跳过工作流程。
