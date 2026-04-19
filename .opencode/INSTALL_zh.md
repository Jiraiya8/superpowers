# 为 OpenCode 安装 Superpowers

## 前提条件

- [OpenCode.ai](https://opencode.ai) 已安装

## 安装

在你的 `opencode.json`（全局或项目级）的 `plugin` 数组中添加 superpowers：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"]
}
```

重启 OpenCode。就这样 — 插件自动安装并注册所有技能。

通过问"告诉我你的 superpowers"验证。

## 从旧的 symlink 安装迁移

如果你之前使用 `git clone` 和 symlink 安装 superpowers，移除旧设置：

```bash
# 移除旧 symlink
rm -f ~/.config/opencode/plugins/superpowers.js
rm -rf ~/.config/opencode/skills/superpowers

# 可选移除克隆的 repo
rm -rf ~/.config/opencode/superpowers

# 如果你为 superpowers 添加了 skills.paths 到 opencode.json，移除它
```

然后按照上面的安装步骤。

## 使用

使用 OpenCode 原生 `skill` 工具：

```
use skill tool to list skills
use skill tool to load superpowers/brainstorming
```

## 更新

Superpowers 在你重启 OpenCode 时自动更新。

要固定特定版本：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git#v5.0.3"]
}
```

## 故障排除

### 插件不加载

1. 检查日志：`opencode run --print-logs "hello" 2>&1 | grep -i superpowers`
2. 验证 `opencode.json` 中的 plugin 行
3. 确保你运行的是近期版本的 OpenCode

### 技能未找到

1. 使用 `skill` 工具列出已发现的
2. 检查插件是否加载（见上）

### 工具映射

当技能引用 Claude Code 工具：
- `TodoWrite` → `todowrite`
- `Task` with subagents → `@mention` 语法
- `Skill` 工具 → OpenCode 原生 `skill` 工具
- 文件操作 → 你的原生工具

## 获取帮助

- 报告 issue：https://github.com/obra/superpowers/issues
- 完整文档：https://github.com/obra/superpowers/blob/main/docs/README.opencode.md