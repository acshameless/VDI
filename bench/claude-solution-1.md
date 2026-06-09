# SWE-bench_Pro 评测方案：自研 IDE + 内置模型 端到端验证

> **文档版本**: v1.0 — 生产级验证方案  
> **适用框架**: SWE-bench_Pro (ScaleAI) | Docker 容器化评估  
> **核心原则**: 评估框架不变 · 替换模型调用层 · 严格环境隔离

---

## 目录

1. [方案概述与架构](#1-方案概述与架构)
2. [关键实现细节（完整填表）](#2-关键实现细节完整填表)
3. [阶段 1：环境准备](#3-阶段-1环境准备)
4. [阶段 2：导出任务列表并准备容器](#4-阶段-2导出任务列表并准备容器)
5. [阶段 3：IDE 辅助代码修复](#5-阶段-3ide-辅助代码修复)
6. [阶段 4：收集补丁并生成预测文件](#6-阶段-4收集补丁并生成预测文件)
7. [阶段 5：运行官方评估](#7-阶段-5运行官方评估)
8. [可重复性与质量管控](#8-可重复性与质量管控)
9. [常见问题与排查](#9-常见问题与排查)
10. [附录：脚本与模板](#10-附录脚本与模板)

---

## 1. 方案概述与架构

### 1.1 整体数据流

```
┌─────────────────────────────────────────────────────────────────┐
│                    评测主机（宿主机）                             │
│                                                                  │
│  ┌─────────────────┐     任务列表      ┌──────────────────────┐ │
│  │  SWE-bench_Pro  │ ────────────────► │  任务调度 / 工作区   │ │
│  │  HuggingFace    │                   │  scripts/  patches/  │ │
│  │  数据集         │                   └────────┬─────────────┘ │
│  └─────────────────┘                           │               │
│                                                │ docker run    │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │              Docker 容器（jefzda/sweap-images）            │ │
│  │                                                           │ │
│  │   /repo/  ← 目标代码库（已 checkout 至 base_commit）      │ │
│  │      ↑                          ↑                         │ │
│  │  VS Code Remote            IDE 内置模型调用                 │ │
│  │  (Dev Containers)          (推理 → 生成 patch)              │ │
│  │                                                           │ │
│  │   git diff HEAD > /patches/<instance_id>.patch            │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌─────────────────┐      JSON       ┌──────────────────────┐  │
│  │  patch 收集脚本  │ ─────────────► │  swe_bench_pro_      │  │
│  │  gather_patches │                │  eval.py (评分)       │  │
│  └─────────────────┘                └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 与标准 SWE-bench 评估的差异对照

| 维度 | 标准 SWE-agent 流程 | 本方案（自研 IDE） |
|------|-------------------|----------------|
| 补丁生成主体 | 全自动 Agent 循环 | 人工在 IDE 中操作，内置模型辅助 |
| 模型调用方式 | API 批量调用 | IDE 内置推理（可离线/私有部署） |
| 评估脚本 | `swe_bench_pro_eval.py` | **相同**（不变） |
| Docker 环境 | 相同 | **相同**（不变） |
| 补丁格式 | unified diff JSON | **相同**（不变） |
| 评估指标 | FAIL_TO_PASS / PASS_TO_PASS | **相同**（不变） |

---

## 2. 关键实现细节（完整填表）

| 需求点 | 实现方式 | 说明 |
|--------|---------|------|
| **IDE 如何访问容器内的代码** | VS Code **Dev Containers** 插件（`ms-vscode-remote.remote-containers`）通过 `docker exec` 将 VS Code Server 注入容器，直接在容器内的 `/repo` 目录打开工程；或使用 `docker cp` / 绑定挂载（`-v`）将代码暴露至宿主机路径 | 容器启动时带 `--name <instance_id>` 参数，Dev Container 按容器名连接；绑定挂载方案需在 `docker run` 加 `-v /host/workspace:/repo`，但会破坏容器纯洁性，**仅用于只读浏览**，写回须通过 `docker cp` |
| **模型如何调用** | IDE 内置推理引擎（如自研 LSP 插件）在容器内以 `localhost` 或 Unix Socket 暴露推理接口；VS Code Extension 通过 `vscode.commands` 触发补全/修改；亦可通过 `docker exec -it <id> bash` 在容器内直接调用模型 CLI | 若模型部署在宿主机，容器内通过 `host.docker.internal:<PORT>` 访问；若需 GPU，需加 `--gpus all` 参数启动容器并确认 CUDA 驱动版本兼容 |
| **调试与测试** | 在容器内运行 `python -m pytest <test_file> -x -v` 验证修复效果；使用 VS Code 集成终端（已连接容器）或 `docker exec` 执行命令；FAIL_TO_PASS 测试必须全部通过，PASS_TO_PASS 测试不能新增失败 | SWE-bench_Pro 每个实例的 `FAIL_TO_PASS` 和 `PASS_TO_PASS` 字段明确列出需验证的测试名称，可提前读取并在容器内有针对性地运行 |
| **可重复性** | 每次评估从镜像 `jefzda/sweap-images:<dockerhub_tag>` 全新启动容器（`docker run --rm`），不复用已修改容器；git 操作在容器内的 `/repo` 独立进行，宿主机保留 patch 文件备份 | 禁止将修改后的容器 commit 为新镜像用于后续评估；每个 instance 必须使用对应的专属镜像 tag，确保 base_commit 精确匹配 |
| **评估指标** | **Resolved Rate（%）**：`FAIL_TO_PASS` 测试全部通过 **且** `PASS_TO_PASS` 测试无新增失败的实例数 / 总实例数；官方脚本自动输出 `results.json`，包含每个实例的 `resolved` 布尔值和汇总 | 补充指标：partial credit（通过了部分 FAIL_TO_PASS）、patch size 分布、fix category 分析（bug/feature/refactor）；可对比金标准（gold patch）的 resolved rate 作为上界 |

---

## 3. 阶段 1：环境准备

### 3.1 宿主机依赖安装

```bash
# ---- Python 环境 ----
python3 -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate

pip install --upgrade pip
pip install datasets huggingface_hub  # 数据集访问
pip install pandas tqdm               # 任务管理辅助

# ---- 克隆 SWE-bench_Pro 评估框架 ----
git clone https://github.com/scaleapi/SWE-bench_Pro-os.git
cd SWE-bench_Pro-os
pip install -r requirements.txt

# ---- (可选) Modal 云端评估 ----
pip install modal
modal setup                           # 按提示生成 token，写入 ~/.modal.toml
```

### 3.2 Docker 配置（关键）

```bash
# 验证 Docker 版本（需 ≥ 20.10）
docker --version
docker info | grep -E "Total Memory|CPUs"

# Docker Desktop（macOS/Windows）资源建议：
#   CPU: ≥ 8 核
#   RAM: ≥ 16 GB（SWE-bench_Pro 任务较重）
#   Disk: ≥ 120 GB

# 若使用 Apple Silicon（M 系列），注意：
# SWE-bench_Pro 镜像为 linux/amd64，需启用 Rosetta 仿真
# Docker Desktop → Settings → Features in Development → "Use Rosetta..."
```

### 3.3 VS Code 插件安装

```bash
# 必装
code --install-extension ms-vscode-remote.remote-containers

# 推荐（你的 IDE 内置模型需要的插件）
code --install-extension <your-ide-llm-extension-id>
```

### 3.4 工作目录结构初始化

```bash
mkdir -p sweap-eval/{tasks,patches,logs,results,workspace}
cd sweap-eval

# 目录说明：
# tasks/      存放任务列表 JSON 和数据集缓存
# patches/    每个 instance 的 .patch 文件（原始）
# logs/       docker 运行日志
# results/    评估结果输出
# workspace/  （可选）绑定挂载时使用的宿主机代码副本
```

---

## 4. 阶段 2：导出任务列表并准备容器

### 4.1 加载数据集并导出任务清单

**脚本 `tasks/01_export_tasks.py`**：

```python
#!/usr/bin/env python3
"""
从 HuggingFace 加载 SWE-bench_Pro，导出任务清单及容器启动脚本。
"""
import json
import os
from datasets import load_dataset

# ------ 配置 ------
OUTPUT_DIR = "tasks"
DATASET_NAME = "ScaleAI/SWE-bench_Pro"
SPLIT = "test"

# 可选：只测试子集，调试时用
SUBSET_IDS = None  # 或 ["instance_abc123", "instance_xyz456"]

# ----- 加载 -----
print(f"Loading {DATASET_NAME} ...")
dataset = load_dataset(DATASET_NAME, split=SPLIT)
print(f"Total instances: {len(dataset)}")

tasks = []
for row in dataset:
    if SUBSET_IDS and row["instance_id"] not in SUBSET_IDS:
        continue
    tasks.append({
        "instance_id":      row["instance_id"],
        "repo":             row["repo"],
        "repo_language":    row["repo_language"],
        "base_commit":      row["base_commit"],
        "dockerhub_tag":    row["dockerhub_tag"],          # 专属镜像 tag
        "problem_statement": row["problem_statement"],
        "requirements":     row.get("requirements", ""),
        "interface":        row.get("interface", ""),
        "fail_to_pass":     json.loads(row["FAIL_TO_PASS"]),
        "pass_to_pass":     json.loads(row["PASS_TO_PASS"]),
    })

# 保存任务清单
os.makedirs(OUTPUT_DIR, exist_ok=True)
task_file = os.path.join(OUTPUT_DIR, "task_list.json")
with open(task_file, "w") as f:
    json.dump(tasks, f, indent=2, ensure_ascii=False)

print(f"Exported {len(tasks)} tasks → {task_file}")

# 生成容器启动脚本（bash）
shell_file = os.path.join(OUTPUT_DIR, "02_pull_images.sh")
with open(shell_file, "w") as f:
    f.write("#!/usr/bin/env bash\n# Auto-generated: pull all required Docker images\n\n")
    for t in tasks:
        tag = t["dockerhub_tag"]
        f.write(f'echo "Pulling jefzda/sweap-images:{tag} ..."\n')
        f.write(f'docker pull jefzda/sweap-images:{tag}\n')
        f.write(f'echo "Done: {t[\"instance_id\"]}"\n\n')

os.chmod(shell_file, 0o755)
print(f"Docker pull script → {shell_file}")
```

运行：

```bash
cd sweap-eval
python tasks/01_export_tasks.py
```

### 4.2 批量拉取 Docker 镜像

```bash
# 先预拉取（避免评估时网络超时）
bash tasks/02_pull_images.sh 2>&1 | tee logs/pull_images.log

# 验证镜像数量
docker images | grep sweap-images | wc -l
```

### 4.3 为单个任务启动容器（标准操作流程）

**脚本 `tasks/03_start_instance.sh`**：

```bash
#!/usr/bin/env bash
# 用法: bash 03_start_instance.sh <instance_id> <dockerhub_tag>
# 示例: bash 03_start_instance.sh instance_abc123 python.django-4a8f2c1

INSTANCE_ID="$1"
DOCKER_TAG="$2"
IMAGE="jefzda/sweap-images:${DOCKER_TAG}"

echo "==> Starting container for: ${INSTANCE_ID}"
echo "    Image: ${IMAGE}"

docker run \
    --name "${INSTANCE_ID}" \
    --rm \
    --interactive \
    --tty \
    --memory="8g" \
    --cpus="4" \
    --workdir /repo \
    "${IMAGE}" \
    /bin/bash

# 注意：
# --rm        容器退出后自动删除（保持环境纯洁）
# --name      按 instance_id 命名，方便 VS Code Remote 连接
# -it         交互式（调试用）；自动化时替换为 -d（后台运行）
```

**后台运行版本（自动化批量）**：

```bash
docker run -d \
    --name "${INSTANCE_ID}" \
    --memory="8g" \
    --cpus="4" \
    --workdir /repo \
    "${IMAGE}" \
    /bin/bash -c "tail -f /dev/null"   # 保持容器存活，等待操作
```

---

## 5. 阶段 3：IDE 辅助代码修复

### 5.1 VS Code Dev Containers 接入容器

容器启动后（后台运行模式），在 VS Code 中：

```
① 打开命令面板 (Cmd/Ctrl + Shift + P)
② 输入: "Dev Containers: Attach to Running Container..."
③ 选择容器名称（即 instance_id）
④ VS Code Server 注入容器，工作目录自动设为 /repo
```

或使用 CLI 快捷方式：

```bash
# 直接打开附加到指定容器的 VS Code 窗口
code --folder-uri "vscode-remote://attached-container+$(echo -n "${INSTANCE_ID}" | xxd -p)/repo"
```

### 5.2 问题理解流程（标准 SOP）

在 VS Code 集成终端（容器内）运行：

```bash
# Step 1: 确认当前 base commit（必须与数据集一致）
git log --oneline -3

# Step 2: 阅读问题描述（来自 task_list.json 的 problem_statement）
# 在宿主机查看：cat tasks/task_list.json | jq '.[] | select(.instance_id=="<id>") | .problem_statement'

# Step 3: 先运行 FAIL_TO_PASS 测试，确认初始失败状态
python -m pytest <fail_to_pass_tests> -x -v 2>&1 | tail -30

# Step 4: 运行 PASS_TO_PASS 测试，记录基线
python -m pytest <pass_to_pass_tests> -v --tb=no -q 2>&1 | tail -20
```

### 5.3 模型辅助修复（IDE 内置模型调用方式）

根据你的 IDE 内置模型部署方式，选择以下接入方案之一：

**方案 A：模型服务部署在宿主机（推荐）**

```bash
# 宿主机启动模型推理服务
python -m your_model_server --port 8080 --model /path/to/model

# 容器内访问（Docker Desktop 特殊 DNS）
curl http://host.docker.internal:8080/v1/complete \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Fix the following issue in Python: ..."}'

# IDE 插件配置：将 endpoint 设为 http://host.docker.internal:8080
```

**方案 B：模型服务部署在容器内**

```bash
# 启动容器时挂载模型文件
docker run -d \
    --name "${INSTANCE_ID}" \
    --gpus all \
    -v /host/models:/models:ro \
    --workdir /repo \
    "${IMAGE}" \
    /bin/bash -c "
        # 在容器内启动模型服务
        python /models/serve.py --port 8080 &
        # 保持容器存活
        tail -f /dev/null
    "
```

**方案 C：通过 docker exec 批量推理（无头模式）**

```bash
# 将问题描述传入容器，在容器内调用模型生成 patch
docker exec "${INSTANCE_ID}" bash -c "
    python /path/to/model_cli.py \
        --problem '$(cat tasks/problem_${INSTANCE_ID}.txt)' \
        --repo-path /repo \
        --output /patches/${INSTANCE_ID}.patch
"
```

### 5.4 代码修复与验证循环

```
修改代码
    │
    ▼
在容器内运行 FAIL_TO_PASS 测试
    │
    ├─► 全部通过？─── 是 ──► 运行 PASS_TO_PASS 测试
    │                                   │
    │                    ├─ 无新增失败？─► 导出 patch ✓
    │                    └─ 有回归？────► 回溯修改
    │
    └─► 未通过 ──────────────────────────► 继续修改
```

**容器内验证命令模板**：

```bash
# 精确运行 FAIL_TO_PASS 测试（以 pytest 为例）
python -m pytest \
    "tests/test_feature.py::TestClass::test_method" \
    -x --tb=short -v

# 验证 PASS_TO_PASS 无回归（并行加速）
python -m pytest \
    "tests/test_existing.py" \
    -n auto --tb=no -q

# 多语言环境（JavaScript / TypeScript）
npm test -- --testPathPattern="affected_test"

# Java (Maven)
mvn test -Dtest="AffectedTest" -pl module-name
```

### 5.5 导出 patch（容器内操作）

```bash
# 在容器内生成 unified diff
cd /repo
git diff HEAD > /tmp/${INSTANCE_ID}.patch

# 验证 patch 格式
git apply --check /tmp/${INSTANCE_ID}.patch
echo "Patch valid: $?"

# 将 patch 复制到宿主机
# （在宿主机执行）
docker cp "${INSTANCE_ID}:/tmp/${INSTANCE_ID}.patch" "./patches/${INSTANCE_ID}.patch"

# 查看 patch 摘要
cat ./patches/${INSTANCE_ID}.patch | head -50
```

---

## 6. 阶段 4：收集补丁并生成预测文件

### 6.1 批量收集 patch 并生成评估 JSON

**脚本 `tasks/04_gather_patches.py`**：

```python
#!/usr/bin/env python3
"""
将 patches/ 目录下的所有 .patch 文件打包为 swe_bench_pro_eval.py 所需的 JSON 格式。
"""
import json
import os
import glob
from pathlib import Path

PATCHES_DIR = "patches"
TASKS_FILE  = "tasks/task_list.json"
OUTPUT_FILE = "results/my_ide_patches.json"
PREFIX      = "my_ide_run1"         # 标识本次评估的名称

# 读取任务列表
with open(TASKS_FILE) as f:
    tasks = {t["instance_id"]: t for t in json.load(f)}

# 收集 patch
output = []
missing = []

for patch_file in sorted(glob.glob(f"{PATCHES_DIR}/*.patch")):
    instance_id = Path(patch_file).stem
    if instance_id not in tasks:
        print(f"WARNING: {instance_id} not in task list, skipping")
        continue

    with open(patch_file, "r") as f:
        patch_content = f.read()

    if not patch_content.strip():
        print(f"WARNING: Empty patch for {instance_id}")
        missing.append(instance_id)
        continue

    output.append({
        "instance_id": instance_id,
        "patch":       patch_content,
        "prefix":      PREFIX,
    })

# 对没有 patch 的实例填入空 patch（评估时会被标记为未解决）
for instance_id in tasks:
    if instance_id not in {o["instance_id"] for o in output}:
        missing.append(instance_id)
        output.append({
            "instance_id": instance_id,
            "patch":       "",            # 空 patch → 评估为未解决
            "prefix":      PREFIX,
        })

os.makedirs("results", exist_ok=True)
with open(OUTPUT_FILE, "w") as f:
    json.dump(output, f, indent=2)

print(f"\n✅ Collected {len(output)} patches → {OUTPUT_FILE}")
print(f"   Submitted: {len(output) - len(missing)}")
print(f"   Missing / empty: {len(missing)}")
if missing:
    print(f"   Missing instances: {missing[:5]}{'...' if len(missing)>5 else ''}")
```

运行：

```bash
python tasks/04_gather_patches.py
```

### 6.2 预测文件格式说明

生成的 `results/my_ide_patches.json` 结构如下：

```json
[
  {
    "instance_id": "instance_python_django_4a8f2c1",
    "patch": "diff --git a/django/core/handlers/base.py b/django/core/handlers/base.py\nindex 3a2f1b8..9c4d7e2 100644\n--- a/django/core/handlers/base.py\n+++ b/django/core/handlers/base.py\n@@ -45,6 +45,8 @@ class BaseHandler:\n     def load_middleware(self):\n         self._middleware_chain = handler\n+        if not self._middleware_chain:\n+            raise ImproperlyConfigured('...')\n",
    "prefix": "my_ide_run1"
  },
  {
    "instance_id": "instance_typescript_nodebb_7b3c9f0",
    "patch": "diff --git a/src/user.ts b/src/user.ts\n...",
    "prefix": "my_ide_run1"
  }
]
```

> **关键约束**：
> - `patch` 字段必须是标准 `unified diff` 格式（`git diff` 或 `diff -u` 输出）
> - `instance_id` 必须与数据集中完全一致
> - 未能生成 patch 的实例用空字符串 `""` 占位，评估时自动标记为未解决

---

## 7. 阶段 5：运行官方评估

### 7.1 第一步：验证金标准补丁（强烈推荐）

在评估自己的 patch 前，先用官方金标准（gold patch）验证评估环境是否正常工作。这是排查环境问题的基准。

**提取金标准 patch**：

```python
# 脚本: tasks/00_extract_gold_patches.py
import json
from datasets import load_dataset

dataset = load_dataset("ScaleAI/SWE-bench_Pro", split="test")

gold_patches = []
for row in dataset:
    gold_patches.append({
        "instance_id": row["instance_id"],
        "patch":       row["patch"],        # 官方金标准补丁
        "prefix":      "gold",
    })

with open("results/gold_patches.json", "w") as f:
    json.dump(gold_patches, f, indent=2)

print(f"Extracted {len(gold_patches)} gold patches")
```

**运行金标准评估（本地 Docker）**：

```bash
cd SWE-bench_Pro-os

python swe_bench_pro_eval.py \
    --raw_sample_path swe_bench_pro_full.csv \
    --patch_path ../sweap-eval/results/gold_patches.json \
    --output_dir ../sweap-eval/results/gold_eval \
    --scripts_dir run_scripts \
    --num_workers 4 \
    --use_local_docker \
    --dockerhub_username jefzda

# 预期结果：resolved_rate 接近 100%（部分实例可能因环境差异略低）
```

**运行金标准评估（Modal 云端，推荐用于大规模）**：

```bash
python swe_bench_pro_eval.py \
    --raw_sample_path swe_bench_pro_full.csv \
    --patch_path ../sweap-eval/results/gold_patches.json \
    --output_dir ../sweap-eval/results/gold_eval \
    --scripts_dir run_scripts \
    --num_workers 50 \          # Modal 可并发更多 worker
    --dockerhub_username jefzda
```

### 7.2 第二步：评估自研 IDE 的 patch

```bash
python swe_bench_pro_eval.py \
    --raw_sample_path swe_bench_pro_full.csv \
    --patch_path ../sweap-eval/results/my_ide_patches.json \
    --output_dir ../sweap-eval/results/my_ide_eval \
    --scripts_dir run_scripts \
    --num_workers 4 \
    --use_local_docker \
    --dockerhub_username jefzda
```

### 7.3 结果解读

评估完成后，`results/my_ide_eval/` 目录包含：

```
my_ide_eval/
├── results.json          ← 主结果文件（每实例的 resolved 状态）
├── summary.json          ← 汇总指标（resolved_rate 等）
└── logs/
    ├── <instance_id>.log ← 每个实例的评估详细日志
    └── ...
```

**解析结果脚本 `tasks/05_analyze_results.py`**：

```python
#!/usr/bin/env python3
"""解析评估结果，输出详细报告"""
import json
from pathlib import Path

RESULTS_FILE = "results/my_ide_eval/results.json"
GOLD_FILE    = "results/gold_eval/results.json"
TASKS_FILE   = "tasks/task_list.json"

with open(RESULTS_FILE)  as f: results = json.load(f)
with open(GOLD_FILE)     as f: gold    = json.load(f)
with open(TASKS_FILE)    as f: tasks   = {t["instance_id"]: t for t in json.load(f)}

gold_resolved  = {k for k, v in gold.items()    if v.get("resolved")}
my_resolved    = {k for k, v in results.items() if v.get("resolved")}

total    = len(results)
resolved = len(my_resolved)

print("=" * 60)
print(f"  评估报告 — 自研 IDE 补丁")
print("=" * 60)
print(f"  总实例数:          {total}")
print(f"  已解决:            {resolved} ({resolved/total*100:.1f}%)")
print(f"  金标准上界:        {len(gold_resolved)} ({len(gold_resolved)/total*100:.1f}%)")
print(f"  相对金标准达成率:  {resolved/max(len(gold_resolved),1)*100:.1f}%")

print("\n  按语言分布:")
by_lang = {}
for iid, task in tasks.items():
    lang = task.get("repo_language", "Unknown")
    is_res = iid in my_resolved
    by_lang.setdefault(lang, {"total": 0, "resolved": 0})
    by_lang[lang]["total"] += 1
    if is_res:
        by_lang[lang]["resolved"] += 1

for lang, stat in sorted(by_lang.items()):
    r = stat["resolved"]
    t = stat["total"]
    print(f"    {lang:20s}: {r}/{t} ({r/t*100:.0f}%)")

print("\n  未解决但金标准可解决（IDE 提升空间）:")
improvable = gold_resolved - my_resolved
print(f"    {len(improvable)} 个实例")
for iid in list(improvable)[:5]:
    print(f"    - {iid}")
if len(improvable) > 5:
    print(f"    ... 及其他 {len(improvable)-5} 个")
```

### 7.4 关键评估指标说明

| 指标 | 定义 | 计算方式 |
|------|------|---------|
| **Resolved Rate** | 核心指标：成功解决的实例比例 | `已解决数 / 总实例数 × 100%` |
| **FAIL_TO_PASS** | 原本失败、打 patch 后通过的测试 | 评估脚本自动验证，必须全部通过才算 resolved |
| **PASS_TO_PASS** | 原本通过、打 patch 后仍通过的测试 | 有任何一个回归，该实例标记为未解决 |
| **相对达成率** | 与金标准的差距 | `IDE resolved / Gold resolved × 100%` |

---

## 8. 可重复性与质量管控

### 8.1 实例操作记录（Audit Log）

每个实例的修复过程必须记录，以确保可审计和可复现：

```bash
# 操作日志结构（针对每个 instance）
logs/
└── <instance_id>/
    ├── 00_problem_statement.txt   # 问题描述原文
    ├── 01_fail_tests_before.log   # 修复前的测试结果
    ├── 02_model_interactions.jsonl # 模型调用记录（输入/输出）
    ├── 03_pass_tests_after.log    # 修复后的测试结果
    └── 04_patch.diff             # 最终 patch
```

**记录模型调用的 wrapper（示例）**：

```python
# model_wrapper.py
import json, datetime

def call_model_with_logging(prompt: str, instance_id: str, model_fn) -> str:
    """调用 IDE 内置模型并记录完整交互"""
    start = datetime.datetime.utcnow().isoformat()
    response = model_fn(prompt)
    
    record = {
        "timestamp": start,
        "instance_id": instance_id,
        "prompt_length": len(prompt),
        "response_length": len(response),
        # 不记录完整 prompt/response 以节省空间
        # 按需改为记录完整内容
    }
    
    log_path = f"logs/{instance_id}/02_model_interactions.jsonl"
    with open(log_path, "a") as f:
        f.write(json.dumps(record) + "\n")
    
    return response
```

### 8.2 环境一致性检查清单

在每个实例修复开始前，执行：

```bash
# 检查脚本: tasks/check_env.sh
#!/usr/bin/env bash
INSTANCE_ID="$1"

echo "=== 环境一致性检查: ${INSTANCE_ID} ==="

# 1. 确认容器使用正确镜像
docker inspect "${INSTANCE_ID}" | grep '"Image":'

# 2. 确认 git HEAD 为正确的 base_commit
docker exec "${INSTANCE_ID}" git -C /repo log --oneline -1

# 3. 确认没有未提交的更改（修复前应为干净状态）
DIRTY=$(docker exec "${INSTANCE_ID}" git -C /repo status --porcelain)
if [ -z "$DIRTY" ]; then
    echo "✅ Working tree clean"
else
    echo "❌ Working tree dirty! Files modified:"
    echo "$DIRTY"
    exit 1
fi

echo "=== 检查通过 ==="
```

### 8.3 批量自动化执行框架

对于需要大规模批量处理的情况（模型推理自动生成 patch 后由 IDE 辅助验证），可使用以下并行框架：

```python
# tasks/batch_runner.py
import json
import subprocess
import concurrent.futures
from pathlib import Path

MAX_PARALLEL = 2   # 并发容器数，根据宿主机资源调整

def process_instance(task: dict) -> dict:
    iid   = task["instance_id"]
    tag   = task["dockerhub_tag"]
    image = f"jefzda/sweap-images:{tag}"
    
    # 1. 启动容器
    subprocess.run([
        "docker", "run", "-d", "--name", iid,
        "--memory=8g", "--cpus=4",
        "--workdir", "/repo",
        image, "/bin/bash", "-c", "tail -f /dev/null"
    ], check=True)
    
    try:
        # 2. 执行模型推理（示例：调用宿主机模型服务）
        result = subprocess.run([
            "docker", "exec", iid, "bash", "-c",
            f"curl -s http://host.docker.internal:8080/fix "
            f"--data-urlencode 'problem={task[\"problem_statement\"]}' "
            f"| git apply -"
        ], capture_output=True, text=True, timeout=600)
        
        # 3. 导出 patch
        patch_result = subprocess.run(
            ["docker", "exec", iid, "git", "-C", "/repo", "diff", "HEAD"],
            capture_output=True, text=True
        )
        
        patch_path = Path(f"patches/{iid}.patch")
        patch_path.write_text(patch_result.stdout)
        
        return {"instance_id": iid, "status": "ok", "patch_size": len(patch_result.stdout)}
    
    except Exception as e:
        return {"instance_id": iid, "status": "error", "error": str(e)}
    
    finally:
        # 4. 清理容器
        subprocess.run(["docker", "rm", "-f", iid], capture_output=True)


with open("tasks/task_list.json") as f:
    tasks = json.load(f)

with concurrent.futures.ThreadPoolExecutor(max_workers=MAX_PARALLEL) as executor:
    futures = {executor.submit(process_instance, t): t for t in tasks}
    for future in concurrent.futures.as_completed(futures):
        result = future.result()
        print(f"  {result['instance_id']}: {result['status']}")
```

---

## 9. 常见问题与排查

### 9.1 Docker 相关

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 镜像拉取失败 | 网络问题 / DockerHub 限速 | 配置镜像加速器；使用 `docker login` 后拉取 |
| 容器内存 OOM | 内存分配不足 | 调大 `--memory` 参数至 `12g`，或 Docker Desktop 整体分配增至 16GB |
| ARM 架构兼容问题 | M 系列 Mac，镜像为 amd64 | 启用 Docker Desktop Rosetta 仿真；添加 `--platform linux/amd64` 参数 |
| 磁盘空间不足 | 镜像累积 | 定期运行 `docker image prune -f`；使用 `docker system df` 检查 |
| VS Code 连接超时 | 容器启动慢 | 检查 `docker logs <instance_id>`；容器内 Node.js 版本须 ≥ 16 |

### 9.2 patch 格式问题

```bash
# 验证 patch 可以正确应用
git apply --check patches/<instance_id>.patch

# 检查编码问题（必须为 UTF-8）
file patches/<instance_id>.patch

# 查看 patch 是否包含二进制文件（应只有文本变更）
grep "^Binary files" patches/<instance_id>.patch
```

### 9.3 评估脚本超时

```bash
# 调整单实例评估超时（默认 300 秒）
python swe_bench_pro_eval.py \
    --patch_path results/my_ide_patches.json \
    --timeout 600 \        # 复杂实例给更多时间
    --num_workers 2        # 减少并发避免资源竞争
```

### 9.4 PASS_TO_PASS 回归诊断

```bash
# 在容器内精确对比 patch 前后的测试结果
docker exec "${INSTANCE_ID}" bash -c "
    # 记录 patch 前的通过测试
    git stash
    python -m pytest tests/ -v --tb=no -q 2>&1 > /tmp/before.txt
    
    # 应用 patch
    git stash pop
    
    # 记录 patch 后的通过测试
    python -m pytest tests/ -v --tb=no -q 2>&1 > /tmp/after.txt
    
    # 对比差异（找出新增失败）
    diff /tmp/before.txt /tmp/after.txt | grep '^>' | grep FAILED
"
```

---

## 10. 附录：脚本与模板

### 10.1 脚本目录总览

```
sweap-eval/
├── tasks/
│   ├── 00_extract_gold_patches.py  # 提取金标准补丁
│   ├── 01_export_tasks.py          # 导出任务清单
│   ├── 02_pull_images.sh           # 批量拉取 Docker 镜像（自动生成）
│   ├── 03_start_instance.sh        # 启动单个容器
│   ├── 04_gather_patches.py        # 收集 patch → JSON
│   ├── 05_analyze_results.py       # 结果分析报告
│   ├── batch_runner.py             # 批量自动化框架
│   └── check_env.sh                # 环境一致性检查
├── patches/
│   └── <instance_id>.patch         # 每实例的 patch 文件
├── results/
│   ├── gold_patches.json           # 金标准 patch（验证用）
│   ├── my_ide_patches.json         # 自研 IDE 生成的 patch
│   ├── gold_eval/                  # 金标准评估结果
│   └── my_ide_eval/                # IDE 评估结果
└── logs/
    ├── pull_images.log
    └── <instance_id>/              # 每实例操作日志
```

### 10.2 完整评估检查清单

```
环境准备阶段：
  [ ] Python venv 创建，依赖安装完成
  [ ] Docker Desktop 运行，内存 ≥ 16GB，磁盘 ≥ 120GB
  [ ] VS Code Dev Containers 插件已安装
  [ ] HuggingFace 数据集可访问（或离线缓存已就绪）

任务准备阶段：
  [ ] task_list.json 导出完成，instance 数量正确
  [ ] Docker 镜像预拉取完成（验证 docker images 数量）
  [ ] 金标准 patch 提取完成（gold_patches.json）

评估前验证：
  [ ] 金标准评估通过（resolved_rate 接近预期上界）
  [ ] 至少 1 个实例手动端到端验证通过（修复→patch→评估）

批量修复阶段：
  [ ] 每个实例有完整操作日志（日志目录完整）
  [ ] 所有 patch 文件格式验证通过（git apply --check）
  [ ] patch 收集完成，缺失实例用空 patch 占位

最终评估：
  [ ] swe_bench_pro_eval.py 运行完成，无报错
  [ ] results.json 生成，字段完整
  [ ] 结果分析报告输出，按语言分布统计完成
  [ ] 与金标准对比，记录相对达成率
```

### 10.3 参考资料

| 资源 | 链接 |
|------|------|
| SWE-bench_Pro GitHub | https://github.com/scaleapi/SWE-bench_Pro-os |
| 数据集（HuggingFace） | https://huggingface.co/datasets/ScaleAI/SWE-bench_Pro |
| Docker Hub 镜像 | https://hub.docker.com/r/jefzda/sweap-images |
| 公开排行榜 | https://scale.com/leaderboard/swe_bench_pro_public |
| SWE-bench 官方文档 | https://www.swebench.com/SWE-bench/ |
| VS Code Dev Containers | https://code.visualstudio.com/docs/devcontainers/containers |

---

*文档维护建议：每次评估运行后，将 `summary.json` 中的关键数据（总实例数、resolved rate、耗时）记录至版本控制，用于追踪模型迭代进展。*
