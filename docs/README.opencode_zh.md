# Superpowers for OpenCode

[OpenCode.ai](https://opencode.ai) 使用 Superpowers 的完整指南。

## 安装

在 `opencode.json`（全局或项目级）的 `plugin` array 中添加 superpowers：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"]
}
```

重启 OpenCode。插件通过 Bun 自动安装并注册所有技能。

验证："Tell me about your superpowers"

### 从旧 symlink 安装迁移

如之前用 `git clone` 和 symlink 安装 superpowers，移除旧设置：

```bash
# 移除旧 symlink
rm -f ~/.config/opencode/plugins/superpowers.js
rm -rf ~/.config/opencode/skills/superpowers

# 可选移除克隆的 repo
rm -rf ~/.config/opencode/superpowers

# 如为 superpowers 在 opencode.json 添加了 skills.paths 则移除
```

然后按上述安装步骤执行。

## 使用

### 查找技能

使用 OpenCode 原生 `skill` 工具列出所有可用技能：

```
use skill tool to list skills
```

### 加载技能

```
use skill tool to load superpowers/brainstorming
```

### 个人技能

在 `~/.config/opencode/skills/` 创建自己的技能：

```bash
mkdir -p ~/.config/opencode/skills/my-skill
```

创建 `~/.config/opencode/skills/my-skill/SKILL.md`：

```markdown
---
name: my-skill
description: [条件]时使用 — [它做什么]
---

# My Skill

[Your skill content here]
```

### 项目技能

在项目 `.opencode/skills/` 创建项目特定技能。

**技能优先级：** 项目技能 > 个人技能 > Superpowers 技能

## 更新

重启 OpenCode 时 superpowers 自动更新。插件每次启动从 git repository 重新安装。

如需固定特定版本，使用 branch 或 tag：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git#v5.0.3"]
}
```

## 工作原理

插件做两件事：

1. **通过 `experimental.chat.system.transform` 钩子注入 bootstrap context**，每次对话添加 superpowers 感知。
2. **通过 `config` 钩子注册技能目录**，OpenCode 自动发现所有 superpowers 技能无需 symlink 或手动配置。

### 工具映射

为 Claude Code 编写的技能自动适配 OpenCode：

- `TodoWrite` → `todowrite`
- `Task` with subagents → OpenCode 的 `@mention` 系统
- `Skill` tool → OpenCode 原生 `skill` tool
- File operations → OpenCode 原生工具

## 故障排除

### 插件未加载

1. 检查 OpenCode logs：`opencode run --print-logs "hello" 2>&1 | grep -i superpowers`
2. 验证 `opencode.json` 中 plugin 行正确
3. 确保运行最近版本 OpenCode

### 技能未找到

1. 使用 OpenCode `skill` 工具列出可用技能
2. 检查插件加载（见上）
3. 每个技能需要有效 YAML frontmatter 的 `SKILL.md` 文件

### Bootstrap 未出现

1. 检查 OpenCode 版本支持 `experimental.chat.system.transform` 钩子
2. 配置变更后重启 OpenCode

## 获取帮助

- 报告问题：https://github.com/obra/superpowers/issues
- 主文档：https://github.com/obra/superpowers
- OpenCode docs：https://opencode.ai/docs/