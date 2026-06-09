明白了。您是想评估 **“自研 IDE + 其内置模型”** 这个端到端系统的代码修复能力，而不是仅仅评估模型本身。SWE‑bench 本质上是一个 **模型评估基准**，但完全可以用它来评测一个包含 IDE 交互、模型推理、甚至人工辅助的系统。

下面我将您的需求梳理成一个可落地的技术方案，并以 Markdown 格式呈现。

---

## 需求梳理：使用 SWE‑bench_Pro 评估“自研 IDE + 内置模型”

### 1. 目标

评测一个**集成了大语言模型的 IDE**（例如类似 Cursor、GitHub Copilot 的智能 IDE，或者您自研的具备代码理解/生成能力的开发环境）在实际软件工程问题中的端到端修复能力。

评测指标与 SWE‑bench 标准一致：**能否生成正确的补丁**，使得原本失败的单元测试（FAIL_TO_PASS）变为通过，同时不破坏已有的通过测试（PASS_TO_PASS）。

### 2. 核心原则

- **评估框架不变**：仍使用官方 SWE‑bench 的 Docker 环境、测试逻辑和评分脚本。
- **替换模型调用层**：不使用 SWE‑agent 或 API 自动调用，而是**人工在 IDE 中操作**，最终产出一个 `model_patch`。
- **严格隔离**：所有代码浏览、修改、调试都必须**在对应任务的 Docker 容器内**进行，以保证环境与评估时完全一致。

### 3. 整体流程（5 个阶段）

#### 阶段 1：环境准备

- 克隆 SWE‑bench_Pro 离线数据集（已有）。
- 安装 `swebench`、`datasets` 等依赖（建议使用虚拟环境）。
- 确保 Docker Desktop 运行，且分配足够内存（≥8GB）。

#### 阶段 2：导出待测任务列表并准备容器

```bash
# 从数据集中提取所有 instance_id，或者只选一个子集（如 Lite）
python helper_code/extract_gold_patches.py \
    --dataset_path /path/to/dataset \
    --output_dir ./my_ide_eval
```

对每个想要评测的 `instance_id`：

```python
from swebench.harness.docker_build import build_instance_image

image_tag = build_instance_image(instance_id)
# 启动容器，并挂载 IDE 工作目录（如果需要）
docker run -it -v /path/to/ide/workspace:/workspace --name ide_$instance_id $image_tag bash
```

#### 阶段 3：在 IDE 中完成人工辅助的代码修复

- **容器内操作**：所有代码修改必须在容器内的 `/testbed/` 目录中进行。
- **IDE 集成方式**（任选一种）：
  - **方式 A（完全人工）**：开发者在 IDE 中阅读 `problem_statement`，手动编写补丁 → 记录模型能力较弱，仅测试 IDE 本身。
  - **方式 B（模型辅助）**：IDE 内置模型根据问题描述和代码上下文**自动生成建议补丁**，开发者可接受/修改 → 评估“IDE + 模型”协同效果。
  - **方式 C（半自动）**：IDE 直接调用模型生成补丁，开发者只负责触发和确认。
- **生成补丁**：修改完成后，在容器内执行：
  ```bash
  git diff > /workspace/fix_${instance_id}.patch
  ```
- **导出补丁**：`docker cp` 到宿主机上的统一目录（如 `./patches/`）。

#### 阶段 4：收集补丁并生成预测文件

使用官方 `gather_patches.py`（或手动编写脚本）将所有 `.patch` 文件转换为单个 `predictions.jsonl`：

```bash
python helper_code/gather_patches.py \
    --patch_dir ./patches \
    --output predictions.jsonl
```

每行的格式为：
```json
{"instance_id": "owner__repo-123", "model_name_or_path": "my_ide_model", "model_patch": "diff --git ..."}
```

#### 阶段 5：运行官方评估

**先验证黄金补丁**（可选但推荐）：
```bash
python -m swebench.harness.run_evaluation \
    --dataset_name /path/to/dataset \
    --predictions_path gold \
    --run_id validate_gold
```

**再评估您的补丁**：
```bash
python -m swebench.harness.run_evaluation \
    --dataset_name /path/to/dataset \
    --predictions_path ./predictions.jsonl \
    --run_id my_ide_eval \
    --max_workers 1   # 人工操作一般一次一个实例，避免资源冲突
```

查看结果：`logs/swebench/my_ide_eval/report.json` 中的 `resolved` 字段即为通过率。

### 4. 关键实现细节

| 需求点 | 实现方式 |
|--------|----------|
| IDE 如何访问容器内的代码 | 将容器内的 `/testbed/` 映射到宿主机目录（`-v`），然后 IDE 打开该宿主机目录。注意文件权限。 |
| 模型如何调用 | IDE 后端直接调用本地/云端模型 API，输入 `problem_statement` 和当前代码上下文。 |
| 调试与测试 | 可在容器内手动运行 pytest 来验证 `FAIL_TO_PASS` 中的测试是否通过。 |
| 可重复性 | 每个容器独立，基线与数据集给定的 `base_commit` 保持一致。 |
| 评估指标 | 官方报告会给出 `resolved`（正确修复数 / 总任务数）。 |

### 5. 可选扩展

- **批量评估**：若 IDE 支持完全自动化（例如插件监听 issue 并自动修复），可设置 `--max_workers > 1` 并发运行多个容器。
- **记录决策过程**：记录 IDE 生成的原始补丁、人工修改后的补丁，分析模型贡献度。
- **对比实验**：用同一组任务评测“仅模型（无 IDE 交互）”与“IDE+模型”，量化 IDE 带来的提升。

### 6. 注意事项

- **Docker 资源**：同时运行多个容器会消耗大量内存/CPU，建议根据硬件逐步增加并发。
- **测试超时**：某些测试执行时间较长，可在 `run_evaluation` 中设置 `--timeout`。
- **补丁格式**：`model_patch` 必须是 **unified diff** 格式，可使用 `git diff` 生成。
- **环境一致性**：务必保证 IDE 内的 Python 解释器、依赖版本与容器内一致（可直接使用容器内的 Python）。

---

如果您需要更具体的某个脚本示例（例如如何自动从 IDE 收集补丁并生成 JSONL），我可以进一步提供代码片段。
