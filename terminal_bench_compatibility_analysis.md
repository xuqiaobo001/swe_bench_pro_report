# Terminal-Bench 2.0 用例集构建方案分析

> 基于 Terminal-Bench 2.0 标准构建评测用例集的可行性分析

---

## 目录

- [1. Terminal-Bench 2.0 概述](#1-terminal-bench-20-概述)
- [2. 方案可行性分析](#2-方案可行性分析)
- [3. CLI Agent 适配方案](#3-cli-agent-适配方案)
- [4. Windows GUI 统一适配方案](#4-windows-gui-统一适配方案)
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

### 1.2 任务结构（实际格式）

Terminal-Bench 任务标准格式：

```
task/
├── task.toml           # 任务配置 (TOML 格式)
├── instruction.md      # 任务描述 (Markdown 格式)
├── environment/        # 环境配置目录
│   ├── Dockerfile      # Docker 构建文件
│   └── ...              # 其他依赖文件、源码等
├── solution/           # 参考解决方案
│   └── solve.sh         # 解决方案脚本
└── tests/              # 测试验证
    ├── test.sh          # 测试入口脚本
    └── test_outputs.py  # pytest 测试文件
```

**task.toml 示例**：

```toml
name = "adaptive-rejection-sampler"
description = "Implement an adaptive rejection sampler for Bayesian inference"
difficulty = "medium"
tags = ["statistics", "sampling", "python"]
time_limit = 1800
expert_time_estimate_min = 480.0
junior_time_estimate_min = 240.0
```

**instruction.md 示例**：

```markdown
## Task Description

Your task is to implement an adaptive rejection sampler...

## Requirements

1. Implement the `AdaptiveRejectionSampler` class
2. Support log-concave distributions
3. Pass all test cases
```

**environment/Dockerfile 示例**：

```dockerfile
FROM python:3.11-slim

RUN pip install numpy scipy pytest

WORKDIR /app
COPY . /app
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
| Linux CLI Agent (Opencode) | ✅ 完全可行 | 低 | Harbor 已原生支持 |
| Windows GUI (JetBrains IDE) | ⚠️ 需适配 | 中 | 通过统一 Bridge 层支持 |
| Windows GUI (VS Code) | ⚠️ 需适配 | 中 | 通过统一 Bridge 层支持 |
| Windows GUI (CodeArts Doer) | ⚠️ 需适配 | 中 | 通过统一 Bridge 层支持 |

### 2.2 优势分析

**1. 标准化程度高**

- 任务格式统一（TOML + Markdown + Shell）
- 验证机制标准化（tests/test.sh）
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

### 2.3 鸎险与挑战

**1. 数据污染风险**

Terminal-Bench 2.0 已公开发布，存在被用于模型训练的风险:

| 风险等级 | 说明 |
|----------|------|
| **高风险** | 任务描述、解决方案可能已在训练数据中 |
| **中风险** | 任务模式可能被学习 |
| **低风险** | 具体验证逻辑难以记忆 |

**缓解措施**:
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

Terminal-Bench 2.0 原生支持 Claude Code:

```bash
export ANTHROPIC_API_KEY=<YOUR-KEY>

harbor run \
  --dataset terminal-bench@2.0 \
  --agent claude-code \
  --model anthropic/claude-opus-4-1 \
  --n-concurrent 4
```

### 3.2 Opencode（已支持）

Harbor 框架已原生支持 Opencode，可直接使用:

```bash
# 运行 Opencode 评测
harbor run \
  --dataset terminal-bench@2.0 \
  --agent opencode \
  --model <model-endpoint> \
  --n-concurrent 4
```

Harbor 支持的 CLI Agent 列表（持续更新）：
- Claude Code
- Codex CLI
- Gemini CLI
- OpenHands
- Opencode
- 其他 Agent 可通过 `BaseInstalledAgent` 或 `BaseAgent` 扩展

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

    def prepare_prompt(self, instruction_md: str) -> str:
        """将任务描述转换为 CLI 输入"""
        return instruction_md  # 默认直接使用原始描述

    def run(self, task_dir: str, **kwargs) -> dict:
        os.chdir(task_dir)

        # 读取 instruction.md
        with open("instruction.md") as f:
            instruction = f.read()

        prompt = self.prepare_prompt(instruction)

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
            "error": result.stderr
        }
```

---

## 4. Windows GUI 统一适配方案

### 4.1 统一架构设计

针对 JetBrains IDE、Visual Studio Code、CodeArts Doer 三种 Windows GUI 工具，建议采用**统一的 GUI Agent Bridge 层**:

```
┌─────────────────────────────────────────────────────────────┐
│                    Harbor 评测框架                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  GUI Agent Bridge (统一层)                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Task Parser │  │ UI Automator│  │ Result Coll.│         │
│  │ (TOML+MD)   │  │ (LSP/Playwr.)│  │ (Log/Screen)│         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
   │  JetBrains  │     │  VS Code    │     │ CodeArts    │
   │  Adapter    │     │  Adapter    │     │   Doer      │
   │  (LSP)      │     │  (LSP/Ext)  │     │  (LSP/API)  │
   └─────────────┘     └─────────────┘     └─────────────┘
          │                   │                   │
          ▼                   ▼                   ▼
   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
   │ IntelliJ    │     │ VS Code     │     │ CodeArts    │
   │  Platform   │     │  Desktop    │     │   Desktop   │
   └─────────────┘     └─────────────┘     └─────────────┘
```

### 4.2 统一适配层核心接口

```python
# gui_bridge.py
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Optional, Dict, Any
import subprocess
import time
import os

@dataclass
class TaskContext:
    """任务上下文"""
    task_dir: str
    task_name: str
    instruction: str
    time_limit: int
    environment_type: str  # docker, local

@dataclass
class ExecutionResult:
    """执行结果"""
    success: bool
    output: str
    error: str
    duration: float
    screenshots: list  # 截图路径列表

class GUIAgentBridge(ABC):
    """GUI Agent 统一桥接层基类"""

    def __init__(self, config: Dict[str, Any] = None):
        self.config = config or {}

    @property
    @abstractmethod
    def name(self) -> str:
        """Agent 名称"""
        pass

    @property
    @abstractmethod
    def supported_platforms(self) -> list:
        """支持的平台列表"""
        pass

    @abstractmethod
    def launch(self, context: TaskContext) -> bool:
        """启动 GUI 工具"""
        pass

    @abstractmethod
    def open_project(self, project_path: str) -> bool:
        """打开项目"""
        pass

    @abstractmethod
    def send_prompt(self, prompt: str) -> bool:
        """发送提示词到 AI 助手"""
        pass

    @abstractmethod
    def wait_for_completion(self, timeout: int) -> ExecutionResult:
        """等待任务完成"""
        pass

    @abstractmethod
    def close(self):
        """关闭 GUI 工具"""
        pass

    def run_task(self, context: TaskContext) -> ExecutionResult:
        """运行完整任务流程"""
        start_time = time.time()

        try:
            # 1. 启动 GUI
            if not self.launch(context):
                return ExecutionResult(False, "", "Failed to launch GUI", 0, [])

            # 2. 打开项目
            if not self.open_project(context.task_dir):
                return ExecutionResult(False, "", "Failed to open project", 0, [])

            # 3. 发送任务提示
            if not self.send_prompt(context.instruction):
                return ExecutionResult(False, "", "Failed to send prompt", 0, [])

            # 4. 等待完成
            result = self.wait_for_completion(context.time_limit)
            result.duration = time.time() - start_time

            return result

        finally:
            self.close()
```

### 4.3 JetBrains IDE 适配器（IntelliJ/PyCharm/GoLand 等）

```python
# jetbrains_adapter.py
from gui_bridge import GUIAgentBridge, TaskContext, ExecutionResult
import subprocess
import time
import os

class JetBrainsAdapter(GUIAgentBridge):
    """JetBrains IDE 统一适配器（支持 AI Assistant 插件）"""

    def __init__(self, ide_path: str, project_type: str = "python"):
        super().__init__()
        self.ide_path = ide_path
        self.project_type = project_type
        self.process = None

    @property
    def name(self) -> str:
        return "jetbrains-ai-assistant"

    @property
    def supported_platforms(self) -> list:
        return ["windows", "linux", "macos"]

    def launch(self, context: TaskContext) -> bool:
        """启动 JetBrains IDE"""
        self.process = subprocess.Popen([
            self.ide_path,
            context.task_dir
        ])
        time.sleep(10)  # 等待 IDE 启动
        return True

    def open_project(self, project_path: str) -> bool:
        """项目已在启动时打开"""
        return True

    def send_prompt(self, prompt: str) -> bool:
        """
        发送提示词到 JetBrains AI Assistant

        方案1: 使用 JetBrains Gateway/LSP（推荐）
        方案2: 使用 UI 自动化（pyautogui）
        方案3: 使用 REST API（如果 AI Assistant 提供）
        """
        # 使用 JetBrains LSP 或 REST API
        # 参考: https://plugins.jetbrains.com/docs/intellij/ai-assistant.html

        # 示例：通过 LSP 发送请求
        import requests
        try:
            response = requests.post(
                "http://localhost:63342/ai-assistant/chat",
                json={"prompt": prompt, "project": self.project_type},
                timeout=30
            )
            return response.status_code == 200
        except:
            # 降级到 UI 自动化
            return self._send_prompt_via_ui(prompt)

    def _send_prompt_via_ui(self, prompt: str) -> bool:
        """通过 UI 自动化发送提示词"""
        import pyautogui
        import pyperclip

        # 复制提示词到剪贴板
        pyperclip.copy(prompt)

        # 打开 AI Assistant（假设快捷键是 Ctrl+Shift+A）
        pyautogui.hotkey('ctrl', 'shift', 'a')
        time.sleep(2)

        # 粘贴并发送
        pyautogui.hotkey('ctrl', 'v')
        time.sleep(0.5)
        pyautogui.press('enter')

        return True

    def wait_for_completion(self, timeout: int) -> ExecutionResult:
        """等待 AI Assistant 完成响应"""
        start_time = time.time()
        output = ""

        while time.time() - start_time < timeout:
            # 检查 AI Assistant 响应是否完成
            # 可以通过 LSP 或检查 UI 状态
            time.sleep(5)

            # 收集输出（通过 LSP 或日志文件）
            # output += ...

        return ExecutionResult(True, output, "", time.time() - start_time, [])

    def close(self):
        """关闭 IDE"""
        if self.process:
            self.process.terminate()
            self.process = None
```

### 4.4 VS Code 适配器

```python
# vscode_adapter.py
from gui_bridge import GUIAgentBridge, TaskContext, ExecutionResult
import subprocess
import time
import os

class VSCodeAdapter(GUIAgentBridge):
    """VS Code 统一适配器（支持 GitHub Copilot / Copilot Chat）"""

    def __init__(self, vscode_path: str = "code"):
        super().__init__()
        self.vscode_path = vscode_path
        self.process = None

    @property
    def name(self) -> str:
        return "vscode-copilot"

    @property
    def supported_platforms(self) -> list:
        return ["windows", "linux", "macos"]

    def launch(self, context: TaskContext) -> bool:
        """启动 VS Code"""
        self.process = subprocess.Popen([
            self.vscode_path,
            "--folder-uri", f"file://{context.task_dir}",
            "--disable-extensions", "false"  # 保留 Copilot 扩展
        ])
        time.sleep(5)
        return True

    def open_project(self, project_path: str) -> bool:
        """项目已在启动时打开"""
        return True

    def send_prompt(self, prompt: str) -> bool:
        """
        发送提示词到 Copilot Chat

        方案1: 使用 VS Code CLI（推荐）
        方案2: 使用 VS Code Extension API
        方案3: 使用 UI 自动化
        """
        # 方案1: 使用 VS Code CLI
        try:
            result = subprocess.run([
                self.vscode_path,
                "--command", "github.copilot.chat.send",
                "--args", prompt
            ], capture_output=True, timeout=30)
            return result.returncode == 0
        except:
            return self._send_prompt_via_ui(prompt)

    def _send_prompt_via_ui(self, prompt: str) -> bool:
        """通过 UI 自动化发送提示词"""
        import pyautogui
        import pyperclip

        pyperclip.copy(prompt)

        # 打开 Copilot Chat（Ctrl+Shift+I 或 Ctrl+Alt+I）
        pyautogui.hotkey('ctrl', 'shift', 'i')
        time.sleep(2)

        pyautogui.hotkey('ctrl', 'v')
        time.sleep(0.5)
        pyautogui.press('enter')

        return True

    def wait_for_completion(self, timeout: int) -> ExecutionResult:
        """等待 Copilot Chat 完成响应"""
        start_time = time.time()
        output = ""

        while time.time() - start_time < timeout:
            time.sleep(5)
            # 收集输出

        return ExecutionResult(True, output, "", time.time() - start_time, [])

    def close(self):
        """关闭 VS Code"""
        if self.process:
            self.process.terminate()
            self.process = None
```

### 4.5 CodeArts Doer 适配器

```python
# codearts_doer_adapter.py
from gui_bridge import GUIAgentBridge, TaskContext, ExecutionResult
import subprocess
import time
import os

class CodeArtsDoerAdapter(GUIAgentBridge):
    """CodeArts Doer 统一适配器（华为 AI 编程助手）"""

    def __init__(self, doer_path: str):
        super().__init__()
        self.doer_path = doer_path
        self.process = None

    @property
    def name(self) -> str:
        return "codearts-doer"

    @property
    def supported_platforms(self) -> list:
        return ["windows"]  # CodeArts Doer 主要在 Windows 上运行

    def launch(self, context: TaskContext) -> bool:
        """启动 CodeArts Doer"""
        self.process = subprocess.Popen([
            self.doer_path,
            "--project", context.task_dir
        ])
        time.sleep(10)  # 等待应用启动
        return True

    def open_project(self, project_path: str) -> bool:
        """项目已在启动时打开"""
        return True

    def send_prompt(self, prompt: str) -> bool:
        """
        发送提示词到 CodeArts Doer

        方案1: 使用 CodeArts Doer API（如果有）
        方案2: 使用 UI 自动化（pyautogui/pywinauto）
        """
        # 尝试 API 方式
        try:
            import requests
            response = requests.post(
                "http://localhost:8080/api/chat",  # 假设的 API 端点
                json={"prompt": prompt},
                timeout=30
            )
            return response.status_code == 200
        except:
            return self._send_prompt_via_ui(prompt)

    def _send_prompt_via_ui(self, prompt: str) -> bool:
        """通过 UI 自动化发送提示词"""
        import pyautogui
        import pyperclip

        pyperclip.copy(prompt)

        # 打开 Doer 聊天窗口（假设快捷键）
        pyautogui.hotkey('ctrl', 'shift', 'd')
        time.sleep(2)

        pyautogui.hotkey('ctrl', 'v')
        time.sleep(0.5)
        pyautogui.press('enter')

        return True

    def wait_for_completion(self, timeout: int) -> ExecutionResult:
        """等待 Doer 完成响应"""
        start_time = time.time()
        output = ""

        while time.time() - start_time < timeout:
            time.sleep(5)
            # 收集输出

        return ExecutionResult(True, output, "", time.time() - start_time, [])

    def close(self):
        """关闭 CodeArts Doer"""
        if self.process:
            self.process.terminate()
            self.process = None
```

### 4.6 Windows 环境运行 Terminal-Bench 任务

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

**Windows 运行脚本**:

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

# 根据类型触发 GUI Agent
switch ($AgentType) {
    "vscode" {
        code --folder-uri "vscode-remote://wsl+docker/tasks/$TaskId"
    }
    "jetbrains" {
        & "C:\Program Files\JetBrains\IntelliJ IDEA\idea64.exe" "C:\tasks\$TaskId"
    }
    "codearts" {
        & "C:\Program Files\Huawei\CodeArts Doer\Doer.exe" "C:\tasks\$TaskId"
    }
}

# 等待任务完成
# ...
```

### 4.7 三种 GUI 工具适配对比

| 特性 | JetBrains IDE | VS Code | CodeArts Doer |
|------|--------------|---------|----------------|
| **AI 助手** | AI Assistant | GitHub Copilot | Doer AI |
| **自动化方式** | LSP / REST API | CLI / Extension API | API / UI 自动化 |
| **平台支持** | Win/Linux/Mac | Win/Linux/Mac | Windows |
| **扩展性** | 插件系统 | 扩展系统 | 内置 |
| **推荐自动化方案** | LSP | CLI | UI 自动化 |

---

## 5. 用例集构建建议

### 5.1 任务格式规范（兼容 Terminal-Bench 2.0）

```toml
# task.toml 完整模板
name = "unique-task-id"
description = "任务简短描述"
difficulty = "easy|medium|hard"
tags = ["tag1", "tag2", "tag3"]

# 时间限制（秒）
time_limit = 1800

# 预估时间（分钟）
expert_time_estimate_min = 60.0
junior_time_estimate_min = 30.0
```

```markdown
# instruction.md 模板

## Task Description

详细描述任务要求，包括：
- 背景说明
- 具体要求
- 预期输出

## Requirements

1. 要求1
2. 要求2
3. 要求3

## Hints (Optional)

- 提示1
- 提示2
```

### 5.2 目录结构

```
my-benchmark/
├── dataset.yaml           # 数据集配置
├── tasks/
│   ├── task-001/
│   │   ├── task.toml
│   │   ├── instruction.md
│   │   ├── environment/
│   │   │   └── Dockerfile
│   │   ├── solution/
│   │   │   └── solve.sh
│   │   └── tests/
│   │       ├── test.sh
│   │       └── test_outputs.py
│   ├── task-002/
│   │   └── ...
│   └── ...
├── agents/                # Agent 适配器
│   ├── claude_code.py
│   ├── opencode.py
│   └── gui_bridge/
│       ├── gui_bridge.py
│       ├── jetbrains_adapter.py
│       ├── vscode_adapter.py
│       └── codearts_doer_adapter.py
└── scripts/
    ├── run_cli.sh
    └── run_gui.ps1
```

### 5.3 鸶证脚本标准

```bash
#!/bin/bash
# tests/test.sh - 任务验证脚本

set -e

# 运行 pytest 测试
python -m pytest tests/test_outputs.py -v --tb=short

if [ $? -ne 0 ]; then
    echo "ERROR: Tests failed"
    exit 1
fi

echo "All tests passed!"
exit 0
```

```python
# tests/test_outputs.py - pytest 测试文件
import pytest

def test_basic_functionality():
    """测试基本功能"""
    # 实现测试逻辑
    assert True

def test_edge_cases():
    """测试边界情况"""
    # 实现测试逻辑
    assert True
```

### 5.4 多平台兼容性要求

为确保任务在 CLI 和 GUI 环境都能运行:

| 要求 | CLI | GUI |
|------|-----|-----|
| 任务描述 | instruction.md (Markdown) | instruction.md (Markdown) |
| 文件路径 | POSIX 路径 | 统一使用相对路径 |
| 验证方式 | tests/test.sh | tests/test.sh (相同) |
| 超时处理 | 进程 kill | 窗口关闭 |

---

## 6. 实施路线图

### 6.1 阶段规划

```
Phase 1: 基础建设（1周）
├── 搭建 Terminal-Bench 2.0 环境
├── 熟悉 Harbor 框架
└── 运行现有任务验证环境

Phase 2: CLI Agent 集成（1周）
├── 测试 Claude Code 集成
├── 测试 Opencode 集成
└── 编写自动化评测脚本

Phase 3: GUI Agent 统一适配层（2周）
├── 设计 GUI Bridge 接口
├── 实现 JetBrains 适配器
├── 实现 VS Code 适配器
├── 实现 CodeArts Doer 适配器
└── 测试 Windows 环境

Phase 4: 用例集构建（3周）
├── 设计任务模板
├── 编写新任务（建议50+）
├── 编写验证脚本
└── 人工验证任务可解性

Phase 5: 评测与优化（1周）
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
| Windows GUI 评测 | ⚠️ 需适配 | 建议实现统一 Bridge 层 |
| 自建未污染用例集 | ✅ 可行 | 复用 Terminal-Bench 格式 |

### 推荐方案

1. **短期**：直接使用 Terminal-Bench 2.0 + Harbor 框架评测 CLI Agent
2. **中期**: 开发统一 GUI Bridge 层，支持 JetBrains、VS Code 和 CodeArts Doer
3. **长期**: 基于 Terminal-Bench 格式构建私有用例集，避免数据污染

### 三种 GUI 工具统一方案优势

| 优势 | 说明 |
|------|------|
| **代码复用** | 统一的 GUIAgentBridge 基类减少重复代码 |
| **易于扩展** | 新增 GUI 工具只需实现适配器 |
| **维护简单** | 核心逻辑集中在 Bridge 层 |
| **一致性** | 所有 GUI 工具使用相同的评测流程 |

---

## 参考资料

- [Terminal-Bench 2.0 GitHub](https://github.com/harbor-framework/terminal-bench-2)
- [Terminal-Bench Pro (阿里)](https://github.com/alibaba/terminal-bench-pro)
- [Harbor 框架文档](https://github.com/laude-institute/harbor)
- [Terminal-Bench 论文](https://arxiv.org/html/2601.11868v1)
- [JetBrains AI Assistant](https://plugins.jetbrains.com/docs/intellij/ai-assistant.html)
- [VS Code Copilot](https://code.visualstudio.com/docs/copilot)
- [CodeArts Doer](https://www.huaweicloud.com/product/codearts.html)

---

*文档版本：2.0*
*最后更新：2026-04-01*
