# Codex 工具映射

技能使用 Claude Code 工具名称。当你在技能中遇到这些名称时，请使用你的平台等效工具：

| Skill references | Codex equivalent |
|-----------------|------------------|
| `Task` tool (dispatch subagent) | `spawn_agent` (see [Named agent dispatch](#named-agent-dispatch)) |
| Multiple `Task` calls (parallel) | Multiple `spawn_agent` calls |
| Task returns result | `wait` |
| Task completes automatically | `close_agent` to free slot |
| `TodoWrite` (task tracking) | `update_plan` |
| `Skill` tool (invoke a skill) | Skills load natively — just follow the instructions |
| `Read`, `Write`, `Edit` (files) | Use your native file tools |
| `Bash` (run commands) | Use your native shell tools |

## 子代理分发需要多代理支持

添加到你的 Codex 配置（`~/.codex/config.toml`）：

```toml
[features]
multi_agent = true
```

这会启用 `spawn_agent`、`wait` 和 `close_agent`，用于技能如 `dispatching-parallel-agents` 和 `subagent-driven-development`。

## Named agent dispatch

Claude Code skills reference named agent types like `superpowers:code-reviewer`.
Codex does not have a named agent registry — `spawn_agent` creates generic agents
from built-in roles (`default`, `explorer`, `worker`).

当技能说要分发命名代理类型时：

1. Find the agent's prompt file (e.g., `agents/code-reviewer.md` or the skill's
   local prompt template like `code-quality-reviewer-prompt.md`)
2. Read the prompt content
3. Fill any template placeholders (`{BASE_SHA}`, `{WHAT_WAS_IMPLEMENTED}`, etc.)
4. Spawn a `worker` agent with the filled content as the `message`

| Skill instruction | Codex equivalent |
|-------------------|------------------|
| `Task tool (superpowers:code-reviewer)` | `spawn_agent(agent_type="worker", message=...)` with `code-reviewer.md` content |
| `Task tool (general-purpose)` with inline prompt | `spawn_agent(message=...)` with the same prompt |

### Message framing

`message` 参数是用户级输入，而非系统提示。结构化以实现最大指令遵循：

```
你的任务是执行以下操作。严格遵循下面的指令。

<agent-instructions>
[从代理的 .md 文件填充的 prompt 内容]
</agent-instructions>

现在执行。只输出按照上面指令中指定格式的结构化响应。
```

- 使用任务委派框架（"你的任务是..."）而非角色框架（"你是...")
- 将指令包裹在 XML 标签中 — 模型将标签块视为权威
- 以明确的执行指令结尾以防止指令被总结

### 何时可以移除此变通方案

此方法补偿 Codex 的插件系统尚未支持 `plugin.json` 中的 `agents` 字段。当 `RawPluginManifest` 获得 `agents` 字段时，插件可以符号链接到 `agents/`（镜像现有的 `skills/` 符号链接），技能可以直接分发命名代理类型。

## 环境检测

创建 worktrees 或完成分支的技能应在继续之前使用只读 git 命令检测其环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → already in a linked worktree (skip creation)
- `BRANCH` empty → detached HEAD (cannot branch/push/PR from sandbox)

See `using-git-worktrees` Step 0 and `finishing-a-development-branch`
Step 1 for how each skill uses these signals.

## Codex App Finishing

When the sandbox blocks branch/push operations (detached HEAD in an
externally managed worktree), the agent commits all work and informs
the user to use the App's native controls:

- **"Create branch"** — names the branch, then commit/push/PR via App UI
- **"Hand off to local"** — transfers work to the user's local checkout

The agent can still run tests, stage files, and output suggested branch
names, commit messages, and PR descriptions for the user to copy.