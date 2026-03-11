# Agent 性能与成本优化：Token 经济学与工程实践

> 撰写日期：2026年3月 | 结合业界最新进展

## 一、Agent 为什么这么贵？

一个残酷的现实：**AI Agent 的 API 调用成本是简单聊天机器人的 3-10 倍。** 一次不受约束的 Agent 任务仅 API 费用就可达 $5-8。如果不做优化，未受控的部署在 6-9 个月内就会超出预算 2-4 倍。

### Agent 成本的四大乘数效应

```
传统 Chatbot               AI Agent
┌──────────┐              ┌──────────┐
│ 1x LLM   │              │ 规划      │ → 1-2x LLM 调用
│ 调用      │              │ 工具选择   │ → 1x LLM 调用
│          │              │ 工具执行   │ → 外部 API 成本
│          │              │ 结果验证   │ → 1x LLM 调用
│          │              │ 响应合成   │ → 1x LLM 调用
│          │              │ (可能循环) │ → 以上 × N
└──────────┘              └──────────┘
  单次调用                   3-10x 调用
```

**成本乘数详解：**

| 乘数因素 | 说明 | 影响程度 |
|---------|------|---------|
| 多步骤循环 | 一次任务可能需要 3-10 轮 LLM 调用 | 3-10x |
| 上下文膨胀 | 每轮对话累积 Token，呈二次增长 | 2-5x |
| 输出 Token 溢价 | 输出 Token 价格是输入的 3-8 倍 | 3-8x |
| 工具调用开销 | 工具 Schema 描述占用大量输入 Token | 1.5-3x |
| 重试与修复 | 工具调用失败后的重试循环 | 1-3x |

### 2026 年主要模型定价对比

| 模型层级 | 代表模型 | 输入价格 ($/M tokens) | 输出价格 ($/M tokens) | 输出/输入比 |
|---------|---------|---------------------|---------------------|-----------|
| Nano/Flash | GPT-4o-mini, Claude Haiku, Gemini Flash | $0.07-0.30 | $0.28-1.25 | 4:1 |
| Mid-tier | GPT-4o, Claude Sonnet | $2.50-3.00 | $10.00-15.00 | 4:1 |
| Frontier | o1, Claude Opus, Gemini Ultra | $10.00-15.00 | $40.00-60.00 | 4-8:1 |
| Reasoning | o3, DeepSeek-R1 | $15.00+ | $60.00+ | 4-8:1 |

**关键洞察：** Nano 层与 Frontier 层之间存在约 **190 倍** 的成本差距。这正是模型路由优化的巨大空间。

## 二、模型路由——最高 ROI 的优化手段

### 核心思路

不是所有任务都需要最强的模型。**将 90% 的请求路由到 Nano/Flash 模型，仅 10% 需要 Frontier 模型**，可以在保持 90-95% 质量的前提下节省约 86% 的成本。

### 路由策略对比

#### 策略 1：静态路由

```python
MODEL_ROUTING = {
    "intent_classification": "gpt-4o-mini",    # 简单分类
    "tool_selection": "gpt-4o-mini",            # 工具选择
    "data_extraction": "gpt-4o-mini",           # 结构化提取
    "complex_reasoning": "gpt-4o",              # 复杂推理
    "code_generation": "claude-sonnet-4",       # 代码生成
    "final_synthesis": "gpt-4o",                # 最终综合
}

def route_request(task_type: str) -> str:
    return MODEL_ROUTING.get(task_type, "gpt-4o-mini")
```

**优点**：简单、可预测、零延迟开销  
**缺点**：无法适应任务复杂度的动态变化

#### 策略 2：基于分类器的路由（推荐用于生产）

```python
from dataclasses import dataclass
from enum import Enum

class ModelTier(Enum):
    NANO = "gpt-4o-mini"
    MID = "gpt-4o"
    FRONTIER = "o1"

@dataclass
class RoutingDecision:
    tier: ModelTier
    confidence: float
    reason: str

class ModelRouter:
    """用一个轻量分类器判断每个请求所需的模型层级"""

    def __init__(self):
        self.classifier = load_routing_classifier()  # 小型分类模型

    def route(self, query: str, context: dict) -> RoutingDecision:
        features = self._extract_features(query, context)
        tier_scores = self.classifier.predict(features)

        if tier_scores["nano"] > 0.8:
            return RoutingDecision(ModelTier.NANO, tier_scores["nano"], "简单任务")
        elif tier_scores["mid"] > 0.6:
            return RoutingDecision(ModelTier.MID, tier_scores["mid"], "中等复杂度")
        else:
            return RoutingDecision(ModelTier.FRONTIER, tier_scores["frontier"], "需要深度推理")

    def _extract_features(self, query: str, context: dict) -> dict:
        return {
            "query_length": len(query),
            "has_code": self._contains_code(query),
            "reasoning_keywords": self._count_reasoning_words(query),
            "tool_count": len(context.get("available_tools", [])),
            "conversation_depth": context.get("turn_count", 0),
        }
```

**优点**：动态适应、质量与成本平衡  
**缺点**：需要训练数据和维护分类器

#### 策略 3：Agent 内部分步路由

在 Agent 的执行循环中，不同步骤使用不同模型：

| 步骤 | 模型 | 原因 |
|------|------|------|
| 意图识别 | Nano (Mini) | 简单分类 |
| 计划制定 | Mid (4o) | 需要推理 |
| 工具参数生成 | Nano (Mini) | 结构化输出 |
| 结果验证 | Nano (Mini) | 简单判断 |
| 最终响应合成 | Mid (4o) | 质量要求高 |
| 复杂多跳推理 | Frontier (o1) | 深度推理 |

### 路由效果量化

| 方案 | 平均成本/任务 | 质量保持率 | 成本节省 |
|------|-------------|-----------|---------|
| 全部 Frontier | $5.00 | 100% (基线) | 0% |
| 全部 Mid-tier | $1.50 | 95% | 70% |
| 静态路由 | $0.80 | 92% | 84% |
| 分类器路由 | $0.70 | 94% | 86% |
| 分步路由 | $0.60 | 93% | 88% |

## 三、缓存策略——低投入高回报

### Prompt 缓存（Provider-native Caching）

Prompt 缓存是指 LLM 提供商在服务端缓存预计算的 KV Attention 张量，对重复的 Prompt 前缀免去重复计算。

#### 各提供商实现对比

| 提供商 | 实现方式 | 触发条件 | 成本节省 | 延迟节省 |
|-------|---------|---------|---------|---------|
| Anthropic Claude | 显式标记 `cache_control` | 手动指定 + 自动缓存(2026) | 90% (输入Token) | 75-85% |
| OpenAI | 自动前缀缓存 | ≥1024 Token，128 Token 对齐 | 50% (输入Token) | 类似 |
| Google Gemini | Context Caching API | 手动创建缓存上下文 | 75% | 60-80% |

#### Anthropic 显式缓存示例

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "你是一个客户支持Agent...(很长的系统提示词)...",
            "cache_control": {"type": "ephemeral"}  # 标记缓存
        },
        {
            "type": "text",
            "text": TOOL_SCHEMAS_JSON,  # 工具Schema定义，通常很大
            "cache_control": {"type": "ephemeral"}
        },
    ],
    messages=[{"role": "user", "content": user_query}],
)

# 查看缓存命中情况
print(f"缓存写入 Token: {response.usage.cache_creation_input_tokens}")
print(f"缓存命中 Token: {response.usage.cache_read_input_tokens}")
# 缓存命中的 Token 仅收 $0.30/M，相比正常 $3.00/M 节省 90%
```

#### 最佳缓存场景

| 场景 | 可缓存内容 | 预期节省 |
|------|-----------|---------|
| Agent 系统提示词 | System Prompt（通常 2000-5000 Token） | 90% |
| 工具 Schema 定义 | 所有可用工具的 JSON Schema | 90% |
| RAG 固定文档集 | 知识库检索的背景文档 | 70-85% |
| 多轮对话历史 | 对话前 N 轮的固定部分 | 50-70% |

### 语义缓存（Semantic Caching）

对语义相似的请求复用之前的结果：

```python
import numpy as np
from typing import Optional

class SemanticCache:
    def __init__(self, embedding_model, similarity_threshold=0.95):
        self.embedding_model = embedding_model
        self.threshold = similarity_threshold
        self.cache = []  # [(embedding, query, response)]

    def get(self, query: str) -> Optional[str]:
        query_embedding = self.embedding_model.encode(query)
        for cached_emb, cached_query, cached_response in self.cache:
            similarity = np.dot(query_embedding, cached_emb)
            if similarity >= self.threshold:
                return cached_response
        return None

    def put(self, query: str, response: str):
        embedding = self.embedding_model.encode(query)
        self.cache.append((embedding, query, response))
```

**适用场景**：FAQ类查询、重复性工具调用、标准化数据查询  
**不适用**：个性化推理、实时数据查询、创意生成

## 四、上下文压缩——对抗 Token 膨胀

Agent 执行多轮后，上下文窗口迅速膨胀。未优化的 10 轮对话可能消耗 50 倍于单次调用的 Token。

### 动态系统指令检索（Instruction & Tool Retrieval, ITR）

这是 2026 年初的一项重要研究成果（arXiv: 2602.17046）：不是每一步都把所有工具和指令塞进上下文，而是**按需检索**相关的指令和工具。

```
传统方式：每步都加载全部工具 Schema
┌─────────────────────────────────────┐
│ System Prompt (2000 tokens)          │
│ Tool 1 Schema (500 tokens)           │
│ Tool 2 Schema (500 tokens)           │
│ ... (20 个工具)                      │
│ Tool 20 Schema (500 tokens)          │
│ 对话历史 (N tokens)                  │
│ ────────────────────────────────     │
│ 总计: 12000 + N tokens / 每一步      │
└─────────────────────────────────────┘

ITR 方式：按需加载相关工具
┌─────────────────────────────────────┐
│ System Prompt 核心 (500 tokens)      │
│ 相关指令片段 (300 tokens)            │
│ 相关 Tool Schema × 2 (1000 tokens)  │
│ 压缩后对话历史 (M tokens, M << N)   │
│ ────────────────────────────────     │
│ 总计: 1800 + M tokens / 每一步       │
└─────────────────────────────────────┘
```

**效果：**
- 每步上下文 Token 减少 **95%**
- 端到端任务成本降低 **70%**
- Agent 可运行的循环次数增加 **2-20 倍**

### 对话历史压缩

```python
class ConversationCompressor:
    """压缩对话历史，保留关键信息同时减少 Token"""

    def compress(self, messages: list[dict], max_tokens: int) -> list[dict]:
        if self._count_tokens(messages) <= max_tokens:
            return messages

        compressed = []
        compressed.append(messages[0])  # 保留系统提示

        # 对中间轮次进行摘要
        middle_messages = messages[1:-3]
        if middle_messages:
            summary = self._summarize(middle_messages)
            compressed.append({
                "role": "system",
                "content": f"[对话历史摘要] {summary}"
            })

        # 保留最近 3 轮完整对话
        compressed.extend(messages[-3:])
        return compressed

    def _summarize(self, messages: list[dict]) -> str:
        key_facts = []
        for msg in messages:
            if "tool_result" in str(msg.get("content", "")):
                key_facts.append(self._extract_key_result(msg))
            elif msg["role"] == "assistant" and self._is_decision_point(msg):
                key_facts.append(self._extract_decision(msg))
        return "; ".join(key_facts)
```

## 五、FinOps for Agents——成本即工程指标

2026 年，业界将成本视为与延迟、可靠性同等重要的**一等工程指标**。这催生了 Agent FinOps 的实践体系。

### Agent COGS（Cost of Goods Sold）全栈

```
Agent 成本全栈分解：

┌──────────────────────────────────────┐
│ 人工审核成本 (Human-in-the-Loop)     │ ← 高风险场景
├──────────────────────────────────────┤
│ 治理与可观测性 (Governance & Obs)    │ ← 日志、追踪、评估
├──────────────────────────────────────┤
│ 记忆与检索 (Memory & Retrieval)      │ ← 向量数据库、RAG
├──────────────────────────────────────┤
│ 编排运行时 (Orchestration Runtime)   │ ← 计算、容器
├──────────────────────────────────────┤
│ 工具与副作用 (Tools & Side Effects)  │ ← 外部API调用
├──────────────────────────────────────┤
│ █████ 模型推理 (Model Inference)████ │ ← 通常占 60-80%
└──────────────────────────────────────┘
```

### 关键 FinOps 实践

#### 1. 设置循环限制和工具调用上限

```python
class AgentBudgetGuard:
    """防止 Agent 陷入无限循环导致成本失控"""

    def __init__(self, max_loops=10, max_tool_calls=20, max_cost_usd=2.0):
        self.max_loops = max_loops
        self.max_tool_calls = max_tool_calls
        self.max_cost_usd = max_cost_usd
        self.current_loops = 0
        self.current_tool_calls = 0
        self.current_cost = 0.0

    def check_budget(self, step_cost: float) -> bool:
        self.current_cost += step_cost
        self.current_loops += 1

        if self.current_loops > self.max_loops:
            raise BudgetExceededError(f"循环次数超限: {self.current_loops}/{self.max_loops}")
        if self.current_cost > self.max_cost_usd:
            raise BudgetExceededError(f"成本超限: ${self.current_cost:.2f}/${self.max_cost_usd:.2f}")
        return True

    def record_tool_call(self):
        self.current_tool_calls += 1
        if self.current_tool_calls > self.max_tool_calls:
            raise BudgetExceededError(f"工具调用超限: {self.current_tool_calls}/{self.max_tool_calls}")
```

#### 2. 成本归因与追踪

```python
class CostTracker:
    """将 Token 成本归因到具体的功能、用户、业务线"""

    def record(self, request_id: str, metadata: dict, usage: dict):
        cost = self._calculate_cost(usage)
        self.metrics.emit({
            "request_id": request_id,
            "cost_usd": cost,
            "input_tokens": usage["input_tokens"],
            "output_tokens": usage["output_tokens"],
            "cached_tokens": usage.get("cached_tokens", 0),
            "model": metadata["model"],
            "feature": metadata["feature"],        # 归因到功能
            "user_tier": metadata["user_tier"],     # 归因到用户层级
            "business_unit": metadata["biz_unit"],  # 归因到业务线
            "agent_step": metadata["step_name"],    # 归因到Agent步骤
        })
```

#### 3. 异常成本告警

设置基于 Token 消耗的异常检测：
- 单次任务成本超过 P95 阈值 → 触发告警
- 小时级成本环比增长 >50% → 触发告警
- 特定 Agent 步骤的 Token 消耗突然翻倍 → 触发告警

## 六、综合优化效果

将上述技术组合使用的预期效果：

| 优化技术 | 单独效果 | 累计效果 |
|---------|---------|---------|
| 模型路由 | 40-86% 节省 | 40-86% |
| Prompt 缓存 | 40-90% 节省 (输入Token) | 55-92% |
| 上下文压缩 | 50-95% 节省 (每步Token) | 70-95% |
| 工具调用批处理 | 20-40% 节省 | 75-97% |
| 循环限制 | 防止极端情况 | 风险兜底 |

**业界实测数据：** 综合应用路由、缓存和批处理，生产系统可实现 **47-80% 的总成本降低**。

## 七、性能优化：延迟也很关键

成本之外，用户体验同样重要。Agent 的延迟优化策略：

### 1. 并行工具调用

```python
import asyncio

async def execute_tools_parallel(tool_calls: list[dict]) -> list[dict]:
    """并行执行独立的工具调用，而非串行"""
    tasks = [execute_single_tool(tc) for tc in tool_calls]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return [
        {"tool": tc["name"], "result": r if not isinstance(r, Exception) else str(r)}
        for tc, r in zip(tool_calls, results)
    ]

# 串行：Tool A (1s) → Tool B (2s) → Tool C (1s) = 4s
# 并行：max(Tool A, Tool B, Tool C) = 2s
```

### 2. 流式输出

对于最终响应，使用流式输出减少用户感知延迟（首 Token 时间）。

### 3. 预测性预取

在 Agent 规划阶段，预测可能需要的工具结果并提前获取：

```python
async def speculative_prefetch(plan: AgentPlan):
    """在规划完成后，立即预取高概率需要的工具结果"""
    likely_tools = [step.tool for step in plan.steps if step.confidence > 0.8]
    prefetch_tasks = [prefetch_tool_result(tool) for tool in likely_tools[:3]]
    await asyncio.gather(*prefetch_tasks)
```

## 八、总结：成本优化路线图

**Phase 1：基础（第 1-2 周）**
- 启用 Provider 原生 Prompt 缓存
- 设置循环和工具调用上限
- 建立成本监控看板

**Phase 2：路由（第 3-4 周）**
- 实现静态模型路由（按任务类型）
- 收集路由决策的质量数据
- 分析各步骤的实际模型需求

**Phase 3：精细化（第 5-8 周）**
- 实现分类器路由或分步路由
- 部署上下文压缩
- 实现语义缓存
- 设置成本归因和告警

**Phase 4：持续优化（长期）**
- A/B 测试不同优化策略
- 跟踪新模型发布，更新路由策略
- 持续分析成本-质量 Tradeoff
- FinOps 实践制度化

**核心理念：Agent 的成本优化不是一次性工程，而是持续的运营实践。在 Token 经济学时代，每一个 Token 都应该创造价值。**

---

## 参考资源

- [Zylos Research - AI Agent Cost Optimization: Token Economics](https://zylos.ai/research/2026-02-19-ai-agent-cost-optimization-token-economics)
- [Zylos Research - AI Agent Model Routing Strategies](https://zylos.ai/research/2026-03-02-ai-agent-model-routing)
- [Zylos Research - Prompt Caching Architecture Patterns](https://zylos.ai/research/2026-02-24-prompt-caching-ai-agents-architecture)
- [arXiv: 2602.17046 - Dynamic System Instructions and Tool Exposure](https://arxiv.org/abs/2602.17046)
- [InfoWorld - FinOps for Agents](https://www.infoworld.com/article/4138748/finops-for-agents-loop-limits-tool-call-caps-and-the-new-unit-economics-of-agentic-saas.html)
- [Mavik Labs - LLM Cost Optimization 2026](https://www.maviklabs.com/blog/llm-cost-optimization-2026)
