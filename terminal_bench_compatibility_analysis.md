# Terminal-Bench 2.0 用例集构建方案分析

> 基于 Terminal-Bench 2.0 标准构建评测用例集的可行性分析

---

## 目录

- [1. Terminal-Bench 2.0 概述](#1-terminal-bench-20-概述)
- [2. 方案可行性分析](#2-方案可行性分析)
- [3. CLI Agent 适配方案](#3-cli-agent-适配方案)
- [4. Windows GUI 适配方案](#4-windows-gui-适配方案)
- [5. 用例集构建建议](#5-用例集构建建议)
- [6. 实施路线图](#6-实施路线图)

---

## 1. Terminal-Bench 2.0 概述

### 1.1 基本信息

| 项目 | 说明 |
|------|------|
| **仓库地址** | https://github.com/harbor-framework/terminal-bench-2 |
| **任务数量** | 89 个 |
| **评测框架** | Harbor |
| **运行环境** | Docker 容器 |
| **已评测 Agent** | Claude Code, Codex CLI, Gemini CLI |

### 1.2 任务结构

Terminal-Bench 任务标准格式：

```
task/
├── task.yaml          # 任务配置
├── setup.sh           # 环境初始化脚本
├── verify.sh          # 验证脚本
├── solution/          # 参考解决方案
│   └── solution.sh
└── assets/            # 相关资源文件
```

**task.yaml 示例**：

```yaml
name: "debug-async-code"
description: "Debug an async Python application with race conditions"
difficulty: "hard"
tags:
  - debugging
  - python
  - async
time_limit: 1800
setup:
  - setup.sh
verify:
  - verify.sh
```

### 1.3 Harbor 评测框架

Harbor 是 Terminal-Bench 官方的评测框架：

```bash
# 安装
pip install harbor

# 运行评测
harbor run \
  --dataset terminal-bench@2.0 \
  --agent claude-code \
  --model anthropic/claude-opus-4-1 \
  --n-concurrent 4
```

---

## 2. 方案可行性分析

### 2.1 可行性总结

| 评测场景 | 可行性 | 难度 | 说明 |
|----------|--------|------|------|
| Linux CLI Agent (Claude Code) | ✅ 完全可行 | 低 | 原生支持 |
| Linux CLI Agent (Opencode) | ✅ 完全可行 | 低 | 需实现 Agent 适配器 |
| Windows GUI (VS Code) | ⚠️ 需适配 | 中 | 需要桥接层 |
| Windows GUI (CodeArts Doer) | ⚠️ 需适配 | 中 | 需要桥接层 |

### 2.2 优势分析

**1. 标准化程度高**

- 任务格式统一（YAML + Shell）
- 验证机制标准化（verify.sh）
- 评分规则明确

**2. 基础设施成熟**

- Harbor 框架提供完整的评测流水线
- Docker 容器确保环境一致性
- 支持并行评测

**3. 社区认可度高**

- 被 Anthropic、OpenAI、Google 等前沿实验室使用
- 有公开的 Leaderboard
- 持续维护更新

**4. 扩展性好**

- 支持自定义 Agent 适配器
- 支持自定义数据集
- 支持云端执行（Modal 等）

### 2.3 风险与挑战

**1. 数据污染风险**

Terminal-Bench 2.0 已公开发布，存在被用于模型训练的风险：

| 风险等级 | 说明 |
|----------|------|
| **高风险** | 任务描述、解决方案可能已在训练数据中 |
| **中风险** | 任务模式可能被学习 |
| **低风险** | 具体验证逻辑难以记忆 |

**缓解措施**：
- 使用 Terminal-Bench Pro 的私有集（需申请）
- 自行构建新任务，复用框架格式
- 对任务描述进行改写

**2. CLI 与 GUI 的差异**

| 维度 | CLI Agent | GUI Agent |
|------|-----------|-----------|
| 交互方式 | 命令行 | 图形界面 |
| 输入方式 | 文本命令 | 点击、菜单、快捷键 |
| 状态感知 | 文本输出 | 视觉反馈 |
| 自动化程度 | 高（脚本化） | 低（需 UI 自动化） |

---

## 3. CLI Agent 适配方案

### 3.1 Claude Code（已支持）

Terminal-Bench 2.0 原生支持 Claude Code：

```bash
export ANTHROPIC_API_KEY=<YOUR-KEY>

harbor run \
  --dataset terminal-bench@2.0 \
  --agent claude-code \
  --model anthropic/claude-opus-4-1 \
  --n-concurrent 4
```

### 3.2 Opencode（需适配）

需要实现 `BaseInstalledAgent` 或 `BaseAgent` 子类：

```python
# opencode_agent.py
from harbor.agents import BaseInstalledAgent

class OpencodeAgent(BaseInstalledAgent):
    name = "opencode"

    def __init__(self, model: str, **kwargs):
        super().__init__(model=model, **kwargs)

    def run(self, task_dir: str, **kwargs) -> dict:
        """
        在指定任务目录中运行 Opencode
        """
        import subprocess
        import os

        # 切换到任务目录
        os.chdir(task_dir)

        # 启动 Opencode
        result = subprocess.run(
            ["opencode", "--task", "task.yaml"],
            capture_output=True,
            text=True,
            timeout=self.time_limit
        )

        return {
            "success": result.returncode == 0,
            "output": result.stdout,
            "error": result.stderr
        }
```

**注册 Agent**：

```python
# 在 Harbor 中注册
from harbor import register_agent
register_agent("opencode", OpencodeAgent)
```

**运行评测**：

```bash
harbor run \
  --dataset my-dataset@1.0 \
  --agent opencode \
  --model custom/model \
  --n-concurrent 4
```

### 3.3 CLI Agent 通用适配模板

```python
from harbor.agents import BaseAgent
from abc import abstractmethod
import subprocess
import os

class CLIAgentAdapter(BaseAgent):
    """CLI Agent 通用适配器基类"""

    def __init__(self, model: str, time_limit: int = 1800, **kwargs):
        super().__init__(model=model, **kwargs)
        self.time_limit = time_limit

    @property
    @abstractmethod
    def cli_command(self) -> str:
        """返回 CLI 工具的命令名"""
        pass

    @property
    @abstractmethod
    def name(self) -> str:
        """返回 Agent 名称"""
        pass

    def prepare_prompt(self, task_yaml: dict) -> str:
        """将任务描述转换为 CLI 输入"""
        return f"""
Task: {task_yaml['name']}
Description: {task_yaml['description']}
Difficulty: {task_yaml.get('difficulty', 'medium')}

Please complete this task. The working directory is ready with all necessary files.
"""

    def run(self, task_dir: str, **kwargs) -> dict:
        os.chdir(task_dir)

        # 读取任务配置
        import yaml
        with open("task.yaml") as f:
            task_config = yaml.safe_load(f)

        prompt = self.prepare_prompt(task_config)

        # 执行 CLI 工具
        result = subprocess.run(
            [self.cli_command, "--prompt", prompt],
            capture_output=True,
            text=True,
            timeout=self.time_limit,
            input=prompt
        )

        return {
            "agent": self.name,
            "success": result.returncode == 0,
            "output": result.stdout,
            "error": result.stderr,
            "task": task_config['name']
        }
```

---

## 4. Windows GUI 适配方案

### 4.1 技术挑战

Windows GUI 工具（VS Code、CodeArts Doer）的自动化评测面临以下挑战：

| 挑战 | 说明 |
|------|------|
| **UI 自动化** | 需要模拟鼠标点击、键盘输入 |
| **状态检测** | 需要解析视觉反馈 |
| **环境差异** | Windows vs Linux 容器环境 |
| **进程管理** | GUI 进程的生命周期管理 |

### 4.2 适配方案架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Harbor 评测框架                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  GUI Agent Bridge                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Task Parser │  │ UI Automator│  │ Result Collector│       │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
   │  VS Code    │     │ CodeArts    │     │   Cursor    │
   │  Adapter    │     │   Doer      │     │  Adapter    │
   └─────────────┘     └─────────────┘     └─────────────┘
```

### 4.3 VS Code 适配方案

**方案 A：使用 VS Code 扩展 API**

```python
# vscode_agent.py
import subprocess
import json
import time

class VSCodeAgent:
    """VS Code 自动化评测适配器"""

    def __init__(self, extension_id: str = None):
        self.extension_id = extension_id
        self.workspace = None

    def setup(self, task_dir: str):
        """准备 VS Code 工作区"""
        self.workspace = task_dir
        # 启动 VS Code 并打开工作区
        subprocess.Popen([
            "code",
            "--folder-uri", f"file://{task_dir}",
            "--disable-extensions" if not self.extension_id else f"--install-extension {self.extension_id}"
        ])
        time.sleep(5)  # 等待 VS Code 启动

    def execute_command(self, command: str):
        """通过 VS Code 命令面板执行命令"""
        # 使用 VS Code CLI 或扩展 API
        subprocess.run([
            "code",
            "--command", command
        ])

    def run_copilot_chat(self, prompt: str):
        """触发 Copilot Chat"""
        # 模拟快捷键打开 Copilot Chat
        # 需要配合 UI 自动化工具
        pass
```

**方案 B：使用 Playwright 进行 UI 自动化**

```python
# vscode_playwright_agent.py
from playwright.sync_api import sync_playwright
import time

class VSCodePlaywrightAgent:
    """使用 Playwright 自动化 VS Code Web 版"""

    def __init__(self, vscode_web_url: str):
        self.url = vscode_web_url

    def run_task(self, task_dir: str, prompt: str):
        with sync_playwright() as p:
            browser = p.chromium.launch(headless=False)
            context = browser.new_context()
            page = context.new_page()

            # 打开 VS Code Web
            page.goto(self.url)

            # 打开工作区
            # ... (具体的 UI 操作)

            # 输入 prompt 到 Copilot Chat
            page.click('[aria-label="GitHub Copilot Chat"]')
            page.fill('[placeholder="Ask Copilot"]', prompt)
            page.press('[placeholder="Ask Copilot"]', 'Enter')

            # 等待响应
            time.sleep(30)

            # 收集结果
            # ...

            browser.close()
```

### 4.4 CodeArts Doer 适配方案

CodeArts Doer 是华为的 AI 编程助手，适配方案类似：

```python
# codearts_doer_agent.py
import subprocess
import time
import os

class CodeArtsDoerAgent:
    """CodeArts Doer 自动化评测适配器"""

    def __init__(self, doer_path: str):
        self.doer_path = doer_path

    def setup(self, task_dir: str):
        """启动 CodeArts Doer 并打开项目"""
        # 启动 CodeArts Doer
        subprocess.Popen([
            self.doer_path,
            "--open", task_dir
        ])
        time.sleep(10)  # 等待应用启动

    def send_prompt(self, prompt: str):
        """发送 prompt 到 Doer"""
        # 方案1: 如果 Doer 提供 CLI 接口
        # 方案2: 使用 UI 自动化（pyautogui）
        # 方案3: 使用 Doer 的 API（如果有）
        pass
```

### 4.5 Windows 环境运行 Terminal-Bench 任务

**方案：使用 WSL2 + Docker**

```yaml
# docker-compose.windows.yml
version: '3'
services:
  task-runner:
    build: .
    volumes:
      - ./tasks:/tasks
      - ./results:/results
    environment:
      - TASK_ID=${TASK_ID}
      - AGENT_TYPE=${AGENT_TYPE}
```

**Windows 运行脚本**：

```powershell
# run_benchmark.ps1
param(
    [string]$TaskId,
    [string]$AgentType = "vscode"
)

# 启动 WSL2 中的 Docker 容器
wsl docker run -d `
    -v /mnt/c/tasks:/tasks `
    -v /mnt/c/results:/results `
    -e TASK_ID=$TaskId `
    -e AGENT_TYPE=$AgentType `
    benchmark-runner

# 等待容器准备就绪
Start-Sleep -Seconds 10

# 触发 GUI Agent（在 Windows 端）
# 连接到容器中的代码目录
code --folder-uri "vscode-remote://wsl+docker/tasks/$TaskId"

# 等待任务完成
# ...
```

---

## 5. 用例集构建建议

### 5.1 任务格式规范（兼容 Terminal-Bench 2.0）

```yaml
# task.yaml 完整模板
id: "unique-task-id"
name: "任务名称"
description: |
  详细描述任务要求，包括：
  - 背景说明
  - 具体要求
  - 预期输出

difficulty: "easy|medium|hard"
tags:
  - tag1
  - tag2
  - tag3

# 时间限制（秒）
time_limit: 1800

# 环境配置
environment:
  base_image: "python:3.10"
  setup_commands:
    - "pip install -r requirements.txt"
  env_vars:
    - name: "DEBUG"
      value: "true"

# 输入文件（Agent 可见）
inputs:
  - path: "src/main.py"
    description: "主程序入口"
  - path: "tests/"
    description: "测试目录"

# 验证配置
verification:
  type: "script"  # script | pytest | custom
  command: "python -m pytest tests/ -v"
  success_criteria:
    - "所有测试通过"
    - "无回归错误"

# 评分配置
scoring:
  method: "binary"  # binary | partial | custom
  weights:
    correctness: 0.6
    code_quality: 0.2
    efficiency: 0.2

# 元数据
metadata:
  author: "作者"
  created_at: "2026-04-01"
  version: "1.0"
  language: "python"
  domain: "debugging"
```

### 5.2 目录结构

```
my-benchmark/
├── dataset.yaml           # 数据集配置
├── tasks/
│   ├── task-001/
│   │   ├── task.yaml
│   │   ├── setup.sh
│   │   ├── verify.sh
│   │   ├── solution/
│   │   │   └── solution.sh
│   │   └── assets/
│   │       └── ...
│   ├── task-002/
│   │   └── ...
│   └── ...
├── agents/                # Agent 适配器
│   ├── claude_code.py
│   ├── opencode.py
│   ├── vscode.py
│   └── codearts_doer.py
├── adapters/              # GUI 桥接层
│   ├── gui_bridge.py
│   └── playwright_utils.py
└── scripts/
    ├── run_cli.sh
    └── run_gui.ps1
```

### 5.3 验证脚本标准

```bash
#!/bin/bash
# verify.sh - 任务验证脚本

set -e

# 1. 检查必要文件是否存在
check_files() {
    local files=("src/main.py" "src/utils.py")
    for f in "${files[@]}"; do
        if [ ! -f "$f" ]; then
            echo "ERROR: Missing file $f"
            exit 1
        fi
    done
}

# 2. 运行单元测试
run_tests() {
    python -m pytest tests/ -v --tb=short
    if [ $? -ne 0 ]; then
        echo "ERROR: Tests failed"
        exit 1
    fi
}

# 3. 检查代码风格
check_style() {
    flake8 src/ --max-line-length=100
    if [ $? -ne 0 ]; then
        echo "WARNING: Style issues found"
        # 不作为失败条件
    fi
}

# 4. 功能验证
verify_functionality() {
    python -c "
import sys
sys.path.insert(0, 'src')
from main import main
# 验证核心功能
result = main('--test')
assert result == 0, 'Functionality check failed'
"
}

# 执行验证
echo "Starting verification..."
check_files
run_tests
check_style
verify_functionality
echo "Verification completed successfully!"
exit 0
```

### 5.4 多平台兼容性要求

为确保任务在 CLI 和 GUI 环境都能运行：

| 要求 | CLI | GUI |
|------|-----|-----|
| 任务描述 | 纯文本 | 可包含 Markdown |
| 文件路径 | POSIX 路径 | 统一使用相对路径 |
| 验证方式 | Shell 脚本 | Shell + UI 检测 |
| 超时处理 | 进程 kill | 窗口关闭 |

---

## 6. 实施路线图

### 6.1 阶段规划

```
Phase 1: 基础建设（2周）
├── 搭建 Terminal-Bench 2.0 环境
├── 熟悉 Harbor 框架
└── 运行现有任务验证环境

Phase 2: CLI Agent 适配（2周）
├── 实现 Opencode 适配器
├── 测试 Claude Code 集成
└── 编写自动化评测脚本

Phase 3: GUI Agent 适配（3周）
├── 设计 GUI Bridge 架构
├── 实现 VS Code 适配器
├── 实现 CodeArts Doer 适配器
└── 测试 Windows 环境

Phase 4: 用例集构建（4周）
├── 设计任务模板
├── 编写新任务（建议50+）
├── 编写验证脚本
└── 人工验证任务可解性

Phase 5: 评测与优化（2周）
├── 运行完整评测
├── 分析结果
└── 优化任务和适配器
```

### 6.2 资源需求

| 资源 | 说明 |
|------|------|
| **开发环境** | Linux 服务器 + Windows 测试机 |
| **API 调用** | 各 LLM 服务的 API Key |
| **计算资源** | Docker 运行环境，建议 16GB+ 内存 |
| **人力** | 1-2 名开发人员 |

### 6.3 快速启动命令

```bash
# 1. 安装 Harbor
pip install harbor

# 2. 克隆 Terminal-Bench 2.0
git clone https://github.com/harbor-framework/terminal-bench-2

# 3. 运行测试
export ANTHROPIC_API_KEY=<YOUR-KEY>
harbor run \
  --dataset terminal-bench@2.0 \
  --agent claude-code \
  --model anthropic/claude-sonnet-4-5 \
  --n-concurrent 4

# 4. 创建自定义数据集
harbor datasets create my-benchmark --template terminal-bench
```

---

## 总结

### 可行性结论

| 目标 | 结论 | 建议 |
|------|------|------|
| Linux CLI Agent 评测 | ✅ 完全可行 | 直接使用 Terminal-Bench 2.0 |
| Windows GUI 评测 | ⚠️ 需适配 | 建议实现 Bridge 层 |
| 自建未污染用例集 | ✅ 可行 | 复用 Terminal-Bench 格式 |

### 推荐方案

1. **短期**：直接使用 Terminal-Bench 2.0 + Harbor 框架评测 CLI Agent
2. **中期**：开发 GUI Bridge 层，支持 VS Code 和 CodeArts Doer
3. **长期**：基于 Terminal-Bench 格式构建私有用例集，避免数据污染

---

## 参考资料

- [Terminal-Bench 2.0 GitHub](https://github.com/harbor-framework/terminal-bench-2)
- [Terminal-Bench Pro (阿里)](https://github.com/alibaba/terminal-bench-pro)
- [Harbor 框架文档](https://github.com/laude-institute/harbor)
- [Terminal-Bench 论文](https://arxiv.org/html/2601.11868v1)

---

*文档版本：1.0*
*最后更新：2026-04-01*
