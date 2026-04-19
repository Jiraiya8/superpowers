# Claude Code 技能测试

使用 Claude Code CLI 对 superpowers 技能进行自动化测试。

## 概述

本测试套件验证技能是否正确加载以及 Claude 是否按预期遵循它们。测试在无头模式（`claude -p`）下调用 Claude Code 并验证行为。

## 要求

- 已安装 Claude Code CLI 并在 PATH 中（`claude --version` 应正常工作）
- 已安装本地 superpowers 插件（参见主 README 了解安装方法）

## 运行测试

### 运行所有快速测试（推荐）：
```bash
./run-skill-tests.sh
```

### 运行集成测试（较慢，10-30 分钟）：
```bash
./run-skill-tests.sh --integration
```

### 运行特定测试：
```bash
./run-skill-tests.sh --test test-subagent-driven-development.sh
```

### 运行并显示详细输出：
```bash
./run-skill-tests.sh --verbose
```

### 设置自定义超时：
```bash
./run-skill-tests.sh --timeout 1800  # 集成测试 30 分钟
```

## 测试结构

### test-helpers.sh
技能测试的通用函数：
- `run_claude "prompt" [timeout]` - 使用提示运行 Claude
- `assert_contains output pattern name` - 验证模式存在
- `assert_not_contains output pattern name` - 验证模式不存在
- `assert_count output pattern count name` - 验证精确计数
- `assert_order output pattern_a pattern_b name` - 验证顺序
- `create_test_project` - 创建临时测试目录
- `create_test_plan project_dir` - 创建示例计划文件

### 测试文件

每个测试文件：
1. 引入 `test-helpers.sh`
2. 使用特定提示运行 Claude Code
3. 使用断言验证预期行为
4. 成功返回 0，失败返回非零

## 示例测试

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/test-helpers.sh"

echo "=== Test: My Skill ==="

# 询问 Claude 关于技能
output=$(run_claude "What does the my-skill skill do?" 30)

# 验证响应
assert_contains "$output" "expected behavior" "Skill describes behavior"

echo "=== All tests passed ==="
```

## 当前测试

### 快速测试（默认运行）

#### test-subagent-driven-development.sh
测试技能内容和要求（约 2 分钟）：
- 技能加载和可访问性
- 工作流顺序（规范合规性在代码质量之前）
- 自我审查要求已记录
- 计划读取效率已记录
- 规范合规性审查者的怀疑态度已记录
- 审查循环已记录
- 任务上下文提供已记录

### 集成测试（使用 --integration 标志）

#### test-subagent-driven-development-integration.sh
完整工作流执行测试（约 10-30 分钟）：
- 创建真实测试项目，使用 Node.js 设置
- 创建包含 2 个任务的实现计划
- 使用 subagent-driven-development 执行计划
- 验证实际行为：
  - 计划在开始时读取一次（非每个任务）
  - 完整的任务文本在子代理提示中提供
  - 子代理在报告前执行自我审查
  - 规范合规性审查在代码质量之前进行
  - 规范审查者独立阅读代码
  - 生成可工作的实现
  - 测试通过
  - 创建正确的 git 提交

**测试内容：**
- 工作流端到端实际工作
- 我们的改进实际被应用
- 子代理正确遵循技能
- 最终代码功能正常且已测试

## 添加新测试

1. 创建新测试文件：`test-<skill-name>.sh`
2. 引入 test-helpers.sh
3. 使用 `run_claude` 和断言编写测试
4. 将测试添加到 `run-skill-tests.sh` 的测试列表
5. 设置可执行权限：`chmod +x test-<skill-name>.sh`

## 超时注意事项

- 默认超时：每个测试 5 分钟
- Claude Code 可能需要时间响应
- 如需要可使用 `--timeout` 调整
- 测试应保持聚焦以避免长时间运行

## 调试失败的测试

使用 `--verbose` 可查看完整的 Claude 输出：
```bash
./run-skill-tests.sh --verbose --test test-subagent-driven-development.sh
```

不使用 verbose 时，仅失败时显示输出。

## CI/CD 集成

在 CI 中运行：
```bash
# 为 CI 环境设置显式超时
./run-skill-tests.sh --timeout 900

# 退出码 0 = 成功，非零 = 失败
```

## 注意事项

- 测试验证技能*指令*，而非完整执行
- 完整工作流测试会非常慢
- 专注于验证关键技能要求
- 测试应具有确定性
- 避免测试实现细节