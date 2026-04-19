# 可视化头脑风暴重构：浏览器显示，终端命令

**日期：** 2026-02-19
**状态：** 已批准
**范围：** `lib/brainstorm-server/`、`skills/brainstorming/visual-companion.md`、`tests/brainstorm-server/`

## 问题

可视化头脑风暴期间，Claude 将 `wait-for-feedback.sh` 作为后台任务运行并在 `TaskOutput(block=true，timeout=600s)` 上阻塞。这完全占用 TUI — 用户无法在可视化头脑风暴运行时向 Claude 输入。浏览器成为唯一的输入通道。

Claude Code 的执行模型是回合制的。单个回合内 Claude 无法同时监听两个通道。阻塞 `TaskOutput` 模式是错误的原始操作 — 它模拟了平台不支持的事件驱动行为。

## 设计

### 核心模型

**浏览器 = 交互式显示。** 展示原型，让用户点击选择选项。选择在服务端记录。

**终端 = 对话通道。** 总是解锁，总是可用。用户在这里与 Claude 交流。

### 循环

1. Claude 将 HTML 文件写入会话目录
2. 服务器通过 chokidar 检测，向浏览器推送 WebSocket reload（不变）
3. Claude 结束回合 — 告诉用户检查浏览器并在终端响应
4. 用户查看浏览器，可选点击选择选项，然后在终端输入反馈
5. 下一回合，Claude 读取 `$SCREEN_DIR/.events` 获取浏览器交互流（点击、选择），与终端文本合并
6. 迭代或推进

无后台任务。无 `TaskOutput` 阻塞。无轮询脚本。

### 关键删除：`wait-for-feedback.sh`

完全删除。其目的是桥接"服务器记录事件到 stdout"和"Claude 需接收这些事件"。`.events` 文件替代此功能 — 服务器直接写入用户交互事件，Claude 用平台提供的任何文件读取机制读取。

### 关键新增：`.events` 文件（每个屏幕的事件流）

服务器将所有用户交互事件写入 `$SCREEN_DIR/.events`，每行一个 JSON 对象。这给 Claude 当前屏幕的完整交互流 — 不只是最终选择，而是用户的探索路径（点击 A，然后 B，确定 C）。

用户探索选项后的示例内容：

```jsonl
{"type":"click","choice":"a","text":"Option A - Preset-First Wizard","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Manual Config","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid Approach","timestamp":1706000115}
```

- 屏幕内只追加。每个用户事件作为新行追加。
- chokidar 检测到新 HTML 文件（新屏幕推送）时文件被清除（删除），防止陈旧事件延续。
- Claude 读取时文件不存在则无浏览器交互 — Claude 仅使用终端文本。
- 文件仅包含用户事件（`click` 等）— 不是服务器生命周期事件（`server-started`、`screen-added`）。保持小而专注。
- Claude 可读取完整流理解用户探索模式，或只看最后一个 `choice` 事件获取最终选择。

## 按文件变更

### `index.js`（服务器）

**A. 将用户事件写入 `.events` 文件。**

在 WebSocket `message` handler 中，记录事件到 stdout 后：通过 `fs.appendFileSync` 将事件作为 JSON 行追加到 `$SCREEN_DIR/.events`。仅写入用户交互事件（带 `source: 'user-event'` 的），不是服务器生命周期事件。

**B. 新屏幕时清除 `.events`。**

在 chokidar `add` handler（检测到新 `.html` 文件）中，如 `$SCREEN_DIR/.events` 存在则删除。这是明确的"新屏幕"信号 — 比在每次 reload 触发的 GET `/` 上清除更好。

**C. 替换 `wrapInFrame` 内容注入。**

当前正则锚定在 `<div class="feedback-footer">`，该元素正被移除。改为注释占位符：删除 `#claude-content` 内现有默认内容（`<h2>Visual Brainstorming</h2>` 和副标题段落），替换为单个 `<!-- CONTENT -->` 标记。内容注入变为 `frameTemplate.replace('<!-- CONTENT -->'，content)`。更简单且模板格式变更时不会破坏。

### `frame-template.html`（UI frame）

**移除：**
- `feedback-footer` div（textarea、Send button、label、`.feedback-row`）
- 相关 CSS（`.feedback-footer`、`.feedback-footer label`、`.feedback-row`、其中的 textarea 和 button 样式）

**添加：**
- `#claude-content` 内 `<!-- CONTENT -->` 占位符，替代默认文本
- footer 位置的选择指示条，两个状态：
  - 默认："Click an option above，then return to the terminal"
  - 选择后："Option B selected — return to terminal to continue"
- 指示条 CSS（微妙，与现有 header 相似的视觉权重）

**保持不变：**
- 带 "Brainstorm Companion" 标题和连接状态的 header bar
- `.main` wrapper 和 `#claude-content` container
- 所有组件 CSS（`.options`、`.cards`、`.mockup`、`.split`、`.pros-cons`、placeholders、mock 元素）
- Dark/light theme 变量和 media query

### `helper.js`（客户端脚本）

**移除：**
- `sendToClaude()` 函数和 "Sent to Claude" 页面接管
- `window.send()` 函数（与移除的 Send button 关联）
- Form submission handler — 无 feedback textarea 后无用途，增加日志噪音
- Input change handler — 同理
- `pageshow` event listener（之前添加以修复 textarea 持久 — 无 textarea 了）

**保留：**
- WebSocket connection、reconnect logic、event queue
- Reload handler（服务器推送时 `window.location.reload()`）
- `window.toggleSelect()` 用于选择高亮
- `window.selectedChoice` tracking
- `window.brainstorm.send()` 和 `window.brainstorm.choice()` — 这些与移除的 `window.send()` 不同。它们调用 `sendEvent` 通过 WebSocket 记录到服务器。对自定义完整文档页面有用。

**收窄：**
- Click handler：仅捕获 `[data-choice]` 点击，不是所有 button/link。当浏览器是反馈通道时需要广泛捕获；现在仅用于选择追踪。

**添加：**
- `data-choice` 点击时，更新选择指示条文本显示选择的选项。

**从 `window.brainstorm` API 移除：**
- `brainstorm.sendToClaude` — 不再存在

### `visual-companion.md`（技能说明）

**重写 "The Loop" 部分** 为上述非阻塞流程。移除所有关于：
- `wait-for-feedback.sh`
- `TaskOutput` blocking
- Timeout/retry 逻辑（600s timeout、30-minute cap）
- 描述 `send-to-claude` JSON 的 "User Feedback Format" 部分

**替换为：**
- 新循环（write HTML → end turn → 用户在终端响应 → read `.events` → iterate）
- `.events` 文件格式文档
- 终端消息是主要反馈；`.events` 提供完整浏览器交互流作为额外上下文的指导

**保留：**
- Server startup/shutdown 说明
- Content fragment vs full document 指导
- CSS class 参考和可用组件
- 设计提示（scale fidelity to the question、每屏 2-4 选项等）

### `wait-for-feedback.sh`

**完全删除。**

### `tests/brainstorm-server/server.test.js`

需要更新的测试：
- 断言 fragment 响应中 `feedback-footer` 存在的测试 — 改为断言选择指示条或 `<!-- CONTENT -->` 替换
- 断言 `helper.js` 包含 `send` 的测试 — 更新以反映收窄的 API
- 断言 `sendToClaude` CSS 变量使用的测试 — 移除（函数不存在）

## 平台兼容性

服务器代码（`index.js`、`helper.js`、`frame-template.html`）完全平台无关 — 纯 Node.js 和浏览器 JavaScript。无 Claude Code 特定引用。已通过后台终端交互在 Codex 上验证工作。

技能说明（`visual-companion.md`）是平台适配层。每个平台的 Claude 使用自己的工具启动服务器、读取 `.events` 等。非阻塞模型在各平台自然工作，因为不依赖任何平台特定的阻塞原始操作。

## 启用的能力

- **TUI 在可视化头脑风暴期间总是响应**
- **混合输入** — 浏览器点击 + 终端输入，自然合并
- **优雅降级** — 浏览器宕机或用户不打开？终端仍工作
- **更简单架构** — 无后台任务、无轮询脚本、无超时管理
- **跨平台** — 相同服务器代码在 Claude Code、Codex 和任何未来平台工作

## 失去的能力

- **纯浏览器反馈工作流** — 用户必须回到终端继续。选择指示条引导他们，但比旧的点击 Send-and-wait 流多一步。
- **浏览器内嵌文本反馈** — textarea 消失了。所有文本反馈通过终端。这是有意为之 — 终端是比 frame 中小 textarea 更好的文本输入通道。
- **浏览器 Send 后立即响应** — 旧系统用户点击 Send 时 Claude 立即响应。现在用户切换到终端时有间隙。实践中是几秒，且用户可以在终端消息中添加上下文。