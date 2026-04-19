# 零依赖头脑风暴服务器

用仅使用 Node.js 内置模块的单个零依赖 `server.js` 替换头脑风暴伴生服务器的 vendored node_modules（express、ws、chokidar — 714 个追踪文件）。

## 动机

将 node_modules vendored 到 git 仓库会带来供应链风险：冻结的依赖不获得安全补丁，714 个第三方代码文件未经审计提交，对 vendored 代码的修改看起来像普通提交。虽然实际风险低（仅 localhost 开发服务器），但消除它是直接的。

## 架构

使用 `http`、`crypto`、`fs` 和 `path` 的单个 `server.js` 文件（约 250-300 行）。文件有两个角色：

- **直接运行时**（`node server.js`）：启动 HTTP/WebSocket 服务器
- **被 require 时**（`require('./server.js')`）：导出 WebSocket 协议函数供单元测试

### WebSocket 协议

仅实现 RFC 6455 的文本帧：

**握手：** 从客户端的 `Sec-WebSocket-Key` 使用 SHA-1 + RFC 6455 magic GUID 计算 `Sec-WebSocket-Accept`。返回 101 Switching Protocols。

**帧解码（客户端到服务器）：** 处理三种掩码长度编码：
- Small：payload < 126 字节
- Medium：126-65535 字节（16 位扩展）
- Large：> 65535 字节（64 位扩展）

使用 4 字节掩码密钥 XOR 解掩 payload。返回 `{ opcode，payload，bytesConsumed }` 或 `null` 表示不完整缓冲。拒绝未掩码帧。

**帧编码（服务器到客户端）：** 使用相同三种长度编码的未掩码帧。

**处理的 opcode：** TEXT（0x01）、CLOSE（0x08）、PING（0x09）、PONG（0x0A）。未知 opcode 收到带状态 1003（Unsupported Data）的关闭帧。

**有意跳过：** 二进制帧、分片消息、扩展（permessage-deflate）、子协议。这些对于 localhost 客户端间的小 JSON 文本消息不需要。扩展和子协议在握手时协商 — 通过不声明它们，它们永远不会激活。

**缓冲积累：** 每个连接维护一个缓冲。在 `data` 时，追加并循环 `decodeFrame` 直到返回 null 或缓冲为空。

### HTTP 服务器

三条路由：

1. **`GET /`** — 按 mtime 从 screen 目录服务最新的 `.html`。检测完整文档 vs 片段，在 frame 模板中包装片段，注入 helper.js。返回 `text/html`。当没有 `.html` 文件时，服务硬编码等待页（"Waiting for Claude to push a screen..."）并注入 helper.js。
2. **`GET /files/*`** — 从 screen 目录服务静态文件，使用硬编码扩展名映射的 MIME 类型查找（html、css、js、png、jpg、gif、svg、json）。未找到返回 404。
3. **其他** — 404。

WebSocket upgrade 通过 HTTP server 的 `'upgrade'` 事件处理，与 request handler 分开。

### 配置

环境变量（全部可选）：

- `BRAINSTORM_PORT` — 绑定端口（默认：随机高端口 49152-65535）
- `BRAINSTORM_HOST` — 绑定接口（默认：`127.0.0.1`）
- `BRAINSTORM_URL_HOST` — startup JSON 中 URL 的主机名（默认：host 为 `127.0.0.1` 时 `localhost`，否则与 host 相同）
- `BRAINSTORM_DIR` — screen 目录路径（默认：`/tmp/brainstorm`）

### 启动序列

1. 创建 `SCREEN_DIR` 如果不存在（`mkdirSync` recursive）
2. 从 `__dirname` 加载 frame 模板和 helper.js
3. 在配置的 host/port 启动 HTTP server
4. 在 `SCREEN_DIR` 启动 `fs.watch`
5. 成功监听时，向 stdout 记录 `server-started` JSON：`{ type，port，host，url_host，url，screen_dir }`
6. 将相同 JSON 写入 `SCREEN_DIR/.server-info`，以便 stdout 隐藏时代理能找到连接详情（后台执行）

### 应用层 WebSocket 消息

当 TEXT 帧从客户端到达：

1. 解析为 JSON。解析失败则记录到 stderr 继续。
2. 向 stdout 记录为 `{ source: 'user-event'，...event }`。
3. 如果 event 包含 `choice` 属性，将 JSON 追加到 `SCREEN_DIR/.events`（每个 event 一行）。

### 文件监视

`fs.watch(SCREEN_DIR)` 替代 chokidar。在 HTML 文件事件：

- 新文件（`rename` event 对存在的文件）：如有 `.events` 文件则删除（`unlinkSync`），向 stdout 记录 `screen-added` 为 JSON
- 文件变更（`change` event）：向 stdout 记录 `screen-updated` 为 JSON（不清除 `.events`）
- 两种事件：向所有连接的 WebSocket 客户端发送 `{ type: 'reload' }`

每个 filename 约 100ms 超时防抖以防止重复事件（macOS 和 Linux 上常见）。

### 错误处理

- WebSocket 客户端的畸形 JSON：记录到 stderr 继续
- 未处理的 opcode：以状态 1003 关闭
- 客户端断开：从广播集移除
- `fs.watch` 错误：记录到 stderr 继续
- 无优雅关闭逻辑 — shell 脚本通过 SIGTERM 处理进程生命周期

## 变化

| 之前 | 之后 |
|---|---|
| `index.js` + `package.json` + `package-lock.json` + 714 个 `node_modules` 文件 | `server.js`（单文件） |
| express、ws、chokidar 依赖 | 无 |
| 无静态文件服务 | `/files/*` 从 screen 目录服务 |

## 保持不变

- `helper.js` — 无变化
- `frame-template.html` — 无变化
- `start-server.sh` — 一行更新：`index.js` 改为 `server.js`
- `stop-server.sh` — 无变化
- `visual-companion.md` — 无变化
- 所有现有服务器行为和外部契约

## 平台兼容性

- `server.js` 仅使用跨平台 Node 内置模块
- `fs.watch` 在 macOS、Linux 和 Windows 的单层平面目录上可靠
- Shell 脚本需要 bash（Windows 上用 Git Bash，这是 Claude Code 必需的）

## 测试

**单元测试**（`ws-protocol.test.js`）：通过 require `server.js` 导出直接测试 WebSocket 帧编码/解码、握手计算和协议边界情况。

**集成测试**（`server.test.js`）：测试完整服务器行为 — HTTP 服务、WebSocket 通信、文件监视、头脑风暴工作流。使用 `ws` npm 包作为仅测试的客户端依赖（不发送给最终用户）。