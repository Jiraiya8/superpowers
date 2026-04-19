# 用户反馈驱动的技能改进

**日期：** 2025-11-28
**状态：** Draft
**来源：** 两个在实际开发场景中使用 superpowers 的 Claude 实例

---

## 执行摘要

两个 Claude 实例提供了来自实际开发会话的详细反馈。他们的反馈揭示了现有技能中的 **系统性缺口**，导致可预防的 bug 尽管遵循技能仍被发布。

**关键洞察：** 这些是问题报告，不只是解决方案提案。问题是真实的；解决方案需要仔细评估。

**关键主题：**
1. **验证缺口** — 我们验证操作成功但不验证达到预期结果
2. **进程卫生** — 后台进程积累并在子代理间干扰
3. **上下文优化** — 子代理获得太多无关信息
4. **自检缺失** — 移交前无审视自己工作的 prompt
5. **Mock 安全** — Mock 可能偏离接口而不被发现
6. **技能激活** — 技能存在但未被读取/使用

---

## 发现的问题

### 问题 1：配置变更验证缺口

**发生情况：**
- 子代理测试 "OpenAI integration"
- 设置 `OPENAI_API_KEY` env var
- 获得 status 200 响应
- 报告 "OpenAI integration working"
- **但** 响应包含 `"model": "claude-sonnet-4-20250514"` — 实际使用的是 Anthropic

**根本原因：**
`verification-before-completion` 检查操作成功但不检查结果反映预期的配置变更。

**影响：** 高 — 集成测试的虚假信心，bug 发布到生产

**示例失败模式：**
- 切换 LLM provider → 验证 status 200 但不检查 model name
- 启用 feature flag → 验证无错误但不检查 feature 活跃
- 更换环境 → 验证部署成功但不检查环境变量

---

### 问题 2：后台进程积累

**发生情况：**
- 会话期间分发多个子代理
- 每个启动后台服务器进程
- 进程积累（4+ 服务器运行）
- 陈旧进程仍绑定端口
- 后续 E2E 测试命中带错误配置的陈旧服务器
- 混乱/错误的测试结果

**根本原因：**
子代理无状态 — 不知道之前子代理的进程。无清理协议。

**影响：** 中高 — 测试命中错误服务器，虚假通过/失败，调试混乱

---

### 问题 3：子代理 Prompt 中的上下文膨胀

**发生情况：**
- 标准方法：给子代理完整计划文件读取
- 实验：仅给任务 + pattern + file + verify command
- 结果：更快、更专注、单次尝试完成更常见

**根本原因：**
子代理浪费 token 和注意力在无关计划部分。

**影响：** 中 — 执行更慢，更多失败尝试

**有效的做法：**
```
你正在向 packnplay 的测试套件添加一个 E2E 测试。

**你的任务：** 将 `TestE2E_FeaturePrivilegedMode` 添加到 `pkg/runner/e2e_test.go`

**测试内容：** 一个请求 `"privileged": true` 的本地 devcontainer feature
应导致容器以 `--privileged` 标志运行。

**遵循 TestE2E_FeatureOptionValidation 的确切模式**（文件末尾）

**编写后运行：** `go test -v ./pkg/runner -run TestE2E_FeaturePrivilegedMode -timeout 5m`
```

---

### 问题 4：移交前无自检

**发生情况：**
- 添加自检 prompt："Look at your work with fresh eyes - what could be better?"
- Task 5 的实现者发现失败测试是实现 bug，不是测试 bug
- 追踪到 line 99：`strings.Join(metadata.Entrypoint，" ")` 创建无效 Docker syntax
- 无自检，只会报告 "test fails" 而无根本原因

**根本原因：**
实现者自然不会在报告完成前退一步审视自己的工作。

**影响：** 中 — Bug 移交给审查者，实现者本可发现

---

### 问题 5：Mock-Interface 漂移

**发生情况：**
```typescript
// Interface 定义 close()
interface PlatformAdapter {
  close(): Promise<void>;
}

// Code（BUGGY）调用 cleanup()
await adapter.cleanup();

// Mock（MATCHES BUG）定义 cleanup()
vi.mock('web-adapter'，() => ({
  WebAdapter: vi.fn().mockImplementation(() => ({
    cleanup: vi.fn().mockResolvedValue(undefined)，// Wrong!
  })),
}));
```
- 测试通过
- Runtime 崩溃："adapter.cleanup is not a function"

**根本原因：**
Mock 源自 buggy 代码调用什么，不是接口定义。TypeScript 无法捕获 inline mock 中的错误方法名。

**影响：** 高 — 测试虚假信心，runtime 崩溃

**为什么 testing-anti-patterns 未防止此问题：**
技能覆盖测试 mock 行为和无理解的 mock，但不是"从接口而非实现派生 mock"的特定模式。

---

### 问题 6：代码审查者文件访问

**发生情况：**
- 分发代码审查子代理
- 找不到测试文件："The file doesn't appear to exist in the repository"
- 文件实际存在
- 审查者不知道要先显式读取

**根本原因：**
审查者 prompt 未包含显式文件读取指令。

**影响：** 低中 — 审查失败或不完整

---

### 问题 7：修复工作流延迟

**发生情况：**
- 实现者在自检时发现 bug
- 实现者知道修复方案
- 当前工作流：报告 → 我分发 fixer → fixer 修复 → 我验证
- 额外往返增加延迟无增值

**根本原因：**
当实现者已诊断时，实现者和 fixer 角色刚性分离。

**影响：** 低 — 延迟，但无正确性问题

---

### 问题 8：技能未被读取

**发生情况：**
- `testing-anti-patterns` 技能存在
- 人类和子代理在写测试前都不读取
- 本可防止一些问题（但非所有 — 见问题 5）

**根本原因：**
无强制子代理读取相关技能。无 prompt 包含技能读取。

**影响：** 中 — 技能投资浪费如不被使用

---

## 建议改进

### 1. verification-before-completion：添加配置变更验证

**添加新部分：**

```markdown
## 验证配置变更

当测试对配置、provider、feature flag 或环境的变更：

**不要只验证操作成功。验证输出反映预期变更。**

### 常见失败模式

操作成功因为存在 *某些* 有效配置，但不是你预期测试的配置。

### 示例

| 变更 | 不足 | 必需 |
|--------|-------------|----------|
| 切换 LLM provider | Status 200 | 响应包含预期 model name |
| 启用 feature flag | 无错误 | Feature 行为实际活跃 |
| 更换环境 | Deploy 成功 | Logs/vars 引用新环境 |
| 设置 credentials | Auth 成功 | 认证用户/context 正确 |

### Gate 函数

```
声明配置变更工作前：

1. IDENTIFY：此变更后应有何不同？
2. LOCATE：该差异在哪可观察？
   - Response 字段（model name，user ID）
   - Log 行（environment，provider）
   - 行为（feature active/inactive）
3. RUN：显示可观察差异的命令
4. VERIFY：输出包含预期差异
5. ONLY THEN：声明配置变更工作

警示信号：
   - "Request succeeded" 未检查内容
   - 检查 status code 但不检查 response body
   - 验证无错误但无正面确认
```

**为什么有效：**
强制验证 INTENT，不只是操作成功。

---

### 2. subagent-driven-development：为 E2E 测试添加进程卫生

**添加新部分：**

```markdown
## E2E 测试进程卫生

当分发启动服务（server、database、message queue）的子代理：

### 问题

子代理无状态 — 不知道之前子代理启动的进程。后台进程持续并可能干扰后续测试。

### 解决方案

**分发 E2E 测试子代理前，在 prompt 包含清理：**

```
启动任何服务前：
1. Kill existing processes: pkill -f "<service-pattern>" 2>/dev/null || true
2. Wait for cleanup: sleep 1
3. Verify port free: lsof -i :<port> && echo "ERROR: Port still in use" || echo "Port free"

测试完成后：
1. Kill the process you started
2. Verify cleanup: pgrep -f "<service-pattern>" || echo "Cleanup successful"
```

### 示例

```
Task: Run E2E test of API server

Prompt includes:
"Before starting the server:
- Kill any existing servers: pkill -f 'node.*server.js' 2>/dev/null || true
- Verify port 3001 is free: lsof -i :3001 && exit 1 || echo 'Port available'

After tests:
- Kill the server you started
- Verify: pgrep -f 'node.*server.js' || echo 'Cleanup verified'"
```

### 为什么重要

- 陈旧进程以错误配置服务请求
- 端口冲突导致静默失败
- 进程积累减慢系统
- 混乱测试结果（命中错误服务器）
```

**权衡分析：**
- 向 prompt 添加样板
- 但防止非常混乱的调试
- 对 E2E 测试子代理值得

---

### 3. subagent-driven-development：添加精益上下文选项

**修改 Step 2: Execute Task with Subagent**

**之前：**
```
Read that task carefully from [plan-file]。
```

**之后：**
```
## Context Approaches

**Full Plan（默认）:**
Use when tasks are complex or have dependencies:
```
Read Task N from [plan-file] carefully。
```

**Lean Context（用于独立任务）:**
Use when task is standalone and pattern-based:
```
你正在实现：[1-2 句任务描述]

File to modify: [exact path]
Pattern to follow: [reference to existing function/test]
What to implement: [specific requirement]
Verification: [exact command to run]

[Do NOT include full plan file]
```

**Use lean context when:**
- Task follows existing pattern（add similar test，implement similar feature）
- Task is self-contained（doesn't need context from other tasks）
- Pattern reference is sufficient（e.g.，"follow TestE2E_FeatureOptionValidation"）

**Use full plan when:**
- Task has dependencies on other tasks
- Requires understanding of overall architecture
- Complex logic that needs context
```

**示例：**
```
精益上下文 prompt:

"你正在为 devcontainer features 的特权模式添加测试。

文件：pkg/runner/e2e_test.go
模式：遵循 TestE2E_FeatureOptionValidation（文件末尾）
测试：元数据中有 `"privileged": true` 的 feature 导致 `--privileged` 标志
验证：go test -v ./pkg/runner -run TestE2E_FeaturePrivilegedMode -timeout 5m

报告：实现、测试结果、任何问题。"
```

**为什么有效：**
减少 token 使用，增加专注，合适时更快完成。

---

### 4. subagent-driven-development：添加自检步骤

**修改 Step 2: Execute Task with Subagent**

**添加到 prompt 模板：**

```
完成时，报告前：

退一步用新鲜眼光审视你的工作。

问自己：
- 这实际解决了指定的任务吗？
- 有我没考虑的边界情况吗？
- 我正确遵循模式了吗？
- 如果测试失败，ROOT CAUSE 是什么（实现 bug vs 测试 bug）？
- 这个实现哪里可以更好？

如果自检发现问题，现在修复。

然后报告：
- 你实现了什么
- 自检发现（如有）
- 测试结果
- 变更的文件
```

**为什么有效：**
在移交前捕获实现者自己能发现的 bug。文档案例：通过自检发现 entrypoint bug。

**权衡：**
每任务约增加 30 秒，但在审查前捕获问题。

---

### 5. requesting-code-review：添加显式文件读取

**修改 code-reviewer 模板：**

**开头添加：**

```markdown
## Files to Review

分析前，读取这些文件：

1. [List specific files that changed in the diff]
2. [Files referenced by changes but not modified]

Use Read tool to load each file。

如果找不到文件：
- Check exact path from diff
- Try alternate locations
- Report: "Cannot locate [path] - please verify file exists"

DO NOT proceed with review until you've read the actual code。
```

**为什么有效：**
显式指令防止 "file not found" 问题。

---

### 6. testing-anti-patterns：添加 Mock-Interface 漂移反模式

**添加新 Anti-Pattern 6：**

```markdown
## Anti-Pattern 6: Mocks Derived from Implementation

**违规：**
```typescript
// Code（BUGGY）calls cleanup()
await adapter.cleanup();

// Mock（MATCHES BUG）has cleanup()
const mock = {
  cleanup: vi.fn().mockResolvedValue(undefined)
};

// Interface（CORRECT）defines close()
interface PlatformAdapter {
  close(): Promise<void>;
}
```

**为什么错误：**
- Mock 将 bug 编码进测试
- TypeScript 无法捕获 inline mock 中错误方法名
- 测试通过因为代码和 mock 都错
- 使用真实对象时 runtime 崩溃

**修复：**
```typescript
// ✅ GOOD: Derive mock from interface

// Step 1: Open interface definition（PlatformAdapter）
// Step 2: List methods defined there（close，initialize，etc。）
// Step 3: Mock EXACTLY those methods

const mock = {
  initialize: vi.fn().mockResolvedValue(undefined),
  close: vi.fn().mockResolvedValue(undefined)，// From interface!
};

// Now test FAILS because code calls cleanup() which doesn't exist
// That failure reveals the bug BEFORE runtime
```

### Gate 函数

```
写任何 mock 前：

  1. STOP - Do NOT look at the code under test yet
  2. FIND: The interface/type definition for the dependency
  3. READ: The interface file
  4. LIST: Methods defined in the interface
  5. MOCK: ONLY those methods with EXACTLY those names
  6. DO NOT: Look at what your code calls

  IF your test fails because code calls something not in mock:
    ✅ GOOD - The test found a bug in your code
    Fix the code to call the correct interface method
    NOT the mock

  Red flags:
    - "I'll mock what the code calls"
    - Copying method names from implementation
    - Mock written without reading interface
    - "The test is failing so I'll add this method to the mock"
```

**检测：**

当看到 runtime error "X is not a function" 且测试通过：
1. Check if X is mocked
2. Compare mock methods to interface methods
3. Look for method name mismatches
```

**为什么有效：**
直接解决反馈中的失败模式。

---

### 7. subagent-driven-development：要求测试子代理读取技能

**当任务涉及测试时添加到 prompt 模板：**

```markdown
写任何测试前：

1. Read testing-anti-patterns skill:
   Use Skill tool: superpowers:testing-anti-patterns

2. Apply gate functions from that skill when:
   - Writing mocks
   - Adding methods to production classes
   - Mocking dependencies

This is NOT optional。Tests that violate anti-patterns will be rejected in review。
```

**为什么有效：**
确保技能实际使用，不只存在。

**权衡：**
每任务增加时间，但防止整类 bug。

---

### 8. subagent-driven-development：允许实现者修复自发现问题

**修改 Step 2：**

**当前：**
```
Subagent reports back with summary of work。
```

**建议：**
```
Subagent performs self-reflection，then:

IF self-reflection identifies fixable issues:
  1. Fix the issues
  2. Re-run verification
  3. Report: "Initial implementation + self-reflection fix"

ELSE:
  Report: "Implementation complete"

Include in report:
- Self-reflection findings
- Whether fixes were applied
- Final verification results
```

**为什么有效：**
当实现者已知道修复时减少延迟。文档案例：为 entrypoint bug 节省一次往返。

**权衡：**
稍复杂 prompt，但更快端到端。

---

## 实现计划

### Phase 1：高影响，低风险（先做）

1. **verification-before-completion：配置变更验证**
   - 清晰添加，不改变现有内容
   - 解决高影响问题（测试虚假信心）
   - 文件：`skills/verification-before-completion/SKILL.md`

2. **testing-anti-patterns：Mock-interface 漂移**
   - 添加新反模式，不修改现有
   - 解决高影响问题（runtime 崩溃）
   - 文件：`skills/testing-anti-patterns/SKILL.md`

3. **requesting-code-review：显式文件读取**
   - 模板简单添加
   - 修复具体问题（审查者找不到文件）
   - 文件：`skills/requesting-code-review/SKILL.md`

### Phase 2：中度变更（仔细测试）

4. **subagent-driven-development：进程卫生**
   - 添加新部分，不改变工作流
   - 解决中高影响（测试可靠性）
   - 文件：`skills/subagent-driven-development/SKILL.md`

5. **subagent-driven-development：自检**
   - 改变 prompt 模板（更高风险）
   - 但文档证明能捕获 bug
   - 文件：`skills/subagent-driven-development/SKILL.md`

6. **subagent-driven-development：技能读取要求**
   - 添加 prompt 开销
   - 但确保技能实际使用
   - 文件：`skills/subagent-driven-development/SKILL.md`

### Phase 3：优化（先验证）

7. **subagent-driven-development：精益上下文选项**
   - 添加复杂度（两种方法）
   - 需验证不会导致混乱
   - 文件：`skills/subagent-driven-development/SKILL.md`

8. **subagent-driven-development：允许实现者修复**
   - 改变工作流（更高风险）
   - 优化，不是 bug 修复
   - 文件：`skills/subagent-driven-development/SKILL.md`

---

## 开放问题

1. **精益上下文方法：**
   - 是否应为基于模式任务的默认？
   - 如何决定使用哪种方法？
   - 太精益而缺失重要上下文的风险？

2. **自检：**
   - 是否显著减慢简单任务？
   - 是否只应用于复杂任务？
   - 如何防止变成机械的"反思疲劳"？

3. **进程卫生：**
   - 应在 subagent-driven-development 还是单独技能？
   - 是否适用于 E2E 测试外的其他工作流？
   - 如何处理进程应持续的情况（dev servers）？

4. **技能读取强制：**
   - 是否要求所有子代理读取相关技能？
   - 如何保持 prompt 不太长？
   - 过度文档化失去专注的风险？

---

## 成功指标

如何知道改进有效？

1. **配置验证：**
   - "测试通过但用了错误配置" 的实例为零
   - Jesse 不说 "that's not actually testing what you think"

2. **进程卫生：**
   - "测试命中错误服务器" 的实例为零
   - E2E 测试运行期间无端口冲突错误

3. **Mock-interface 漂移：**
   - "测试通过但 runtime 因缺失方法崩溃" 的实例为零
   - Mock 和接口间无方法名不匹配

4. **自检：**
   - 可测量：实现者报告是否包含自检发现？
   - 定性：更少 bug 到达代码审查？

5. **技能读取：**
   - 子代理报告引用技能 gate 函数
   - 代码审查中更少反模式违规

---

## 风险和缓解

### 风险：Prompt 膨胀
**问题：** 添加所有要求使 prompt 压倒性
**缓解：**
- 分阶段实现（不一次添加所有）
- 使某些添加条件化（E2E 卫生仅用于 E2E 测试）
- 考虑不同任务类型的模板

### 风险：分析瘫痪
**问题：** 太多反思/验证减慢执行
**缓解：**
- 保持 gate 函数快（秒级，不是分钟）
- 初期使精益上下文可选
- 监控任务完成时间

### 风险：虚假安全感
**问题：** 遵循清单不保证正确性
**缓解：**
- 强调 gate 函数是最低要求，不是最高
- 在技能中保留"使用判断"语言
- 文档技能捕获常见失败，不是所有失败

### 风险：技能分歧
**问题：** 不同技能给出冲突建议
**缓解：**
- 审查所有技能变更确保一致性
- 文档技能如何交互（Integration 部分）
- 部署前用真实场景测试

---

## 建议

**立即执行 Phase 1：**
- verification-before-completion：配置变更验证
- testing-anti-patterns：Mock-interface 漂移
- requesting-code-review：显式文件读取

**与 Jesse 测试 Phase 2 后定稿：**
- 获取自检影响反馈
- 验证进程卫生方法
- 确认技能读取要求值得开销

**待验证后暂缓 Phase 3：**
- 精益上下文需要真实世界测试
- 实现者修复工作流变更需仔细评估

这些变更解决用户文档的真实问题，同时最小化使技能变差的风险。