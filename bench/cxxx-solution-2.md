明白了，你的需求就是像 Codex / Claude Code 那样，**把自己的 Agent CLI + 它自己的模型**作为一个独立的“已安装代理”接入 Terminal-Bench 进行官方认可的测试。

以下是根据这一需求整理的 Markdown 文档。

```markdown
# 将自研 Agent CLI 作为“已安装代理”接入 Terminal-Bench 的需求文档

## 1. 测试目标
- **测试对象**：自研的本地 Agent CLI 工具 + 该工具内置或调用的语言模型（如 Claude、GPT、本地模型等）。
- **测试平台**：[Terminal-Bench](https://github.com/harbor-framework/terminal-bench)
- **集成方式**：遵循 Terminal-Bench 定义的 **“已安装代理（Installed Agent）”** 规范，使自研 Agent CLI 能够像官方支持的 `claude-code`、`codex` 一样，在 Docker 容器内被自动安装、执行，并完成真实终端任务的评估。

## 2. 集成原理
Terminal-Bench 框架提供了 `AbstractInstalledAgent` 抽象基类。通过实现该接口，可以将任何**命令行形式的代理工具**（如 `claude-code`、`your-agent`）封装成一个可被框架识别的插件。

- 框架负责：拉取任务对应的 Docker 镜像、启动容器、准备环境变量、调用你实现的适配器。
- 你的适配器负责：在容器内安装 Agent CLI、执行任务、返回执行结果。
- 测试结果仅取决于你的 Agent CLI 在容器内的实际表现，**不涉及 `terminus` 代理**。

## 3. 适配器实现要求

### 3.1 安装脚本 (`setup.sh`)
你需要提供一个 Shell 脚本，用于在干净的 Docker 容器内安装你的 Agent CLI。该脚本应：

- **可独立运行**，不依赖容器内未预装的工具（但可以安装必要依赖，如 `curl`、`unzip`）。
- **安装完成后**，确保 `your-agent` 命令可以在容器的 `PATH` 中找到。
- **错误处理**：使用 `set -e` 或在每个关键步骤后检查返回值。

示例脚本框架：
```bash
#!/bin/bash
set -e

# 安装必要系统依赖
apt-get update && apt-get install -y curl

# 下载并安装你的 Agent 二进制文件
curl -L https://your.domain/downloads/latest/your-agent -o /usr/local/bin/your-agent
chmod +x /usr/local/bin/your-agent

echo "Your Agent installed successfully"
```

### 3.2 Python 适配器类
创建一个 Python 文件（例如 `your_agent_adapter.py`），继承 `AbstractInstalledAgent` 并实现以下关键成员：

| 方法/属性 | 作用 | 必需 |
| :--- | :--- | :--- |
| `name() -> AgentName` | 返回唯一的 Agent 标识符，如 `AgentName("your-agent")` | ✅ |
| `_install_agent_script_path -> Path` | 返回 `setup.sh` 的本地文件路径 | ✅ |
| `_run_agent_commands() -> list[TerminalCommand]` | 返回在容器内启动 Agent 并执行任务的命令列表 | ✅ |
| `_env -> dict[str, str]` | 设置容器内的环境变量（如 API Key、代理配置） | 强烈推荐 |
| `_timeout_multiplier -> float` | 可选，增大任务超时倍数（如果你的 Agent 需要更长思考时间） | 可选 |

**示例实现**：
```python
from pathlib import Path
from terminal_bench.agents.installed_agents.abstract_installed_agent import AbstractInstalledAgent
from terminal_bench.agents.agent_types import AgentName
from terminal_bench.agents.terminal_command import TerminalCommand

class YourAgentAdapter(AbstractInstalledAgent):

    @staticmethod
    def name() -> AgentName:
        return AgentName("your-agent")

    @property
    def _install_agent_script_path(self) -> Path:
        return Path(__file__).parent / "setup.sh"

    def _run_agent_commands(self) -> list[TerminalCommand]:
        # 假设你的 Agent 接受任务描述作为命令行参数，并自动完成任务后退出
        return [
            TerminalCommand(
                command=f"your-agent solve '{self.task.instruction}'",
                is_blocking=True
            )
        ]

    @property
    def _env(self) -> dict[str, str]:
        # 将宿主机的 API 密钥等传入容器
        return {
            "OPENAI_API_KEY": self._api_key,   # 根据你的模型修改
            "YOUR_AGENT_HOME": "/tmp/your-agent"
        }
```

## 4. 运行测试命令

完成适配器代码后，使用 `--agent-import-path` 运行：

```bash
tb run \
  --dataset terminal-bench-core==0.1.1 \
  --agent-import-path your_agent_adapter:YourAgentAdapter \
  --task-id blind-maze-explorer-5x5
```

- `--agent-import-path` 格式为 `模块名:类名`。
- 可以替换 `--task-id` 为其他任务，或使用 `--all` 运行全量测试（不推荐首次使用）。

## 5. 测试结果的意义

- **评估对象**：你的 **Agent CLI + 它调用的模型** 组合，是一个端到端的完整智能体。
- **结果的可比性**：由于你的 Agent 被视作第三方工具，其得分不会出现在官方 `terminus` 排行榜上。但你可以：
  - 与官方已测试的 `claude-code`、`codex` 等工具的成绩进行横向对比（前提是任务一致、环境相同）。
  - 用于内部迭代，量化每次 Agent 改进带来的性能变化。
- **可重复性**：完整的 Docker 环境确保测试结果在不同机器上可复现。

## 6. 验证与调试建议

1. **本地小范围测试**：在非容器环境中先用 `your-agent solve "some simple task"` 验证 CLI 本身工作正常。
2. **单任务快速验证**：选择最简单的任务（如 `hello-world`）运行，确认 Docker 能正常安装并执行。
3. **查看详细日志**：运行时可加 `--verbose` 参数，检查容器内安装和执行的输出。
4. **环境变量传递**：确保 API Key 等敏感信息不被硬编码，通过 `_env` 动态传递。

## 7. 参考资料

- Terminal-Bench GitHub：[harbor-framework/terminal-bench](https://github.com/harbor-framework/terminal-bench)
- 内置已安装代理示例：查看 `terminal_bench/agents/installed_agents/claude_code_agent.py` 和 `codex_agent.py`
- 抽象基类定义：`terminal_bench/agents/installed_agents/abstract_installed_agent.py`

## 8. 附录：与传统 “terminus” 测试的对比

| 项目 | 使用 `terminus` | 使用你的 Installed Agent |
| :--- | :--- | :--- |
| 被测试对象 | 纯语言模型（通过 API） | 你的 Agent CLI + 其内部模型 |
| 是否需要适配器 | 不需要 | 需要 |
| 是否可对比官方榜单 | 是 | 否（仅可自比或与同类工具比） |
| 测试成本 | API 调用费用 | API 调用费用 + 容器执行时间 |
| 适合场景 | 模型能力基准测试 | Agent 产品化、工程优化评估 |

> **版本**：1.0  
> **最后更新**：2026-06-09  
> **目标用户**：希望将自己的智能体 CLI 接入 Terminal-Bench 进行端到端性能评估的开发者
```

这份文档直接对应你的真实需求：将自研 Agent + 模型视为一个整体，作为“已安装代理”接入官方测试体系。如果需要进一步细化某一部分（比如环境变量安全传递、多命令执行时序等），我可以继续补充。
