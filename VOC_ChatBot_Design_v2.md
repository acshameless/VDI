# VOC 智能客服 ChatBot 系统设计文档 v2

> **基于 tape.systems + bub.build + Qwen 3.5 9B + Confluence MCP/RAG**
> 技术栈：Python 3 | 测试环境：Docker Compose

---

## ⚠️ 架构前提澄清

```
┌─────────────────────────────────────────────────────────┐
│          已存在的业务系统（被监控方，只读查询）              │
│   AWS EKS  /  CloudWatch  /  RDS  /  DynamoDB  /  Redis │
│   ↑ Bot 从这里读取系统状态、告警、业务上下文，用于增强答复  │
└─────────────────────────────────────────────────────────┘
                         ↑ boto3 只读查询
┌─────────────────────────────────────────────────────────┐
│            VOC ChatBot 系统（新建，独立部署）              │
│   bub Agent + Qwen 3.5 9B + tape Store + 知识接入层      │
│   部署：本地 / Docker / 任意服务器（不依赖业务 AWS 账户）   │
└─────────────────────────────────────────────────────────┘
```

**业务系统 → Bot 的读取关系：**

| 业务系统组件 | Bot 读取内容 | 接入方式 |
|---|---|---|
| CloudWatch | 当前活跃告警、最近事件 | boto3 `describe_alarms` |
| RDS | 连接数、慢查询统计（只读） | boto3 RDS describe_* |
| EKS | Pod 状态、最近 Events | boto3 EKS / kubectl |
| DynamoDB | 表级别吞吐量告警 | boto3 CloudWatch Metrics |
| Redis (ElastiCache) | 内存使用率、连接数告警 | boto3 CloudWatch Metrics |

---

## 目录

1. [Confluence 接入方案对比（最优解分析）](#1-confluence-接入方案对比)
2. [更新后的系统架构](#2-更新后的系统架构)
3. [用户故事（保持不变）](#3-用户故事)
4. [Confluence Docker 测试环境](#4-confluence-docker-测试环境)
5. [MCP 接入方案详细实现](#5-mcp-接入方案详细实现)
6. [混合策略：MCP + RAG 最优组合](#6-混合策略)
7. [GSD 需求目标](#7-gsd)
8. [EARS 需求规格](#8-ears)
9. [完整 Docker Compose 测试环境](#9-完整-docker-compose-测试环境)

---

## 1. Confluence 接入方案对比

### 方案总览

| 方案 | 实时性 | 精准度 | 基础设施复杂度 | 适合场景 |
|---|---|---|---|---|
| **A. MCP（推荐）** | ✅ 实时 | ✅ 高（CQL） | 🟢 极低 | 官方指南、精确查找 |
| **B. RAG（向量检索）** | ❌ 有延迟（需定期索引） | 🟡 中（语义近似） | 🔴 高（pgvector + embedding）| 模糊语义检索 |
| **C. 混合（最优解）** | ✅ 实时 | ✅ 高 | 🟡 中 | 生产推荐 |
| D. 全文检索（CQL only） | ✅ 实时 | 🟡 中（关键词） | 🟢 低 | 简单场景 |
| E. 预处理结构化提取 | ❌ 有延迟 | ✅ 高（结构化）| 🟡 中 | FAQ/SOP 类文档 |

---

### 方案 A：MCP 接入（首选，替代 RAG）

**核心优势**：LLM 直接调用工具实时读取 Confluence，不需要预索引、不需要向量数据库。

```
用户问题 → Qwen 3.5 9B 判断意图
         → 调用 Tool: confluence_search(cql="text~'EKS CrashLoop' AND space=OPS")
         → 返回实时页面内容
         → LLM 基于内容生成答复
```

**工具选型**：`mcp-atlassian`（开源，支持 Server/Data Center，MIT 协议）

```bash
pip install mcp-atlassian
# 支持 Confluence Server 用 Personal Access Token
```

**MCP 提供的核心 Confluence 工具**：

```python
# mcp-atlassian 暴露的工具（LLM 可直接调用）
confluence_search(cql, limit)          # CQL 语义 + 全文检索
confluence_get_page(page_id)           # 获取完整页面 Markdown
confluence_get_page_children(page_id)  # 遍历文档树
confluence_list_spaces()               # 列出所有 Space
confluence_create_page(...)            # 写回（知识沉淀时使用）
confluence_update_page(...)            # 更新（resolved 后沉淀）
```

**CQL（Confluence Query Language）示例**：

```sql
-- 精确 Space 限定
text ~ "DynamoDB 超时" AND space = "OPS" AND type = page

-- 按标签检索
label = "troubleshooting" AND space in ("OPS", "DEV")

-- 最近修改的相关文档
text ~ "Redis 连接数" AND lastModified >= "2026-01-01"

-- 按 ancestor 查子树
ancestor = "故障排查" AND text ~ "EKS"
```

**MCP vs RAG 优劣对比**：

| 维度 | MCP | RAG (pgvector) |
|---|---|---|
| Confluence 内容时效 | 始终最新 | 依赖索引周期（可能滞后 24h） |
| 基础设施依赖 | 仅 Confluence REST API | pgvector + embedding 服务 + 定时爬取任务 |
| 检索精度 | CQL 精确匹配，LLM 选择页面 | 语义近似，可能引入不相关内容 |
| 处理长文档 | 需 LLM 上下文窗口承载 | Chunking 后只注入相关 chunk |
| 成本 | 每次查询 1 次 API 调用 | 离线预计算 + 在线向量检索 |
| 适合 Qwen 9B | ✅（工具调用能力足够） | ✅（prompt 注入无需 tool call） |

**关键结论**：对于 Confluence 官方指南，**MCP 是最优解**，完全可以替代 RAG。

---

### 方案 B：RAG 仍然必要的场景

RAG 不应完全放弃，因为有些数据不在 Confluence 中：

| 数据源 | 使用 MCP？ | 使用 RAG？ | 原因 |
|---|---|---|---|
| Confluence 官方指南 | ✅ MCP | ❌ | 实时准确，无需预索引 |
| 历史 VOC 记录（RDS）| ❌ | ✅ RAG | VOC 是非结构化文本，数量大，需语义检索 |
| 故障告警历史 | ❌ | ✅ RAG | 时序数据 + 语义关联分析 |
| 研发 Slack/钉钉历史 | ❌ | ✅ RAG（可选）| 未来扩展 |

**最终结论**：

```
Confluence → MCP（实时，无向量库）
VOC 历史  → RAG（pgvector，语义检索）
```

---

### 方案 C：混合策略（生产推荐）

```python
class HybridRetriever:
    """
    1. MCP 查 Confluence（精确、实时）
    2. pgvector 查历史 VOC（语义）
    3. 合并注入 prompt
    """
    async def retrieve(self, query: str, session_state: dict):
        # 并行执行
        confluence_docs, voc_cases = await asyncio.gather(
            self.mcp_client.search_confluence(query),
            self.vector_store.search_voc(query, top_k=3)
        )
        return self.merge_context(confluence_docs, voc_cases)
```

---

## 2. 更新后的系统架构

```
┌────────────────────────────────────────────────────────────────────┐
│                      用户接入层（Channel）                            │
│   Telegram Bot  │  Web Chat UI  │  CLI（开发测试）                   │
└──────────────────────────┬─────────────────────────────────────────┘
                           │ bub ChannelManager
┌──────────────────────────▼─────────────────────────────────────────┐
│                   bub.build Agent（Hook Pipeline）                   │
│  resolve_session → load_state → build_prompt → run_model           │
│  [SessionPlugin]   [TapePlugin]  [Hybrid      [QwenPlugin]         │
│                    [AWSPlugin]    Retriever]                        │
│                         ↓                                          │
│                  save_state → render_outbound → dispatch_outbound  │
│                  [TapePlugin]               [EscalationPlugin]     │
└──────────┬─────────────────────┬─────────────────────┬────────────┘
           │                     │                     │
   ┌───────▼──────┐    ┌─────────▼──────┐    ┌────────▼──────────┐
   │  tape Store  │    │  知识接入层      │    │  升级服务          │
   │(tape.systems)│    │                │    │                   │
   │ User Tape    │    │ ┌────────────┐ │    │ on-call 管理       │
   │ Shared Tape  │    │ │MCP Client  │ │    │ tape handoff       │
   │ System Tape  │    │ │(Confluence)│ │    │ Telegram 通知      │
   └──────────────┘    │ └────────────┘ │    └───────────────────┘
                       │ ┌────────────┐ │
                       │ │pgvector RAG│ │
                       │ │(VOC 历史)  │ │
                       │ └────────────┘ │
                       └────────────────┘
                                │
              ┌─────────────────┼──────────────────┐
              │ 只读查询（boto3）                    │
┌─────────────▼──────────────────────────────────────▼──────────────┐
│              已存在的业务系统（只读，不修改）                          │
│   ┌─────────┐  ┌───────────┐  ┌──────┐  ┌──────────┐  ┌───────┐  │
│   │   EKS   │  │CloudWatch │  │ RDS  │  │DynamoDB  │  │ Redis │  │
│   │Pod状态  │  │告警/指标   │  │慢查询│  │吞吐量指标│  │连接数 │  │
│   └─────────┘  └───────────┘  └──────┘  └──────────┘  └───────┘  │
│                     ↑ 只读！Bot 不写入业务系统                       │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│                    本地 LLM（独立部署）                               │
│            Qwen 3.5 9B  via  Ollama（OpenAI-compatible）            │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│              Confluence（知识库，通过 MCP 接入）                      │
│    mcp-atlassian → Confluence Server REST API → 实时读取页面         │
│    写入：Resolved VOC 后，Bot 辅助生成草稿，研发审核后发布            │
└────────────────────────────────────────────────────────────────────┘
```

---

## 3. 用户故事

（与 v1 相同，补充业务系统查询场景）

**US-06（更新）**：作为用户，我希望 Bot 能主动感知业务系统当前状态——如 EKS Pod 是否异常、RDS 是否有慢查询告警——并在答复中说明，而**不是**让我去 CloudWatch 自己查。

**US-13（新增）**：作为用户，当我询问"DynamoDB 写入延迟高"时，Bot 应能实时查询 Confluence 中最新的故障排查指南（而非基于可能已过期的缓存内容），并同时查询当前业务系统的 DynamoDB CloudWatch 指标，综合给出答复。

---

## 4. Confluence Docker 测试环境

### 4.1 快速启动（开发测试用）

> **注意**：`atlassian/confluence` 镜像首次启动需要申请免费 30 天试用 License（在 http://localhost:8090 界面引导申请）。

```yaml
# docker-compose.confluence-test.yml
version: "3.9"
services:
  confluence-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: confluencedb
      POSTGRES_USER: confluenceuser
      POSTGRES_PASSWORD: confluencepass
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --lc-collate=C --lc-ctype=C"
    volumes:
      - confluence_pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "confluenceuser", "-d", "confluencedb"]
      interval: 10s
      timeout: 5s
      retries: 5

  confluence:
    image: atlassian/confluence:9.2          # 最新稳定版
    depends_on:
      confluence-db:
        condition: service_healthy
    ports:
      - "8090:8090"
      - "8091:8091"                          # Synchrony（协同编辑）
    environment:
      ATL_JDBC_URL: "jdbc:postgresql://confluence-db:5432/confluencedb"
      ATL_JDBC_USER: "confluenceuser"
      ATL_JDBC_PASSWORD: "confluencepass"
      ATL_DB_TYPE: "postgresql"
      ATL_DB_SCHEMA_NAME: "public"
      # JVM 内存（测试环境最低配置）
      JVM_MINIMUM_MEMORY: "1024m"
      JVM_MAXIMUM_MEMORY: "2048m"
      # 禁用 Synchrony（节省内存，如不需要协同编辑）
      # ATL_SYNCHRONY_DISABLED: "true"
    volumes:
      - confluence_data:/var/atlassian/application-data/confluence
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8090/status"]
      interval: 30s
      timeout: 10s
      retries: 10
      start_period: 120s

volumes:
  confluence_pgdata:
  confluence_data:
```

### 4.2 启动步骤

```bash
# 1. 启动 Confluence
docker compose -f docker-compose.confluence-test.yml up -d

# 2. 等待启动（约 2-3 分钟）
docker compose -f docker-compose.confluence-test.yml logs -f confluence

# 3. 浏览器打开 http://localhost:8090
#    → 选择"设置我的 Confluence"
#    → 选择"试用版"
#    → 在 my.atlassian.com 获取免费 30 天 Data Center 试用 License
#    → 配置数据库（自动检测，无需手动填写，因为 env 已传入）
#    → 创建管理员账号 admin/admin123
#    → 完成初始化

# 4. 创建 Personal Access Token（用于 mcp-atlassian 认证）
#    Profile → Personal Access Tokens → Create token
#    → 复制 token 写入 .env: CONFLUENCE_PERSONAL_TOKEN=xxx
```

### 4.3 预置测试数据

```bash
# 使用 Python 脚本预置测试文档
pip install atlassian-python-api

python scripts/seed_confluence.py
```

```python
# scripts/seed_confluence.py
from atlassian import Confluence

conf = Confluence(
    url="http://localhost:8090",
    username="admin",
    password="admin123"
)

# 创建测试 Space
conf.create_space("OPS", "运维手册")

# 预置测试文档
pages = [
    ("EKS Pod CrashLoopBackOff 排查指南", "## 症状\n...\n## 排查步骤\n1. kubectl describe pod\n..."),
    ("DynamoDB 写入延迟告警处理", "## 原因\n- Provisioned 吞吐量不足\n## 解决\n- 切换 On-demand 模式\n..."),
    ("RDS 连接数超阈值处理", "## 原因\n## 排查\n## 解决\n..."),
    ("Redis 内存使用率高处理", "## 清理策略\n## 扩容方案\n..."),
]
for title, body in pages:
    conf.create_page("OPS", title, body=f"<p>{body}</p>")

print("✅ 测试数据已预置")
```

---

## 5. MCP 接入方案详细实现

### 5.1 bub MCP Plugin（核心实现）

```python
# src/voc_bot/plugins/confluence_mcp_plugin.py
from __future__ import annotations
import subprocess, json, asyncio
from bub import hookimpl

class ConfluenceMCPPlugin:
    """
    将 mcp-atlassian 作为 bub 的 Confluence 检索工具。
    通过子进程启动 MCP server，LLM 通过 tool_use 调用。
    """

    def __init__(self, confluence_url: str, token: str):
        self.mcp_env = {
            "CONFLUENCE_URL": confluence_url,
            "CONFLUENCE_PERSONAL_TOKEN": token,   # Server/DC 使用 PAT
        }

    @hookimpl
    def system_prompt(self, prompt, state):
        return """你是专业的 AWS 系统支持助手。可用工具：
- confluence_search: 搜索官方文档（CQL语法）
- confluence_get_page: 获取完整页面内容
- voc_search: 检索历史已解决的相似 VOC
- aws_cloudwatch_alarms: 查询当前业务系统告警状态

回答策略：
1. 先调用 aws_cloudwatch_alarms 了解当前系统状态
2. 用 confluence_search 找到相关官方排查文档
3. 用 voc_search 找历史相似案例
4. 综合生成答案，明确标注每个来源
5. 如无把握，建议升级到研发人员"""

    @hookimpl
    async def build_prompt(self, message, session_id, state):
        """
        不在 build_prompt 预检索，而是让 Qwen 通过 tool_use 按需检索。
        这是 MCP 方案与 RAG 方案最核心的区别：
        RAG = 先检索再生成（retrieve-then-generate）
        MCP = 边推理边检索（reason-then-retrieve-then-generate）
        """
        tape_history = state.get("tape_entries", [])
        return [
            {"role": "system", "content": state.get("system_prompt", "")},
            *tape_history[-8:],
            {"role": "user", "content": message["content"]}
        ]
```

### 5.2 AWS 业务系统查询工具（只读）

```python
# src/voc_bot/tools/aws_context.py
import boto3
from bub import tool

@tool
async def aws_cloudwatch_alarms(service: str = None) -> dict:
    """
    查询业务系统当前 CloudWatch 告警状态（只读）。
    参数：service - 过滤服务名（eks/rds/dynamodb/redis/all）
    返回：当前 ALARM 状态的告警列表
    """
    client = boto3.client("cloudwatch", region_name="ap-northeast-1")

    # 只查 ALARM 状态（不查 OK/INSUFFICIENT_DATA）
    response = client.describe_alarms(StateValue="ALARM")

    alarms = []
    for alarm in response["MetricAlarms"]:
        # 按 service 过滤（通过 Namespace 判断）
        namespace = alarm.get("Namespace", "")
        if service and service != "all":
            if service.upper() not in namespace.upper():
                continue
        alarms.append({
            "name": alarm["AlarmName"],
            "reason": alarm["StateReason"],
            "namespace": namespace,
            "metric": alarm.get("MetricName"),
            "updated": alarm["StateUpdatedTimestamp"].isoformat()
        })

    return {
        "active_alarm_count": len(alarms),
        "alarms": alarms,
        "note": "以上为业务系统当前活跃告警，供参考分析"
    }

@tool
async def aws_eks_pod_status(namespace: str = "default") -> dict:
    """查询业务系统 EKS 中 Pod 状态（只读）"""
    # 通过 boto3 eks describe_cluster 获取 kubeconfig
    # 然后用 kubernetes 客户端查询 pod 状态
    # 返回：异常 pod 列表（NotRunning/CrashLoopBackOff）
    ...

@tool
async def aws_rds_metrics(db_identifier: str) -> dict:
    """查询业务 RDS 实例最近指标（连接数、慢查询等，只读）"""
    client = boto3.client("cloudwatch")
    # get_metric_statistics for DatabaseConnections, ReadLatency, WriteLatency
    ...
```

### 5.3 bub Tool 注册（与 MCP 并用）

```python
# src/voc_bot/plugins/tools_plugin.py
from bub import hookimpl
from . import aws_tools   # 注册 aws_cloudwatch_alarms 等 tool
from . import voc_tools   # 注册 voc_search tool（pgvector）

class ToolsPlugin:
    @hookimpl
    def system_prompt(self, prompt, state):
        return None  # 由 ConfluenceMCPPlugin 处理

# bub 中：Confluence 工具通过 mcp-atlassian 提供
# AWS/VOC 工具通过 @tool 装饰器注册
# Qwen 通过 tool_use 统一调用两类工具
```

---

## 6. 混合策略

### 6.1 检索决策树

```
用户问题
    │
    ├── 涉及系统当前状态？ → aws_cloudwatch_alarms（实时）
    │
    ├── 需要官方文档/指南？ → confluence_search via MCP（实时）
    │        └── 精确找到？ → confluence_get_page（完整内容）
    │              └── 未找到？ → 扩大 CQL 范围 / 换关键词
    │
    ├── 需要参考历史案例？ → voc_search via pgvector（语义）
    │
    └── 综合以上结果 → Qwen 3.5 9B 生成答复
```

### 6.2 VOC 历史的 RAG（pgvector，仅此处使用）

```python
# src/voc_bot/tools/voc_tools.py
from bub import tool
import asyncpg

@tool
async def voc_search(query: str, top_k: int = 3, category: str = None) -> list:
    """
    语义检索历史已解决的 VOC 案例。
    （Confluence 用 MCP，VOC 历史用这个 pgvector 检索）
    """
    embedding = await embed(query)  # bge-m3

    sql = """
    SELECT voc_id, category, summary, resolution, resolved_at,
           1 - (embedding <=> $1::vector) AS similarity
    FROM voc_embeddings
    WHERE 1 - (embedding <=> $1::vector) > 0.75
    """ + (f" AND category LIKE '{category}%'" if category else "") + """
    ORDER BY similarity DESC
    LIMIT $2
    """

    async with asyncpg.connect(DATABASE_URL) as conn:
        rows = await conn.fetch(sql, embedding, top_k)

    return [
        {
            "voc_id": r["voc_id"],
            "summary": r["summary"],
            "resolution": r["resolution"],
            "similarity": round(r["similarity"], 3),
        }
        for r in rows
    ]
```

---

## 7. GSD 需求目标

### Goal G1：实时知识响应

| 要素 | 描述 |
|---|---|
| **Goal** | Confluence 知识永远基于最新版本，无陈旧索引风险 |
| **Scenario** | 运维更新了 EKS 排查指南 → 用户立即能通过 Bot 获取最新内容 |
| **Success Metric** | Confluence 内容与 Bot 答复之间零延迟（MCP 实时查询） |

### Goal G2：业务系统状态感知

| 要素 | 描述 |
|---|---|
| **Goal** | Bot 答复能反映业务系统当前实时状态，不是凭空推测 |
| **Scenario** | RDS 告警触发时用户提交 VOC → Bot 主动声明当前告警状态并关联分析 |
| **Success Metric** | 涉及 AWS 告警的 VOC 中，告警上下文注入率 ≥ 90% |

### Goal G3：历史 VOC 知识复用

| 要素 | 描述 |
|---|---|
| **Goal** | 相似问题重复提交率降低 ≥ 40% |
| **Scenario** | 新 VOC 命中历史案例（相似度 ≥ 0.85）→ 推荐历史解决方案 |
| **Success Metric** | resolved VOC 库增长率、重复 VOC 投诉下降率 |

### Goal G4：研发高效介入

| 要素 | 描述 |
|---|---|
| **Goal** | 研发接管后 TTR < 30 分钟 |
| **Scenario** | tape handoff anchor 含完整上下文 → 研发无需用户重复描述 |
| **Scenario** | 研发 /resolve → 自动生成 Confluence 草稿（写回知识库）|
| **Success Metric** | 研发介入后平均解决时间 |

---

## 8. EARS 需求规格

### 8.1 Confluence MCP 接入需求

```
REQ-C01 [WHEN]
  WHEN Qwen 决定需要查询 Confluence 时，
  THE 系统 SHALL 通过 mcp-atlassian 客户端发送 CQL 查询，
  并在 5s 内返回实时页面内容。

REQ-C02 [WHEN]
  WHEN Confluence MCP 查询无结果时，
  THE 系统 SHALL 自动尝试扩展 CQL 关键词重试一次，
  若仍无结果则告知用户文档未找到并建议升级到研发。

REQ-C03 [WHEN]
  WHEN 研发执行 /resolve 并附上解决方案时，
  THE 系统 SHALL 调用 confluence_create_page 生成草稿，
  并发送草稿链接给研发审核，审核通过后正式发布。

REQ-C04 [WHERE]
  WHERE Qwen 引用 Confluence 页面内容时，
  THE 系统 SHALL 在答复中标注页面标题、URL 和最后修改时间。
```

### 8.2 业务系统查询需求（只读）

```
REQ-A01 [WHEN]
  WHEN 用户 VOC 内容涉及 AWS 业务系统组件（EKS/RDS/DynamoDB/Redis）时，
  THE 系统 SHALL 调用 aws_cloudwatch_alarms 查询对应服务当前告警，
  并将结果注入到 Qwen 的推理上下文中。

REQ-A02 [WHERE]
  WHERE 业务系统存在活跃 CloudWatch 告警时，
  THE 系统 SHALL 在 Bot 答复中主动声明告警存在，
  注明告警名称和触发时间，不得隐瞒。

REQ-A03 [WHEN]
  WHEN aws_cloudwatch_alarms 查询因权限或网络失败时，
  THE 系统 SHALL 降级跳过告警注入，继续生成答复，
  并在答复末尾说明"当前无法获取系统实时状态"。

REQ-A04 [WHERE]
  WHERE Bot 查询业务系统时，
  THE Bot SHALL 只执行只读操作（describe_*, get_metric_statistics），
  严禁对业务系统执行任何写入、修改、重启操作。
```

### 8.3 VOC RAG 需求

```
REQ-V01 [WHEN]
  WHEN 用户问题与历史 VOC 向量相似度 ≥ 0.85 时，
  THE 系统 SHALL 推荐最多 3 条历史案例（匿名化），
  展示问题摘要、解决方案和解决时间。

REQ-V02 [WHEN]
  WHEN VOC 被标记为 resolved 时，
  THE 系统 SHALL 在 24h 内将该 VOC 的 embedding 写入 pgvector，
  供后续相似检索使用。
```

### 8.4 tape 上下文需求（与 v1 相同，关键项）

```
REQ-T01 [WHEN]
  WHEN 用户发送消息时，
  THE 系统 SHALL append user_message entry 到 User Tape，
  并在 bot_response entry 中记录引用的 Confluence 页面 ID 和 VOC ID。

REQ-T02 [WHEN]
  WHEN 研发接管（/takeover）时，
  THE 系统 SHALL 将 tape 当前 anchor 之后的 entries 全部提供给研发，
  无需用户重复描述任何背景信息。
```

---

## 9. 完整 Docker Compose 测试环境

```yaml
# docker-compose.yml（BOT 系统完整测试环境）
version: "3.9"

services:
  # ── 知识库：Confluence ──────────────────────────────
  confluence-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: confluencedb
      POSTGRES_USER: confluenceuser
      POSTGRES_PASSWORD: confluencepass
    volumes:
      - confluence_pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "confluenceuser"]
      interval: 10s
      retries: 5

  confluence:
    image: atlassian/confluence:9.2
    depends_on:
      confluence-db:
        condition: service_healthy
    ports:
      - "8090:8090"
    environment:
      ATL_JDBC_URL: "jdbc:postgresql://confluence-db:5432/confluencedb"
      ATL_JDBC_USER: "confluenceuser"
      ATL_JDBC_PASSWORD: "confluencepass"
      ATL_DB_TYPE: "postgresql"
      JVM_MINIMUM_MEMORY: "1024m"
      JVM_MAXIMUM_MEMORY: "2048m"
    volumes:
      - confluence_data:/var/atlassian/application-data/confluence

  # ── VOC 向量库（历史 VOC RAG，非 Confluence）──────────
  vocbot-db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: vocbot
      POSTGRES_USER: vocbot
      POSTGRES_PASSWORD: vocbotpass
    volumes:
      - vocbot_pgdata:/var/lib/postgresql/data
      - ./scripts/init_pgvector.sql:/docker-entrypoint-initdb.d/init.sql

  # ── Session 缓存 ──────────────────────────────────
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  # ── LLM 推理：Qwen 3.5 9B ─────────────────────────
  ollama:
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ollama_models:/root/.ollama
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - capabilities: [gpu]   # 取消注释启用 GPU

  # ── Embedding 服务（仅用于 VOC 历史向量化）────────────
  embedding:
    image: ghcr.io/huggingface/text-embeddings-inference:cpu-1.5
    command: --model-id BAAI/bge-m3 --port 8080
    ports:
      - "8080:8080"

  # ── VOC Bot（bub）────────────────────────────────
  vocbot:
    build: .
    depends_on:
      - ollama
      - vocbot-db
      - redis
      - embedding
    environment:
      # LLM
      OLLAMA_BASE_URL: "http://ollama:11434"
      # 知识库（MCP 接入 Confluence）
      CONFLUENCE_URL: "http://confluence:8090"
      CONFLUENCE_PERSONAL_TOKEN: "${CONFLUENCE_PAT}"  # 从 .env 读取
      # VOC 向量库
      VOCBOT_DATABASE_URL: "postgresql://vocbot:vocbotpass@vocbot-db:5432/vocbot"
      EMBEDDING_URL: "http://embedding:8080"
      # Session
      REDIS_URL: "redis://redis:6379"
      # 业务系统 AWS（只读，使用只读 IAM 角色）
      AWS_REGION: "${AWS_REGION}"
      AWS_ACCESS_KEY_ID: "${AWS_READONLY_KEY}"
      AWS_SECRET_ACCESS_KEY: "${AWS_READONLY_SECRET}"
    volumes:
      - "./:/app"
      - "tape_store:/app/.bub"
    ports:
      - "8000:8000"   # Web Chat UI（可选）

volumes:
  confluence_pgdata:
  confluence_data:
  vocbot_pgdata:
  ollama_models:
  tape_store:
```

### 启动顺序

```bash
# 1. 配置环境变量
cp .env.example .env
# 编辑 .env，填写：
# CONFLUENCE_PAT=your-pat-token（Confluence 初始化完后生成）
# AWS_REGION=ap-northeast-1
# AWS_READONLY_KEY=AKIA...（只读 IAM 用户）
# AWS_READONLY_SECRET=...

# 2. 启动基础设施
docker compose up -d confluence-db confluence vocbot-db redis

# 3. 等待 Confluence 初始化（约 3 分钟）
# 浏览器访问 http://localhost:8090 完成设置，创建 PAT

# 4. 启动 LLM 和 Bot
docker compose up -d ollama embedding
docker compose exec ollama ollama pull qwen2.5:9b

# 5. 预置测试数据
docker compose run --rm vocbot python scripts/seed_confluence.py
docker compose run --rm vocbot python scripts/init_voc_embeddings.py

# 6. 启动 Bot
docker compose up -d vocbot

# 7. 测试
docker compose exec vocbot uv run bub chat
```

### .env.example

```bash
# Confluence（Docker 测试）
CONFLUENCE_URL=http://localhost:8090
CONFLUENCE_PERSONAL_TOKEN=your-token-here

# 业务系统 AWS（只读访问）
AWS_REGION=ap-northeast-1
AWS_READONLY_KEY=AKIAIOSFODNN7EXAMPLE
AWS_READONLY_SECRET=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# 可选：Telegram Bot
TELEGRAM_TOKEN=your-bot-token
```

### init_pgvector.sql（VOC 向量表）

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE voc_embeddings (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    voc_id      VARCHAR(128) UNIQUE NOT NULL,
    category    VARCHAR(64),
    summary     TEXT NOT NULL,
    resolution  TEXT NOT NULL,
    resolved_by VARCHAR(128),
    resolved_at TIMESTAMP,
    embedding   vector(1024),   -- bge-m3 维度
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE INDEX ON voc_embeddings USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

---

## 变更记录

| 版本 | 变更内容 |
|---|---|
| v1.0 | 初版设计（误将业务系统作为 Bot 部署目标） |
| v2.0 | 修正：AWS 业务系统为只读查询对象；Confluence 改为 MCP 优先方案；RAG 仅保留用于 VOC 历史；补充 Docker Confluence 测试环境 |

---

*文档版本：v2.0 | 日期：2026-03-25*
