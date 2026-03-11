# Agent 可观测性与评估体系：从"黑箱"到"透明决策"

> 撰写日期：2026年3月 | 结合业界最新进展

## 一、为什么 Agent 需要专门的可观测性？

传统 APM（Application Performance Monitoring）关注的是"请求是否成功、延迟多高、错误率多少"。但 AI Agent 的本质区别在于：

| 特性 | 传统应用 | AI Agent |
|------|---------|---------|
| 执行路径 | 确定性，代码逻辑固定 | **非确定性**，同一输入可产生不同推理路径 |
| 操作粒度 | 单次请求-响应 | **复合操作**：多次 LLM 调用 + 工具执行 + 决策循环 |
| 失败模式 | HTTP 错误码、异常栈 | **语义失败**：HTTP 200 但输出幻觉、推理偏差 |
| 成本模型 | 固定计算资源 | **动态 Token 消耗**，成本是运行时变量 |
| 质量衡量 | 功能正确性 | **多维度**：准确性、忠实度、安全性、效率 |

这意味着：**Agent 的可观测性必须从"发生了什么"升级到"它决定了什么、为什么这样决定"。**

## 二、Agent 可观测性的五大支柱

业界（Maxim AI、Arize Phoenix、LangSmith 等平台）已形成共识，Agent 可观测性由五大支柱构成：

### 1. Traces（追踪）

Trace 是 Agent 可观测性的核心。每一次 Agent 执行都应生成一棵完整的 Span 树，覆盖：

```
Agent Session (Root Span)
├── Planning Step (Span)
│   └── LLM Call: GPT-4o (Child Span)
│       ├── Input Tokens: 2,340
│       ├── Output Tokens: 512  
│       └── Latency: 1.8s
├── Tool Execution: search_database (Span)
│   ├── Arguments: {"query": "user orders", "limit": 10}
│   ├── Result: [10 records]
│   └── Latency: 0.3s
├── Reasoning Step (Span)
│   └── LLM Call: Claude-3.5 (Child Span)
└── Response Generation (Span)
    └── LLM Call: GPT-4o-mini (Child Span)
```

**关键要点：** Trace 不仅要记录 I/O 边界，更要捕获完整的**决策图谱（Decision Graph）**。

### 2. Metrics（指标）

Agent 的核心指标分为四类：

- **性能指标**：端到端延迟、单步延迟、LLM 调用次数、工具调用次数
- **成本指标**：Token 消耗量（输入/输出分开统计）、每次任务成本、每用户成本
- **质量指标**：任务完成率、回答准确率、幻觉率
- **效率指标**：步骤效率（完成任务的实际步数 vs 最优步数）

### 3. Logs & Payloads（日志与载荷）

持久化保存每一轮的：
- 完整 System Prompt
- 用户输入与上下文
- LLM 原始输出（含推理过程）
- 工具调用的参数与返回值
- 最终交付的响应

这些原始数据是事后分析和回归测试的基础。

### 4. Online Evaluations（在线评估）

在生产环境中实时运行自动化评估器：

- **忠实度（Faithfulness）**：Agent 的回答是否忠于其检索到的事实？
- **安全性（Safety）**：输出是否包含有害内容？
- **PII 检测**：是否泄露了个人敏感信息？
- **工具正确性**：工具调用参数是否正确？

### 5. Human Review Loops（人工审核循环）

对于高风险场景（金融决策、医疗建议、法律意见），建立人工审核工作流：
- 采样审核：按比例抽样 Agent 输出进行人工评估
- 触发审核：当自动评估器标记异常时，升级到人工
- 反馈闭环：人工标注结果回流到评估数据集

## 三、OpenTelemetry GenAI 语义约定——行业标准化

2026 年初，**OpenTelemetry GenAI 语义约定** 正在成为 Agent 可观测性的行业标准。这是一个重要的里程碑。

### 核心概念

OpenTelemetry 的 Generative AI Observability SIG（特别兴趣小组）自 2024 年 4 月开始开发，定义了专门用于 AI Agent 的 Span 类型和属性：

**Agent Span 属性：**

```yaml
# Agent 创建 Span
gen_ai.operation.name: "create_agent"
gen_ai.provider.name: "openai"
gen_ai.agent.id: "agent-001"
gen_ai.agent.name: "customer_support"
gen_ai.agent.description: "处理客户工单的支持 Agent"

# LLM 调用 Span
gen_ai.operation.name: "chat"
gen_ai.system: "openai"
gen_ai.request.model: "gpt-4o"
gen_ai.usage.input_tokens: 2340
gen_ai.usage.output_tokens: 512
gen_ai.response.finish_reasons: ["stop"]
```

### 生态集成

主要 Agent 框架已经或正在集成 OpenTelemetry：

| 框架 | 集成状态 | 说明 |
|------|---------|------|
| LangChain / LangGraph | ✅ 已支持 | 通过 LangSmith 或直接 OTLP 导出 |
| CrewAI | ✅ 已支持 | 内置 OpenTelemetry 导出器 |
| AutoGen / AG2 | ✅ 已支持 | 原生 Span 发射 |
| OpenAI Agents SDK | 🔄 进行中 | 2026 年计划支持 |

主要可观测性平台（Datadog、Honeycomb、New Relic、Grafana）均已支持接收符合 GenAI 语义约定的遥测数据。

### 实践建议

```python
# 使用 OpenTelemetry 为 Agent 添加自定义 Span 的示例
import os
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

# 启用语义约定兼容性
os.environ["OTEL_SEMCONV_STABILITY_OPT_IN"] = "gen_ai"

provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer("my-agent-app")

def run_agent(user_query: str):
    with tracer.start_as_current_span(
        "agent_session",
        attributes={
            "gen_ai.operation.name": "create_agent",
            "gen_ai.agent.name": "research_assistant",
        }
    ) as session_span:

        with tracer.start_as_current_span(
            "llm_call_planning",
            attributes={
                "gen_ai.operation.name": "chat",
                "gen_ai.system": "openai",
                "gen_ai.request.model": "gpt-4o",
            }
        ) as llm_span:
            plan = call_llm_for_planning(user_query)
            llm_span.set_attribute("gen_ai.usage.input_tokens", plan.input_tokens)
            llm_span.set_attribute("gen_ai.usage.output_tokens", plan.output_tokens)

        with tracer.start_as_current_span(
            "tool_execution",
            attributes={
                "gen_ai.operation.name": "execute_tool",
                "tool.name": "web_search",
            }
        ) as tool_span:
            result = execute_tool(plan.tool_call)
            tool_span.set_attribute("tool.result.status", "success")

        return synthesize_response(plan, result)
```

## 四、Agent 评估的三层模型

DeepEval 等评估框架提出了 Agent 评估的三层模型，将评估粒度从粗到细分解：

### 第 1 层：推理层（Reasoning Layer）

评估 Agent 的"思考"质量：

| 指标 | 含义 | 衡量方式 |
|------|------|---------|
| Plan Quality | 生成的计划是否合理、完整 | 对比 Ground Truth 计划 |
| Plan Adherence | 实际执行是否遵循了计划 | 对比计划 vs 实际步骤 |

### 第 2 层：行动层（Action Layer）

评估 Agent 的"操作"质量：

| 指标 | 含义 | 衡量方式 |
|------|------|---------|
| Tool Correctness | 是否选择了正确的工具 | 对比预期工具序列 |
| Argument Correctness | 工具参数是否正确 | 验证参数值与类型 |

### 第 3 层：执行层（Execution Layer）

评估 Agent 的"结果"质量：

| 指标 | 含义 | 衡量方式 |
|------|------|---------|
| Task Completion | 任务是否成功完成 | 二元判断 + 部分完成度 |
| Step Efficiency | 完成效率如何 | 实际步数 / 最优步数 |

### 评估流程示例

```python
from deepeval.metrics import (
    TaskCompletionMetric,
    ToolCorrectnessMetric,
    AgentGoalAccuracyMetric,
)
from deepeval.test_case import LLMTestCase

test_case = LLMTestCase(
    input="帮我查询最近三个月的销售数据并生成趋势报告",
    actual_output=agent_output,
    expected_tools=["query_database", "generate_chart", "create_report"],
    expected_output="包含Q4销售趋势分析的完整报告",
)

task_metric = TaskCompletionMetric(threshold=0.8)
tool_metric = ToolCorrectnessMetric(threshold=0.9)

task_metric.measure(test_case)
tool_metric.measure(test_case)

print(f"任务完成度: {task_metric.score}")
print(f"工具正确率: {tool_metric.score}")
```

## 五、Eval-to-Production Gap：不容忽视的现实

2026 年的生产实践揭示了一个严峻问题：**评估环境中的表现与生产环境之间存在显著落差。**

根据最新研究数据：

| 环境 | 表现 |
|------|------|
| 评估集（Eval Set） | 91% 成功率 |
| 生产环境（Production） | **68% 成功率** |
| 落差 | **23 个百分点** |

### 导致落差的三大因素

1. **分布失配（Distribution Mismatch）**
   - 评估集的输入分布无法覆盖生产环境的长尾场景
   - 用户的表达方式远比测试用例多样

2. **多 Agent 协作失败（Coordination Failures）**
   - 单 Agent 评估通过，但多 Agent 编排时出现信息丢失、死锁、循环调用
   - Agent 间的隐式依赖在评估环境中难以复现

3. **非确定性方差（Non-deterministic Variance）**
   - 同一输入多次运行产生不同结果
   - 温度参数、上下文窗口变化、模型版本更新都引入方差

### 应对策略

```
评估金字塔（Evaluation Pyramid）

         /\
        /  \     生产环境在线评估（持续监控）
       /    \    - 实时指标、异常告警、采样审核
      /------\
     /        \   预发布回归测试（CI/CD 集成）
    /          \  - 覆盖核心场景的端到端测试
   /------------\
  /              \ 单元级评估（开发阶段）
 /                \ - Prompt 级别的精确评估
/------------------\
```

- **持续评估**：不要只在上线前评估，要在生产中持续运行评估器
- **影子模式（Shadow Mode）**：新版本先在影子模式下运行，与生产版本对比
- **金丝雀发布（Canary Release）**：逐步放量，实时监控质量指标
- **多样化测试集**：引入对抗样本、边界情况、多语言场景

## 六、实战：构建你的 Agent 可观测性体系

### 最小可行方案（MVP）

如果你刚开始搭建 Agent 可观测性，建议从以下几步开始：

1. **集成 Tracing**：使用 LangSmith、Arize Phoenix 或 OpenTelemetry 接入追踪
2. **记录核心指标**：Token 消耗、延迟、任务成功率
3. **建立基线**：用第一批生产数据建立质量基线
4. **添加告警**：对成功率下降、成本异常、延迟飙升设置告警

### 成熟方案

在 MVP 基础上逐步演进：

1. **自动化评估**：部署在线评估器（忠实度、安全性、PII 检测）
2. **A/B 测试框架**：对比不同 Prompt、模型、策略的效果
3. **人工审核工作流**：建立专家审核的标准化流程
4. **成本归因**：将 Token 成本归因到具体功能、用户群、业务线
5. **根因分析**：当质量下降时，通过 Trace 快速定位问题步骤

## 七、总结与展望

Agent 可观测性正在从"有则更好"变为"生产必需"。核心趋势：

1. **标准化**：OpenTelemetry GenAI 语义约定将成为统一标准，打破厂商锁定
2. **语义化**：从传统 APM 指标走向语义级监控——不只监控性能，还监控"理解力"
3. **评估内化**：评估不再是独立的测试阶段，而是融入运行时的持续过程
4. **成本可见**：Token 经济学让成本成为可观测性的一等公民

**记住一个核心原则：如果你无法观测 Agent 的推理过程，你就无法信任它的决策。**

---

## 参考资源

- [OpenTelemetry GenAI Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans)
- [DeepEval Agent Evaluation Metrics](https://deepeval.com/guides/guides-ai-agent-evaluation-metrics)
- [Maxim AI - Agent Observability Guide](https://www.getmaxim.ai/articles/agent-observability-the-definitive-guide-to-monitoring-evaluating-and-perfecting-production-grade-ai-agents/)
- [Zylos Research - OpenTelemetry for AI Agents](https://zylos.ai/research/2026-02-28-opentelemetry-ai-agent-observability)
