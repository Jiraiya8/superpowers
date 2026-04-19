# 根本原因追踪

## 概述

Bug 经常出现在调用栈深处（在错误的目录执行 git init、文件创建在错误的位置、数据库使用了错误的路径打开）。你的直觉是修复错误出现的地方，但这只是治标不治本。

**核心原则：**沿着调用链向后追踪，直到找到原始触发点，然后在源头修复。

## 何时使用

```dot
digraph when_to_use {
    "Bug 出现在堆栈深处？" [shape=diamond];
    "可以向后追踪吗？" [shape=diamond];
    "在症状点修复" [shape=box];
    "追踪到原始触发点" [shape=box];
    "更好：同时添加深度防御" [shape=box];

    "Bug 出现在堆栈深处？" -> "可以向后追踪吗？" [label="是"];
    "可以向后追踪吗？" -> "追踪到原始触发点" [label="是"];
    "可以向后追踪吗？" -> "在症状点修复" [label="否 - 死路"];
    "追踪到原始触发点" -> "更好：同时添加深度防御";
}
```

**使用场景：**
- 错误发生在执行深处（不是入口点）
- 堆栈跟踪显示很长的调用链
- 不清楚无效数据从哪里来
- 需要找到哪个测试/代码触发了问题

## 追踪过程

### 1. 观察症状
```
Error: git init failed in /Users/jesse/project/packages/core
```

### 2. 找到直接原因
**什么代码直接导致了这个问题？**
```typescript
await execFileAsync('git', ['init'], { cwd: projectDir });
```

### 3. 问：谁调用了这个？
```typescript
WorktreeManager.createSessionWorktree(projectDir, sessionId)
  → called by Session.initializeWorkspace()
  → called by Session.create()
  → called by test at Project.create()
```

### 4. 继续向上追踪
**传递了什么值？**
- `projectDir = ''`（空字符串！）
- 空字符串作为 `cwd` 会解析为 `process.cwd()`
- 那就是源代码目录！

### 5. 找到原始触发点
**空字符串从哪里来？**
```typescript
const context = setupCoreTest(); // Returns { tempDir: '' }
Project.create('name', context.tempDir); // Accessed before beforeEach!
```

## 添加堆栈跟踪

当你无法手动追踪时，添加检测代码：

```typescript
// 在有问题的操作之前
async function gitInit(directory: string) {
  const stack = new Error().stack;
  console.error('DEBUG git init:', {
    directory,
    cwd: process.cwd(),
    nodeEnv: process.env.NODE_ENV,
    stack,
  });

  await execFileAsync('git', ['init'], { cwd: directory });
}
```

**关键：**在测试中使用 `console.error()`（不是 logger - 可能不会显示）

**运行并捕获：**
```bash
npm test 2>&1 | grep 'DEBUG git init'
```

**分析堆栈跟踪：**
- 寻找测试文件名
- 找到触发调用的行号
- 识别模式（同一个测试？同一个参数？）

## 找出哪个测试导致污染

如果某些东西在测试期间出现但不知道是哪个测试：

使用本目录中的二分查找脚本 `find-polluter.sh`：

```bash
./find-polluter.sh '.git' 'src/**/*.test.ts'
```

逐个运行测试，在第一个污染者处停止。参见脚本了解用法。

## 真实示例：空的 projectDir

**症状：**`.git` 创建在 `packages/core/`（源代码目录）

**追踪链：**
1. `git init` 在 `process.cwd()` 中运行 ← 空的 cwd 参数
2. WorktreeManager 用空的 projectDir 调用
3. Session.create() 传递了空字符串
4. 测试在 beforeEach 之前访问了 `context.tempDir`
5. setupCoreTest() 初始返回 `{ tempDir: '' }`

**根本原因：**顶层变量初始化访问了空值

**修复：**将 tempDir 改为 getter，如果在 beforeEach 之前访问则抛出异常

**还添加了纵深防御：**
- 第 1 层：Project.create() 验证目录
- 第 2 层：WorkspaceManager 验证非空
- 第 3 层：NODE_ENV 守卫拒绝在 tmpdir 之外的 git init
- 第 4 层：git init 之前的堆栈跟踪日志

## 关键原则

```dot
digraph principle {
    "找到直接原因" [shape=ellipse];
    "可以向上追踪一级吗？" [shape=diamond];
    "向后追踪" [shape=box];
    "这是源头吗？" [shape=diamond];
    "在源头修复" [shape=box];
    "在每层添加验证" [shape=box];
    "Bug 不可能发生" [shape=doublecircle];
    "永远不要只修复症状" [shape=octagon, style=filled, fillcolor=red, fontcolor=white];

    "找到直接原因" -> "可以向上追踪一级吗？";
    "可以向上追踪一级吗？" -> "向后追踪" [label="是"];
    "可以向上追踪一级吗？" -> "永远不要只修复症状" [label="否"];
    "向后追踪" -> "这是源头吗？";
    "这是源头吗？" -> "向后追踪" [label="否 - 继续"];
    "这是源头吗？" -> "在源头修复" [label="是"];
    "在源头修复" -> "在每层添加验证";
    "在每层添加验证" -> "Bug 不可能发生";
}
```

**永远不要只修复错误出现的地方。**追溯找到原始触发点。

## 堆栈跟踪技巧

**在测试中：**使用 `console.error()` 而不是 logger - logger 可能被抑制
**在操作之前：**在危险操作之前记录，而不是在它失败之后
**包含上下文：**目录、cwd、环境变量、时间戳
**捕获堆栈：**`new Error().stack` 显示完整调用链

## 真实影响

来自调试会话（2025-10-03）：
- 通过 5 层追踪找到根本原因
- 在源头修复（getter 验证）
- 添加了 4 层防御
- 1847 个测试通过，零污染