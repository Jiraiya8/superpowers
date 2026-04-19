# Gemini CLI 工具映射

技能使用 Claude Code 工具名称。当你在技能中遇到这些名称时，请使用你的平台等效工具：

| Skill references | Gemini CLI equivalent |
|-----------------|----------------------|
| `Read` (file reading) | `read_file` |
| `Write` (file creation) | `write_file` |
| `Edit` (file editing) | `replace` |
| `Bash` (run commands) | `run_shell_command` |
| `Grep` (search file content) | `grep_search` |
| `Glob` (search files by name) | `glob` |
| `TodoWrite` (task tracking) | `write_todos` |
| `Skill` tool (invoke a skill) | `activate_skill` |
| `WebSearch` | `google_web_search` |
| `WebFetch` | `web_fetch` |
| `Task` tool (dispatch subagent) | No equivalent — Gemini CLI does not support subagents |

## 无子代理支持

Gemini CLI 没有等效于 Claude Code 的 `Task` 工具。依赖子代理分发的技能（`subagent-driven-development`、`dispatching-parallel-agents`）将通过 `executing-plans` 回退到单会话执行。

## Gemini CLI 额外工具

这些工具在 Gemini CLI 中可用，但在 Claude Code 中无等效工具：

| Tool | Purpose |
|------|---------|
| `list_directory` | List files and subdirectories |
| `save_memory` | Persist facts to GEMINI.md across sessions |
| `ask_user` | Request structured input from the user |
| `tracker_create_task` | Rich task management (create, update, list, visualize) |
| `enter_plan_mode` / `exit_plan_mode` | Switch to read-only research mode before making changes |