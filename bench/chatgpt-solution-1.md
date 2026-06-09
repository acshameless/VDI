下面我把你的方案扩展成一个**生产级、可审计、可复现、可对比不同 IDE/模型能力的 SWE-bench_Pro 评测方案**。

---

# 使用 SWE-bench_Pro 评估「自研 IDE + 内置模型」的生产级测试方案

---

# 1. 测试目标（Objective）

## 1.1 测试对象

待评估对象：

* 自研 AI IDE
* IDE 内置 Agent
* IDE 集成的大模型
* IDE + Model 的联合工作流

典型例子：

* Cursor
* Windsurf
* Augment
* GitHub Copilot Workspace
* 自研 Coding Agent IDE

---

## 1.2 测试问题

核心问题：

> 在真实软件工程任务中，该 IDE + Model 能否独立定位问题、理解代码库、修改代码，并生成正确补丁？

---

## 1.3 评估标准

完全遵循 SWE-bench 官方定义：

### Success

生成补丁后：

* FAIL_TO_PASS 全部通过

且

* PASS_TO_PASS 不被破坏

即：

```text
Original:
  FAIL_TO_PASS = Fail
  PASS_TO_PASS = Pass

Patched:
  FAIL_TO_PASS = Pass
  PASS_TO_PASS = Pass
```

记为：

```text
Resolved = True
```

---

# 2. 评测原则（Evaluation Principles）

## 2.1 不修改官方评测框架

保留：

```text
SWE-bench Docker
SWE-bench Test Runner
SWE-bench Scoring
SWE-bench Dataset
```

全部官方实现。

---

## 2.2 替换 Agent 执行层

传统：

```text
Issue
 ↓
SWE-agent
 ↓
Patch
 ↓
Evaluate
```

本方案：

```text
Issue
 ↓
人工 + IDE + 内置模型
 ↓
Patch
 ↓
Evaluate
```

---

## 2.3 环境一致性

必须保证：

开发环境 = 评测环境

即：

```text
Docker Container
    =
IDE访问的代码环境
```

禁止：

```text
Host机器代码
```

否则结果不可信。

---

## 2.4 结果可追溯

每次实验必须保存：

* Prompt
* 对话记录
* Patch
* Commit
* 日志

方便后续审计。

---

# 3. 测试架构

## 整体架构

```text
SWE-bench_Pro Dataset
           │
           ▼
     Instance
           │
           ▼
 Docker Container
           │
           ▼
   IDE Remote Attach
           │
           ▼
    AI Model Assist
           │
           ▼
   Human-in-the-loop
           │
           ▼
      Git Patch
           │
           ▼
   SWE-bench Eval
           │
           ▼
      Metrics
```

---

# 4. 阶段1：环境准备

---

## 4.1 基础环境

建议：

```text
Ubuntu 22.04
Docker 24+
Python 3.11
Git
```

---

## 4.2 创建虚拟环境

```bash
python -m venv .venv
source .venv/bin/activate
```

---

## 4.3 安装 SWE-bench

```bash
pip install swebench
pip install datasets
pip install docker
```

---

## 4.4 克隆 SWE-bench_Pro

```bash
git clone <offline_dataset_repo>
```

目录：

```text
swebench-pro/
├── instances.jsonl
├── metadata
└── evaluation
```

---

## 4.5 Docker资源配置

推荐：

```text
CPU      ≥ 8 Core
Memory   ≥ 16GB
Disk     ≥ 100GB
```

最低：

```text
CPU      ≥ 4 Core
Memory   ≥ 8GB
```

---

# 5. 阶段2：导出任务并准备容器

---

## 5.1 选择评测集

建议：

### Smoke Test

```text
10 Tasks
```

用于验证流程。

---

### Benchmark

```text
100 Tasks
```

用于模型对比。

---

### Full Run

```text
全部任务
```

最终报告。

---

## 5.2 导出实例

例如：

```python
from datasets import load_dataset

ds = load_dataset("princeton-nlp/SWE-bench_Pro")
```

导出：

```json
{
  "instance_id":"django__django-12345",
  "repo":"django/django",
  "base_commit":"xxxxx",
  "problem_statement":"..."
}
```

---

## 5.3 构建容器

每个任务：

```bash
docker build ...
```

生成：

```text
instance_id
    ↓
docker image
```

---

## 5.4 容器命名规范

建议：

```text
swebench-{instance_id}
```

例如：

```text
swebench-django-12345
```

---

# 6. 阶段3：IDE 中完成人工辅助修复

这是整个方案最关键部分。

---

# 6.1 IDE如何访问容器内代码

推荐方案：

## VS Code Remote Containers

```bash
code .
```

Attach：

```text
Remote Explorer
    ↓
Attach to Running Container
```

---

或：

## JetBrains Gateway

连接：

```text
Docker Container
```

---

或：

## 自研 IDE Remote Workspace

要求：

```text
Workspace Root
=
Container Filesystem
```

例如：

```text
/workspace/repo
```

---

## 为什么必须这样？

因为：

```text
测试环境
=
开发环境
```

避免：

```text
本地 Python版本
本地依赖
本地缓存
```

影响结果。

---

# 6.2 模型如何调用

允许：

### IDE内置模型

例如：

```text
Claude
GPT
Gemini
DeepSeek
Qwen
```

---

### 本地模型

例如：

```text
DeepSeek-Coder
Qwen3-Coder
CodeLlama
```

---

### Agent模式

例如：

```text
Ask
Edit
Agent
Autonomous
```

均允许。

---

## 必须记录

每个任务：

```text
模型名称
模型版本
上下文长度
温度
Agent模式
```

例如：

```json
{
  "model":"Claude Sonnet 4",
  "temperature":0,
  "mode":"Agent"
}
```

---

# 6.3 人工操作规范

允许：

```text
阅读Issue
阅读代码
搜索代码
运行测试
查看日志
让模型修改代码
手工修改代码
```

---

禁止：

### 查看黄金补丁

```text
gold_patch
```

---

### 查看 PR

```text
linked PR
```

---

### 搜索最终答案

例如：

```text
GitHub PR
GitHub Commit
Issue Discussion
```

否则属于数据泄漏。

---

# 6.4 调试与测试

允许：

```bash
pytest
```

```bash
tox
```

```bash
python -m pytest
```

---

允许重复：

```text
Edit
Run Test
Edit
Run Test
```

直到收敛。

---

## 建议记录

```text
测试次数
token消耗
耗时
```

---

# 7. 阶段4：收集补丁

---

## 7.1 获取最终Patch

容器内：

```bash
git diff
```

导出：

```bash
git diff > patch.diff
```

---

## 7.2 Patch格式

必须：

```diff
diff --git a/...
```

标准 git patch。

---

## 7.3 生成预测文件

SWE-bench标准：

```json
[
  {
    "instance_id":"django__django-12345",
    "model_name_or_path":"my_ide_v1",
    "model_patch":"<git patch>"
  }
]
```

---

## 7.4 推荐增加元数据

```json
{
  "instance_id":"...",
  "model":"...",
  "agent_mode":"...",
  "elapsed_minutes":35,
  "test_runs":7,
  "tokens":52000
}
```

用于后续分析。

---

# 8. 阶段5：运行官方评估

---

## 8.1 黄金补丁验证（推荐）

先验证：

```text
gold_patch
```

是否在当前环境通过。

目的：

* 检查环境正确
* 检查镜像正确

---

## 8.2 评估预测结果

```bash
python -m swebench.harness.run_evaluation \
  --dataset_name SWE-bench_Pro \
  --predictions_path predictions.json
```

（具体命令以当前官方版本为准）

---

## 8.3 输出结果

典型输出：

```json
{
  "resolved": 41,
  "total": 100,
  "resolve_rate": 41.0
}
```

---

# 9. 关键实现细节（补全）

| 需求点           | 实现方式                                                                              |
| ------------- | --------------------------------------------------------------------------------- |
| IDE 如何访问容器内代码 | VS Code Remote Containers / JetBrains Gateway / 自研 Remote Workspace Attach Docker |
| 模型如何调用        | IDE内置模型、本地模型、Agent模式均可；记录模型版本与配置                                                  |
| 调试与测试         | 全部在 SWE-bench Docker Container 内执行 pytest/tox，不允许使用宿主环境                           |
| 可重复性          | 固定数据集版本、固定 Docker Image、固定 Base Commit、固定模型版本、保存完整 Patch 与日志                      |
| 评估指标          | Resolve Rate、FAIL_TO_PASS、PASS_TO_PASS、Patch Success、平均耗时、Token Cost              |
| 数据泄漏控制        | 禁止查看 gold patch、PR、commit、issue solution                                          |
| 审计能力          | 保存 Prompt、聊天记录、终端日志、Patch、评测结果                                                    |
| 成本统计          | Token、运行时长、人工介入次数                                                                 |
| 横向对比          | 同一任务集评测 Cursor / Copilot / 自研 IDE / Agent                                         |
| 失败分析          | 分类 Root Cause（定位失败、理解失败、实现失败、测试失败）                                                |

---

# 10. 生产级增强（推荐）

如果目标是对外发布 Benchmark 或内部模型验收，建议额外增加：

### A. Blind Evaluation

操作者不知道：

```text
gold patch
最终答案
```

---

### B. 双人复核

```text
Engineer A 生成补丁
Engineer B 复核
```

---

### C. 录像留档

保存：

```text
屏幕录制
终端日志
模型对话
```

---

### D. 自动化实验管理

推荐接入：

* MLflow
* Weights & Biases
* Langfuse
* OpenTelemetry

记录：

```text
任务ID
模型版本
Prompt
耗时
成本
结果
```

---

# 最终产出

经过上述方案后，你得到的不仅是一个“SWE-bench 跑通流程”，而是一套可用于：

* 自研 AI IDE 验收
* Cursor/Copilot/Windsurf 对比测试
* 模型版本回归测试
* Agent 能力评估
* 对外 Benchmark 发布

的**生产级 SWE-bench_Pro Human-in-the-Loop Evaluation Framework**。其中最重要的一条原则是：

> **开发、调试、测试、生成补丁的全过程都发生在 SWE-bench 官方任务容器内，最终仅以标准 `model_patch` 作为评测输入，由官方评测框架计算结果。**

这样得到的分数才与 SWE-bench 社区结果具有可比性。
