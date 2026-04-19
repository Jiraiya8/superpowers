# Superpowers

Superpowers 是为你的编码 agent 提供的完整软件开发方法论，建立在一组可组合技能和一些初始指令之上，确保你的 agent 使用它们。

## 工作原理

从你启动编码 agent 的那一刻开始。一旦它看到你在构建什么，它*不会*直接跳到尝试写代码。相反，它退一步问你真正想做什么。

一旦从对话中提炼出规格，它以足够短、能实际阅读和消化的小块呈现给你。

在你批准设计后，你的 agent 组装一个实现计划，足够清晰以至于一个热情但品味差、无判断力、无项目上下文、厌恶测试的初级工程师也能遵循。它强调真正的红/绿 TDD、YAGNI（你不需要它）和 DRY。

接下来，一旦你说"go"，它启动一个 *subagent-driven-development* 过程，让 agent 完成每个工程任务，检查和审核他们的工作，并继续前进。Claude 能够自主工作几个小时而不偏离你们一起制定的计划，这并不罕见。

还有更多内容，但那是系统的核心。因为技能自动触发，你不需要做任何特殊操作。你的编码 agent 就有 Superpowers。


## 赞助

如果 Superpowers 帮助你做赚钱的事情，并且你愿意的话，我非常感激你考虑 [赞助我的开源工作](https://github.com/sponsors/obra)。

谢谢！

- Jesse


## 安装

**注意：** 安装因平台而异。

### Claude Code 官方市场

Superpowers 通过 [官方 Claude 插件市场](https://claude.com/plugins/superpowers) 提供

从 Anthropic 官方市场安装插件：

```bash
/plugin install superpowers@claude-plugins-official
```

### Claude Code (Superpowers Marketplace)

Superpowers marketplace 为 Claude Code 提供 Superpowers 和一些相关插件。

在 Claude Code 中，首先注册 marketplace：

```bash
/plugin marketplace add obra/superpowers-marketplace
```

然后从该 marketplace 安装插件：

```bash
/plugin install superpowers@superpowers-marketplace
```

### OpenAI Codex CLI

- 打开插件搜索界面

```bash
/plugins
```

搜索 Superpowers

```bash
superpowers
```

选择 `Install Plugin`

### OpenAI Codex App

- 在 Codex app 中，点击侧边栏的 Plugins。
- 你应该在 Coding 部分看到 `Superpowers`。
- 点击 Superpowers 旁边的 `+` 并按照提示操作。


### Cursor (通过插件市场)

在 Cursor Agent chat 中，从 marketplace 安装：

```text
/add-plugin superpowers
```

或在插件市场中搜索 "superpowers"。

### OpenCode

告诉 OpenCode：

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
```

**详细文档：** [docs/README.opencode.md](docs/README.opencode.md)

### GitHub Copilot CLI

```bash
copilot plugin marketplace add obra/superpowers-marketplace
copilot plugin install superpowers@superpowers-marketplace
```

### Gemini CLI

```bash
gemini extensions install https://github.com/obra/superpowers
```

更新：

```bash
gemini extensions update superpowers
```

## 基本工作流

1. **brainstorming** - 在写代码前激活。通过问题细化粗略想法，探索替代方案，分段呈现设计以验证。保存设计文档。

2. **using-git-worktrees** - 设计批准后激活。在新分支上创建隔离 workspace，运行项目设置，验证干净测试基线。

3. **writing-plans** - 有批准设计后激活。将工作分解为 bite-sized 任务（每个 2-5 分钟）。每个任务有精确文件路径、完整代码、验证步骤。

4. **subagent-driven-development** 或 **executing-plans** - 有计划后激活。每个任务派遣新子代理，带两阶段审核（规格合规，然后代码质量），或带人类检查点的批量执行。

5. **test-driven-development** - 实现期间激活。强制 RED-GREEN-REFACTOR：写失败测试，看它失败，写最小代码，看它通过，提交。删除测试前写的代码。

6. **requesting-code-review** - 任务间激活。对计划审核，按严重性报告问题。关键问题阻止进度。

7. **finishing-a-development-branch** - 任务完成时激活。验证测试，呈现选项（merge/PR/keep/discard），清理 worktree。

**Agent 在任何任务前检查相关技能。** 强制工作流，不是建议。

## 包含内容

### 技能库

**测试**
- **test-driven-development** - RED-GREEN-REFACTOR 循环（包含测试反模式参考）

**调试**
- **systematic-debugging** - 4 阶段根因过程（包含 root-cause-tracing、defense-in-depth、condition-based-waiting 技术）
- **verification-before-completion** - 确保确实已修复

**协作**
- **brainstorming** - 苏格拉底式设计细化
- **writing-plans** - 详细实现计划
- **executing-plans** - 带检查点的批量执行
- **dispatching-parallel-agents** - 并发子代理工作流
- **requesting-code-review** - 预审核检查清单
- **receiving-code-review** - 回应反馈
- **using-git-worktrees** - 并行开发分支
- **finishing-a-development-branch** - Merge/PR 决策工作流
- **subagent-driven-development** - 带两阶段审核的快速迭代（规格合规，然后代码质量）

**元**
- **writing-skills** - 按最佳实践创建新技能（包含测试方法论）
- **using-superpowers** - 技能系统介绍

## 哲学

- **测试驱动开发** - 总是先写测试
- **系统性胜过临时性** - 流程胜过猜测
- **复杂度降低** - 简洁作为首要目标
- **证据胜过声明** - 声明成功前验证

阅读 [原始发布公告](https://blog.fsck.com/2025/10/09/superpowers/)。

## 贡献

Superpowers 的一般贡献流程如下。请注意，我们一般不接受新技能的贡献，且任何技能更新必须在我们支持的所有编码 agent 上工作。

1. Fork 仓库
2. 切换到 'dev' 分支
3. 为你的工作创建分支
4. 遵循 `writing-skills` 技能创建和测试新及修改技能
5. 提交 PR，确保填写 pull request template

见 `skills/writing-skills/SKILL.md` 获取完整指南。

## 更新

Superpowers 更新有些依赖于编码 agent，但通常是自动的。

## 许可证

MIT License - 见 LICENSE 文件了解详情

## 社区

Superpowers 由 [Jesse Vincent](https://blog.fsck.com) 和 [Prime Radiant](https://primeradiant.com) 的其他人构建。

- **Discord**: [加入我们](https://discord.gg/35wsABTejz) 获取社区支持、问题和分享你用 Superpowers 构建的内容
- **Issues**: https://github.com/obra/superpowers/issues
- **发布公告**: [注册](https://primeradiant.com/superpowers/) 以获取新版本通知