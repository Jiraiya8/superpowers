# OpenCode 支持设计

**日期：** 2025-11-22
**作者：** Bot & Jesse
**状态：** 设计完成，等待实现

## 概述

使用原生 OpenCode 插件架构为 OpenCode.ai 添加完整 superpowers 支持，该架构与现有 Codex 实现共享核心功能。

## 背景

OpenCode.ai 是类似 Claude Code 和 Codex 的编码代理。之前尝试将 superpowers 移植到 OpenCode（PR #93、PR #116）使用文件复制方法。此设计采用不同方法：使用其 JavaScript/TypeScript 插件系统构建原生 OpenCode 插件，同时与 Codex 实现共享代码。

### 平台间关键差异

- **Claude Code**：原生 Anthropic 插件系统 + 文件式技能
- **Codex**：无插件系统 → bootstrap markdown + CLI script
- **OpenCode**：带事件钩子和自定义工具 API 的 JavaScript/TypeScript 插件

### OpenCode 的代理系统

- **Primary agents**：Build（默认，完全访问）和 Plan（受限，只读）
- **Subagents**：General（研究、搜索、多步任务）
- **调用**：Primary agents 自动分发 或手动 `@mention` 语法
- **配置**：`opencode.json` 或 `~/.config/opencode/agent/` 中的自定义代理

## 架构

### 高层结构

1. **共享核心模块**（`lib/skills-core.js`）
   - 通用技能发现和解析逻辑
   - 被 Codex 和 OpenCode 实现使用

2. **平台特定封装器**
   - Codex：CLI script（`.codex/superpowers-codex`）
   - OpenCode：Plugin module（`.opencode/plugin/superpowers.js`）

3. **技能目录**
   - Core：`~/.config/opencode/superpowers/skills/`（或安装位置）
   - Personal：`~/.config/opencode/skills/`（覆盖 core 技能）

### 代码复用策略

从 `.codex/superpowers-codex` 提取通用功能到共享模块：

```javascript
// lib/skills-core.js
module.exports = {
  extractFrontmatter(filePath),      // Parse name + description from YAML
  findSkillsInDir(dir, maxDepth),    // Recursive SKILL.md discovery
  findAllSkills(dirs),                // Scan multiple directories
  resolveSkillPath(skillName, dirs), // Handle shadowing（personal > core）
  checkForUpdates(repoDir)           // Git fetch/status check
};
```

### 技能 Frontmatter 格式

当前格式（无 `when_to_use` 字段）：

```yaml
---
name: skill-name
description: Use when [condition] - [what it does]; [additional context]
---
```

## OpenCode 插件实现

### 自定义工具

**工具 1：`use_skill`**

加载特定技能内容到对话（等同于 Claude 的 Skill 工具）。

```javascript
{
  name: 'use_skill',
  description: 'Load and read a specific skill to guide your work',
  schema: z.object({
    skill_name: z.string().describe('Name of skill（e.g.，"superpowers:brainstorming")')
  }),
  execute: async ({ skill_name }) => {
    const { skillPath，content，frontmatter } = resolveAndReadSkill(skill_name);
    const skillDir = path.dirname(skillPath);

    return `# ${frontmatter.name}
# ${frontmatter.description}
# Supporting tools and docs are in ${skillDir}
# ============================================

${content}`;
  }
}
```

**工具 2：`find_skills`**

列出所有可用技能及其元数据。

```javascript
{
  name: 'find_skills',
  description: 'List all available skills',
  schema: z.object({}),
  execute: async () => {
    const skills = discoverAllSkills();
    return skills.map(s =>
      `${s.namespace}:${s.name}
  ${s.description}
  Directory: ${s.directory}
`).join('\n');
  }
}
```

### 会话启动钩子

当新会话启动时（`session.started` event）：

1. **注入 using-superpowers 内容**
   - using-superpowers 技能的完整内容
   - 建立强制性工作流

2. **自动运行 find_skills**
   - 提前显示完整可用技能列表
   - 包含每个技能目录

3. **注入工具映射指令**
   ```markdown
   **Tool Mapping for OpenCode:**
   When skills reference tools you don't have，substitute:
   - `TodoWrite` → `update_plan`
   - `Task` with subagents → Use OpenCode subagent system（@mention）
   - `Skill` tool → `use_skill` custom tool
   - Read，Write，Edit，Bash → Your native equivalents

   **Skill directories contain:**
   - Supporting scripts（run with bash）
   - Additional documentation（read with read tool）
   - Utilities specific to that skill
   ```

4. **检查更新**（非阻塞）
   - 带超时的快速 git fetch
   - 如有更新则通知

### 插件结构

```javascript
// .opencode/plugin/superpowers.js
const skillsCore = require('../../lib/skills-core');
const path = require('path');
const fs = require('fs');
const { z } = require('zod');

export const SuperpowersPlugin = async ({ client，directory，$ }) => {
  const superpowersDir = path.join(process.env.HOME，'.config/opencode/superpowers');
  const personalDir = path.join(process.env.HOME，'.config/opencode/skills');

  return {
    'session.started': async () => {
      const usingSuperpowers = await readSkill('using-superpowers');
      const skillsList = await findAllSkills();
      const toolMapping = getToolMappingInstructions();

      return {
        context: `${usingSuperpowers}\n\n${skillsList}\n\n${toolMapping}`
      };
    },

    tools: [
      {
        name: 'use_skill',
        description: 'Load and read a specific skill',
        schema: z.object({
          skill_name: z.string()
        }),
        execute: async ({ skill_name }) => {
          // Implementation using skillsCore
        }
      },
      {
        name: 'find_skills',
        description: 'List all available skills',
        schema: z.object({}),
        execute: async () => {
          // Implementation using skillsCore
        }
      }
    ]
  };
};
```

## 文件结构

```
superpowers/
├── lib/
│   └── skills-core.js           # NEW: Shared skill logic
├── .codex/
│   ├── superpowers-codex        # UPDATED: Use skills-core
│   ├── superpowers-bootstrap.md
│   └── INSTALL.md
├── .opencode/
│   ├── plugin/
│   │   └── superpowers.js       # NEW: OpenCode plugin
│   └── INSTALL.md               # NEW: Installation guide
└── skills/                       # Unchanged
```

## 实现计划

### Phase 1：重构共享核心

1. 创建 `lib/skills-core.js`
   - 从 `.codex/superpowers-codex` 提取 frontmatter 解析
   - 提取技能发现逻辑
   - 提取路径解析（带覆盖）
   - 更新仅使用 `name` 和 `description`（无 `when_to_use`）

2. 更新 `.codex/superpowers-codex` 使用共享核心
   - 从 `../lib/skills-core.js` 导入
   - 移除重复代码
   - 保留 CLI wrapper 逻辑

3. 测试 Codex 实现仍工作
   - 验证 bootstrap 命令
   - 验证 use-skill 命令
   - 验证 find-skills 命令

### Phase 2：构建 OpenCode 插件

1. 创建 `.opencode/plugin/superpowers.js`
   - 从 `../../lib/skills-core.js` 导入共享核心
   - 实现 plugin 函数
   - 定义自定义工具（use_skill、find_skills）
   - 实现 session.started 钩子

2. 创建 `.opencode/INSTALL.md`
   - 安装说明
   - 目录设置
   - 配置指导

3. 测试 OpenCode 实现
   - 验证会话启动 bootstrap
   - 验证 use_skill 工具工作
   - 验证 find_skills 工具工作
   - 验证技能目录可访问

### Phase 3：文档和打磨

1. 更新 README 添加 OpenCode 支持
2. 添加 OpenCode 安装到主文档
3. 更新 RELEASE-NOTES
4. 测试 Codex 和 OpenCode 都正常工作

## 下一步

1. **创建隔离工作空间**（使用 git worktrees）
   - Branch：`feature/opencode-support`

2. **适用时遵循 TDD**
   - 测试共享核心函数
   - 测试技能发现和解析
   - 两个平台的集成测试

3. **增量实现**
   - Phase 1：重构共享核心 + 更新 Codex
   - 验证 Codex 仍工作后再继续
   - Phase 2：构建 OpenCode 插件
   - Phase 3：文档和打磨

4. **测试策略**
   - 用真实 OpenCode 安装手动测试
   - 验证技能加载、目录、脚本工作
   - Codex 和 OpenCode 并行测试
   - 验证工具映射正确工作

5. **PR 和合并**
   - 创建完整实现的 PR
   - 在干净环境测试
   - 合并到 main

## 收益

- **代码复用**：技能发现/解析的单一真相来源
- **可维护性**：Bug 修复应用于两个平台
- **可扩展性**：易于添加未来平台（Cursor、Windsurf 等）
- **原生集成**：正确使用 OpenCode 的插件系统
- **一致性**：所有平台相同的技能体验