# Superpowers for Codex

通过原生技能发现使用 Superpowers 配合 OpenAI Codex 的指南。

## 快速安装

告诉 Codex：

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.codex/INSTALL.md
```

## 手动安装

### 系统要求

- OpenAI Codex CLI
- Git

### 步骤

1. 克隆仓库：
   ```bash
   git clone https://github.com/obra/superpowers.git ~/.codex/superpowers
   ```

2. 创建技能 symlink：
   ```bash
   mkdir -p ~/.agents/skills
   ln -s ~/.codex/superpowers/skills ~/.agents/skills/superpowers
   ```

3. 重启 Codex。

4. **子代理技能**（可选）：`dispatching-parallel-agents` 和 `subagent-driven-development` 等技能需要 Codex 的 multi-agent 功能。添加到 Codex config：
   ```toml
   [features]
   multi_agent = true
   ```

### Windows

使用 junction 替代 symlink（无需 Developer Mode）：

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.agents\skills"
cmd /c mklink /J "$env:USERPROFILE\.agents\skills\superpowers" "$env:USERPROFILE\.codex\superpowers\skills"
```

## 工作原理

Codex 有原生技能发现 — 启动时扫描 `~/.agents/skills/`，解析 SKILL.md frontmatter，按需加载技能。通过单个 symlink 使 Superpowers 技能可见：

```
~/.agents/skills/superpowers/ → ~/.codex/superpowers/skills/
```

`using-superpowers` 技能自动发现并强制技能使用规范 — 无需额外配置。

## 使用

技能自动发现。Codex 在以下情况激活它们：
- 你提到技能名称（如 "use brainstorming"）
- 任务匹配技能描述
- `using-superpowers` 技能指示 Codex 使用某技能

### 个人技能

在 `~/.agents/skills/` 创建自己的技能：

```bash
mkdir -p ~/.agents/skills/my-skill
```

创建 `~/.agents/skills/my-skill/SKILL.md`：

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# My Skill

[Your skill content here]
```

`description` 字段是 Codex 决定何时自动激活技能的方式 — 写成清晰的触发条件。

## 更新

```bash
cd ~/.codex/superpowers && git pull
```

技能通过 symlink 立即更新。

## 卸载

```bash
rm ~/.agents/skills/superpowers
```

**Windows（PowerShell）：**
```powershell
Remove-Item "$env:USERPROFILE\.agents\skills\superpowers"
```

可选删除克隆：`rm -rf ~/.codex/superpowers`（Windows：`Remove-Item -Recurse -Force "$env:USERPROFILE\.codex\superpowers"`）。

## 故障排除

### 技能未显示

1. 验证 symlink：`ls -la ~/.agents/skills/superpowers`
2. 检查技能存在：`ls ~/.codex/superpowers/skills`
3. 重启 Codex — 启动时发现技能

### Windows junction 问题

Junction 通常无需特殊权限即可工作。如创建失败，尝试以管理员运行 PowerShell。

## 获取帮助

- 报告问题：https://github.com/obra/superpowers/issues
- 主文档：https://github.com/obra/superpowers