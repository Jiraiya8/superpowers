# 可视化头脑风暴重构实现计划

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development（如有子代理）或 superpowers:executing-plans 实现此计划。步骤使用 checkbox（`- [ ]`）语法进行追踪。

**目标：** 将可视化头脑风暴从阻塞 TUI 反馈模型重构为非阻塞"浏览器显示，终端命令"架构。

**架构：** 浏览器成为交互式显示；终端保持对话通道。服务器将用户事件写入每个屏幕的 `.events` 文件，Claude 在下一回合读取。消除 `wait-for-feedback.sh` 和所有 `TaskOutput` 阻塞。

**技术栈：** Node.js（Express、ws、chokidar）、vanilla HTML/CSS/JS

**规范：** `docs/superpowers/specs/2026-02-19-visual-brainstorming-refactor-design.md`

---

## 文件映射

| 文件 | 动作 | 职责 |
|------|--------|---------------|
| `lib/brainstorm-server/index.js` | 修改 | 服务器：添加 `.events` 文件写入、新屏幕时清除、替换 `wrapInFrame` |
| `lib/brainstorm-server/frame-template.html` | 修改 | 模板：移除 feedback footer、添加内容占位符 + 选择指示条 |
| `lib/brainstorm-server/helper.js` | 修改 | 客户端 JS：移除 send/feedback 函数、收窄为 click 捕获 + 指示条更新 |
| `lib/brainstorm-server/wait-for-feedback.sh` | 删除 | 不再需要 |
| `skills/brainstorming/visual-companion.md` | 修改 | 技能说明：重写循环为非阻塞流程 |
| `tests/brainstorm-server/server.test.js` | 修改 | 测试：更新为新模板结构和 helper.js API |

---

## Chunk 1: Server、Template、Client、Tests、Skill

### Task 1: 更新 `frame-template.html`

**文件：**
- 修改：`lib/brainstorm-server/frame-template.html`

- [ ] **Step 1: 移除 feedback footer HTML**

用选择指示条替换 feedback-footer div（lines 227-233）：

```html
  <div class="indicator-bar">
    <span id="indicator-text">Click an option above，then return to the terminal</span>
  </div>
```

同时用内容占位符替换 `#claude-content` 内的默认内容（lines 220-223）：

```html
    <div id="claude-content">
      <!-- CONTENT -->
    </div>
```

- [ ] **Step 2: 用指示条 CSS 替换 feedback footer CSS**

移除 `.feedback-footer`、`.feedback-footer label`、`.feedback-row` 和 `.feedback-footer` 内 textarea/button 样式（lines 82-112）。

添加指示条 CSS：

```css
    .indicator-bar {
      background: var(--bg-secondary);
      border-top: 1px solid var(--border);
      padding: 0.5rem 1.5rem;
      flex-shrink: 0;
      text-align: center;
    }
    .indicator-bar span {
      font-size: 0.75rem;
      color: var(--text-secondary);
    }
    .indicator-bar .selected-text {
      color: var(--accent);
      font-weight: 500;
    }
```

- [ ] **Step 3: 验证模板渲染**

运行测试套件检查模板仍能加载：
```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：Tests 1-5 应仍通过。Tests 6-8 可能失败（预期 — 它们断言旧结构）。

- [ ] **Step 4: 提交**

```bash
git add lib/brainstorm-server/frame-template.html
git commit -m "Replace feedback footer with selection indicator bar in brainstorm template"
```

---

### Task 2: 更新 `index.js` — 内容注入和 `.events` 文件

**文件：**
- 修改：`lib/brainstorm-server/index.js`

- [ ] **Step 1: 编写 `.events` 文件写入的失败测试**

在 `tests/brainstorm-server/server.test.js` Test 4 区域后添加 — 新测试发送带 `choice` 字段的 WebSocket event 并验证 `.events` 文件写入：

```javascript
    // Test: Choice events written to .events file
    console.log('Test: Choice events written to .events file');
    const ws3 = new WebSocket(`ws://localhost:${TEST_PORT}`);
    await new Promise(resolve => ws3.on('open'，resolve));

    ws3.send(JSON.stringify({ type: 'click'，choice: 'a'，text: 'Option A' }));
    await sleep(300);

    const eventsFile = path.join(TEST_DIR，'.events');
    assert(fs.existsSync(eventsFile)，'.events file should exist after choice click');
    const lines = fs.readFileSync(eventsFile，'utf-8').trim().split('\n');
    const event = JSON.parse(lines[lines.length - 1]);
    assert.strictEqual(event.choice，'a'，'Event should contain choice');
    assert.strictEqual(event.text，'Option A'，'Event should contain text');
    ws3.close();
    console.log('  PASS');
```

- [ ] **Step 2: 运行测试验证失败**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：新测试失败 — `.events` 文件尚不存在。

- [ ] **Step 3: 编写 `.events` 文件新屏幕清除的失败测试**

添加另一个测试：

```javascript
    // Test: .events cleared on new screen
    console.log('Test: .events cleared on new screen');
    // .events file should still exist from previous test
    assert(fs.existsSync(path.join(TEST_DIR，'.events'))，'.events should exist before new screen');
    fs.writeFileSync(path.join(TEST_DIR，'new-screen.html')，'<h2>New screen</h2>');
    await sleep(500);
    assert(!fs.existsSync(path.join(TEST_DIR，'.events'))，'.events should be cleared after new screen');
    console.log('  PASS');
```

- [ ] **Step 4: 运行测试验证失败**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：新测试失败 — 屏幕推送时 `.events` 未清除。

- [ ] **Step 5: 在 `index.js` 实现 `.events` 文件写入**

在 WebSocket `message` handler（`index.js` line 74-77），`console.log` 后添加：

```javascript
    // Write user events to .events file for Claude to read
    if (event.choice) {
      const eventsFile = path.join(SCREEN_DIR，'.events');
      fs.appendFileSync(eventsFile，JSON.stringify(event) + '\n');
    }
```

在 chokidar `add` handler（line 104-111），添加 `.events` 清除：

```javascript
    if (filePath.endsWith('.html')) {
      // Clear events from previous screen
      const eventsFile = path.join(SCREEN_DIR，'.events');
      if (fs.existsSync(eventsFile)) fs.unlinkSync(eventsFile);

      console.log(JSON.stringify({ type: 'screen-added'，file: filePath }));
      // ... existing reload broadcast
    }
```

- [ ] **Step 6: 用注释占位符注入替换 `wrapInFrame`**

替换 `wrapInFrame` 函数（`index.js` lines 27-32）：

```javascript
function wrapInFrame(content) {
  return frameTemplate.replace('<!-- CONTENT -->'，content);
}
```

- [ ] **Step 7: 运行所有测试**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：新 `.events` 测试通过。现有测试可能仍有旧断言的失败（Task 4 修复）。

- [ ] **Step 8: 提交**

```bash
git add lib/brainstorm-server/index.js tests/brainstorm-server/server.test.js
git commit -m "Add .events file writing and comment-based content injection to brainstorm server"
```

---

### Task 3: 简化 `helper.js`

**文件：**
- 修改：`lib/brainstorm-server/helper.js`

- [ ] **Step 1: 移除 `sendToClaude` 函数**

删除 `sendToClaude` 函数（lines 92-106）— 函数体和页面接管 HTML。

- [ ] **Step 2: 移除 `window.send` 函数**

删除 `window.send` 函数（lines 120-129）— 与移除的 Send button 关联。

- [ ] **Step 3: 移除 form submission 和 input change handlers**

删除 form submission handler（lines 57-71）和 input change handler（lines 73-89），包括 `inputTimeout` 变量。

- [ ] **Step 4: 移除 `pageshow` event listener**

删除之前添加的 `pageshow` listener（无 textarea 需清除了）。

- [ ] **Step 5: 将 click handler 收窄为仅 `[data-choice]`**

用收窄版本替换 click handler（lines 36-55）：

```javascript
  // Capture clicks on choice elements
  document.addEventListener('click'，(e) => {
    const target = e.target.closest('[data-choice]');
    if (!target) return;

    sendEvent({
      type: 'click',
      text: target.textContent.trim(),
      choice: target.dataset.choice,
      id: target.id || null
    });
  });
```

- [ ] **Step 6: 添加选择点击时的指示条更新**

在 click handler 的 `sendEvent` 调用后添加：

```javascript
    // Update indicator bar
    const indicator = document.getElementById('indicator-text');
    if (indicator) {
      const label = target.querySelector('h3，.content h3，.card-body h3')?.textContent?.trim() || target.dataset.choice;
      indicator.innerHTML = '<span class="selected-text">' + label + ' selected</span> — return to terminal to continue';
    }
```

- [ ] **Step 7: 从 `window.brainstorm` API 移除 `sendToClaude`**

更新 `window.brainstorm` 对象（lines 132-136）以移除 `sendToClaude`：

```javascript
  window.brainstorm = {
    send: sendEvent,
    choice: (value，metadata = {}) => sendEvent({ type: 'choice'，value，...metadata })
  };
```

- [ ] **Step 8: 运行测试**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```

- [ ] **Step 9: 提交**

```bash
git add lib/brainstorm-server/helper.js
git commit -m "Simplify helper.js: remove feedback functions，narrow to choice capture + indicator"
```

---

### Task 4: 更新测试为新结构

**文件：**
- 修改：`tests/brainstorm-server/server.test.js`

**注意：** 下面行引用来自 _原始_ 文件。Task 2 在文件前面插入了新测试，所以实际行号会偏移。按 `console.log` 标签查找测试（如 "Test 5:"、"Test 6:"）。

- [ ] **Step 1: 更新 Test 5（完整文档断言）**

找到 Test 5 断言 `!fullRes.body.includes('feedback-footer')`。改为：完整文档不应有指示条（它们直接服务）：

```javascript
    assert(!fullRes.body.includes('indicator-bar') || fullDoc.includes('indicator-bar'),
      'Should not wrap full documents in frame template');
```

- [ ] **Step 2: 更新 Test 6（fragment wrapping）**

Line 125：用指示条断言替换 `feedback-footer` 断言：

```javascript
    assert(fragRes.body.includes('indicator-bar')，'Fragment should get indicator bar from frame');
```

同时验证内容占位符已替换（fragment 内容出现，占位符注释不存在）：

```javascript
    assert(!fragRes.body.includes('<!-- CONTENT -->')，'Content placeholder should be replaced');
```

- [ ] **Step 3: 更新 Test 7（helper.js API）**

Lines 140-142：更新断言反映新 API surface：

```javascript
    assert(helperContent.includes('toggleSelect')，'helper.js should define toggleSelect');
    assert(helperContent.includes('sendEvent')，'helper.js should define sendEvent');
    assert(helperContent.includes('selectedChoice')，'helper.js should track selectedChoice');
    assert(helperContent.includes('brainstorm')，'helper.js should expose brainstorm API');
    assert(!helperContent.includes('sendToClaude')，'helper.js should not contain sendToClaude');
```

- [ ] **Step 4: 用指示条测试替换 Test 8（sendToClaude theming）**

替换 Test 8（lines 145-149）— `sendToClaude` 不存在了。测试指示条：

```javascript
    // Test 8: Indicator bar uses CSS variables（theme support）
    console.log('Test 8: Indicator bar uses CSS variables');
    const templateContent = fs.readFileSync(
      path.join(__dirname，'../../lib/brainstorm-server/frame-template.html')，'utf-8'
    );
    assert(templateContent.includes('indicator-bar')，'Template should have indicator bar');
    assert(templateContent.includes('indicator-text')，'Template should have indicator text element');
    console.log('  PASS');
```

- [ ] **Step 5: 运行完整测试套件**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：所有测试通过。

- [ ] **Step 6: 提交**

```bash
git add tests/brainstorm-server/server.test.js
git commit -m "Update brainstorm server tests for new template structure and helper.js API"
```

---

### Task 5: 删除 `wait-for-feedback.sh`

**文件：**
- 删除：`lib/brainstorm-server/wait-for-feedback.sh`

- [ ] **Step 1: 验证无其他文件导入或引用 `wait-for-feedback.sh`**

搜索代码库：
```bash
grep -r "wait-for-feedback" /Users/drewritter/prime-rad/superpowers/ --include="*.js" --include="*.md" --include="*.sh" --include="*.json"
```

预期引用：仅 `visual-companion.md`（Task 6 重写）和可能的 release notes（历史性，保留原样）。

- [ ] **Step 2: 删除文件**

```bash
rm lib/brainstorm-server/wait-for-feedback.sh
```

- [ ] **Step 3: 运行测试确认无破坏**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：所有测试通过（无测试引用此文件）。

- [ ] **Step 4: 提交**

```bash
git add -u lib/brainstorm-server/wait-for-feedback.sh
git commit -m "Delete wait-for-feedback.sh: replaced by .events file"
```

---

### Task 6: 重写 `visual-companion.md`

**文件：**
- 修改：`skills/brainstorming/visual-companion.md`

- [ ] **Step 1: 更新 "How It Works" 描述（line 18）**

替换关于接收反馈"作为 JSON"的句子：

```markdown
服务器监视目录中的 HTML 文件并将最新的服务到浏览器。你写入 HTML 内容，用户在浏览器中查看并可点击选择选项。选择记录到 `.events` 文件，你在下一回合读取。
```

- [ ] **Step 2: 更新 fragment 描述（line 20）**

从 frame 模板提供的内容描述中移除 "feedback footer"：

```markdown
**Content fragments vs full documents：** 如果 HTML 文件以 `<!DOCTYPE` 或 `<html` 开头，服务器直接服务（仅注入 helper script）。否则，服务器自动在 frame 模板中包装内容 — 添加 header、CSS theme、选择指示条和所有交互基础设施。**默认写内容片段。** 只有需要完全控制页面时才写完整文档。
```

- [ ] **Step 3: 重写 "The Loop" 部分（lines 36-61）**

用以下内容替换整个 "The Loop" 部分：

```markdown
## The Loop

1. **写 HTML** 到 `screen_dir` 的新文件：
   - 使用语义化文件名：`platform.html`、`visual-style.html`、`layout.html`
   - **从不复用文件名** — 每个屏幕获得新文件
   - 使用 Write tool — **从不使用 cat/heredoc**（向终端输出噪音）
   - 服务器自动服务最新文件

2. **告诉用户期待什么并结束回合：**
   - 每步提醒 URL（不只是首次）
   - 给屏幕内容简要文本总结（如 "Showing 3 layout options for the homepage"）
   - 请他们在终端响应："Take a look and let me know what you think。Click to select an option if you'd like。"

3. **下一回合** — 用户在终端响应后：
   - 如 `$SCREEN_DIR/.events` 存在则读取 — 包含用户浏览器交互（点击、选择）作为 JSON 行
   - 与用户终端文本合并获取完整情况
   - 终端消息是主要反馈；`.events` 提供结构化交互数据

4. **迭代或推进** — 如果反馈改变当前屏幕，写新文件（如 `layout-v2.html`）。当前步骤验证后才进入下一个问题。

5. 重复直到完成。
```

- [ ] **Step 4: 替换 "User Feedback Format" 部分（lines 165-174）**

替换为：

```markdown
## Browser Events Format

用户在浏览器点击选项时，交互记录到 `$SCREEN_DIR/.events`（每行一个 JSON 对象）。推送新屏幕时文件自动清除。

```jsonl
{"type":"click"，"choice":"a"，"text":"Option A - Simple Layout"，"timestamp":1706000101}
{"type":"click"，"choice":"c"，"text":"Option C - Complex Grid"，"timestamp":1706000108}
{"type":"click"，"choice":"b"，"text":"Option B - Hybrid"，"timestamp":1706000115}
```

完整事件流显示用户探索路径 — 他们可能点击多个选项后才确定。最后一个 `choice` 事件通常是最终选择，但点击模式可能揭示犹豫或偏好值得询问。

如果 `.events` 不存在，用户未与浏览器交互 — 仅使用终端文本。
```

- [ ] **Step 5: 更新 "Writing Content Fragments" 描述（line 65）**

移除 "feedback footer" 引用：

```markdown
只写放入页面的内容。服务器自动在 frame 模板中包装（header、theme CSS、选择指示条和所有交互基础设施）。
```

- [ ] **Step 6: 更新 Reference 部分（lines 200-203）**

移除关于 "JS API" 的 helper.js 引用描述 — API 现在最小。保留路径引用：

```markdown
## Reference

- Frame template（CSS reference）： `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/frame-template.html`
- Helper script（client-side）： `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/helper.js`
```

- [ ] **Step 7: 提交**

```bash
git add skills/brainstorming/visual-companion.md
git commit -m "Rewrite visual-companion.md for non-blocking browser-displays-terminal-commands flow"
```

---

### Task 7: 最终验证

- [ ] **Step 1: 运行完整测试套件**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：所有测试通过。

- [ ] **Step 2: 手动冒烟测试**

手动启动服务器并验证流程端到端工作：

```bash
cd /Users/drewritter/prime-rad/superpowers && lib/brainstorm-server/start-server.sh --project-dir /tmp/brainstorm-smoke-test
```

写测试 fragment，浏览器打开，点击选项，验证 `.events` 文件写入，验证指示条更新。然后停止服务器：

```bash
lib/brainstorm-server/stop-server.sh <screen_dir from start output>
```

- [ ] **Step 3: 验证无陈旧引用残留**

```bash
grep -r "wait-for-feedback\|sendToClaude\|feedback-footer\|send-to-claude\|TaskOutput.*block.*true" /Users/drewritter/prime-rad/superpowers/ --include="*.js" --include="*.md" --include="*.sh" --include="*.html" | grep -v node_modules | grep -v RELEASE-NOTES | grep -v "\.md:.*spec\|plan"
```

预期：release notes 和 spec/plan 文档外无命中（它们是历史性）。

- [ ] **Step 4: 如需清理则最终提交**

```bash
git status
# Review untracked/modified files，stage specific files as needed，commit if clean
```