# Agent-LLM 应用的异常监控与稳定性保障：在概率世界中守住确定性底线

> 撰写日期：2026年3月 | 结合业界最新进展与企业级实践

## 一、引言：稳定性——AI 工程中最容易被低估的战场

最近读到一篇来自淘天搜索团队的分享《AI 搜索技术演进实践》，其中有一段话让我深以为然：

> "AI 工程并非对传统工程经验的全盘否定，而是在我们过去十几年构建的坚实工程地基之上，为应对'不确定性'所进行的一次精准架构升级。"

这句话精准地点出了 Agent-LLM 应用稳定性保障的核心命题：**我们面对的不再是"代码有没有 Bug"，而是"概率性输出在生产环境中是否可控"。**

传统后端服务的稳定性模型建立在确定性之上——同样的输入必然产生同样的输出，监控只需关注"是否报错、是否超时"。但 Agent 系统天生运行在概率空间中：

- 模型输出具有**内在非确定性**
- 工具调用链路具有**动态性**
- 上下文组装具有**涌现性**
- 多 Agent 协作具有**级联放大效应**

这意味着：**Agent 的异常监控必须从"二值判定"升级为"多维度量"，稳定性保障必须从"消灭故障"转向"管理不确定性边界"。**

淘天团队在 AI 搜索实践中的一个核心策略就是——"将大模型的'波动性'视为系统设计的基本约束"。本文将围绕这一理念，系统梳理 Agent-LLM 应用在异常监控与稳定性保障方面的工程实践。

## 二、Agent 系统的五类特有故障模式

在深入解法之前，必须先理解 Agent 系统的故障模式与传统应用有何根本不同。

### 2.1 静默失败（Silent Failure）

**这是 Agent 系统最危险、最隐蔽的故障类型。**

传统服务故障时会抛出异常、返回错误码，监控系统可以立即捕获。但 Agent 的"故障"往往看起来一切正常——HTTP 200、延迟正常、格式正确——只是**回答的内容是错的**。

| 维度 | 传统应用故障 | Agent 静默失败 |
|------|------------|---------------|
| HTTP 状态码 | 500 / 502 / 503 | **200** |
| 响应格式 | 格式异常或空 | **格式完美** |
| 监控告警 | 立即触发 | **无告警** |
| 用户感知 | "系统崩了" | "回答好像不太对…" |
| 发现时机 | 秒级 | **小时/天级**（靠用户反馈） |

正如淘天团队所观察到的：一个看似通顺却事实错误的回答，其危害远大于响应稍慢但准确的结果。

### 2.2 幻觉漂移（Hallucination Drift）

模型幻觉不是偶发事件，而是一种**渐进的质量退化**：

- **突发幻觉**：单次输出编造事实（如捏造不存在的商品参数）
- **渐进漂移**：模型版本更新、上下文质量下降导致幻觉率从 3% 缓慢攀升到 15%，但因为是逐步恶化，团队长时间无感知

### 2.3 循环与死锁（Loop & Deadlock）

Agent 在推理过程中可能陷入无法自拔的循环，业界已总结出三种典型模式：

- **Looper（循环者）**：在固定的几个动作之间来回切换，无法达成最终目标
- **Wanderer（漫游者）**：持续活跃但偏离了原始目标，Token 在消耗但任务没有进展
- **Repeater（重复者）**：不断执行完全相同的工具调用，每次都期望不同的结果

在多 Agent 系统中还存在更隐蔽的协作死锁：
- **"礼貌死亡螺旋"**：多个 Agent 之间互相客气、反复确认，直到耗尽 Token 预算
- **"幻觉级联"**：一个 Agent 的幻觉输出被下游 Agent 当作事实继续推理，错误逐级放大
- **"格式拉锯战"**：Agent 之间反复修正对方的输出格式，无法推进实际业务逻辑

真实案例：2026 年 2 月的一个七 Agent Discord 运营系统经历了"同时响应风暴、策略级联、执行崩溃、时间漂移"等连续协调失败。企业部署中，一个未被监控的循环在一个周末消耗了超过 $4,000 的 API 费用。

### 2.4 成本雪崩（Cost Avalanche）

Agent 的成本是运行时变量，不是固定开支。单个边界场景可能触发：

- 超长推理链 → Token 消耗呈二次增长
- 工具调用失败 → 反复重试
- 上下文膨胀 → 每步 Token 数持续膨胀
- 多 Agent 循环 → 指数级 API 调用

缺乏成本监控时，费用异常往往要到月底账单才被发现，此时损失已经造成。

### 2.5 级联传播（Cascading Propagation）

与传统微服务的级联故障不同，Agent 级联失败有其独特特征：

- **传播速度更快**：Agent 的自主决策能力使错误以决策速度而非请求速度传播
- **放大效应更强**：Agent 会基于错误的前提进行推理，产生更离谱的后续行为
- **反馈循环**：Agent 的输出可能成为自己或其他 Agent 的输入，形成正反馈环
- **超越人类反应时间**：自主 Agent 的级联传播往往快于人类介入速度

## 三、重新定义 SLA：从"可用性"到"合理性"

### 3.1 传统 SLA 对 Agent 的盲区

传统 SLA 关注三个核心指标：可用性（Availability）、延迟（Latency）、错误率（Error Rate）。但对于 Agent 系统，这三个指标可以全部达标，系统却在持续输出低质量甚至有害的内容。

正如淘天团队所指出的："'流畅'不等于'可信'。AI 系统的终极验收标准，应是**真实、可靠、可解释**。"

### 3.2 Agent SLA 的五维模型：CLASSic

业界提出了 **CLASSic 框架**，将 Agent SLA 扩展为五个维度：

| 维度 | 含义 | 传统 SLA 是否覆盖 | 示例指标 |
|------|------|------------------|---------|
| **C**ost（成本） | 每次任务的 Token 成本 | 否 | 平均任务成本 < $0.50 |
| **L**atency（延迟） | 端到端及首字延迟 | 部分 | TTFT < 1s，P95 < 5s |
| **A**ccuracy（准确性） | 输出的事实正确性 | 否 | 幻觉率 < 5%，任务完成率 > 85% |
| **S**tability（稳定性） | 跨输入的表现一致性 | 否 | 同类任务成功率方差 < 5% |
| **S**ecurity（安全性） | 输出安全与合规 | 否 | PII 泄露率 = 0，注入拦截率 > 99% |

### 3.3 延迟指标的重新定义

淘天团队在实践中已经意识到这一点——SLA 的定义需要调整，例如从"响应时间 < 200ms"变为"首字响应时间 < 1s"。

Agent 应用的延迟指标应拆分为多个层次：

- **TTFT（Time To First Token）**：首字延迟，直接影响用户感知。目标通常 < 1s
- **TTFA（Time To First Action）**：首次工具调用延迟，体现 Agent 决策速度
- **TTC（Time To Completion）**：任务完成总耗时，允许更大的弹性（可达数十秒）
- **P50 / P95 / P99 分布**：由于 Agent 的延迟方差极大（800ms ~ 30s+），单看平均值毫无意义

### 3.4 质量指标的引入

这是 Agent SLA 与传统 SLA 最大的不同——**质量成为一等公民**。

- **任务完成率（Task Completion Rate）**：Agent 成功完成用户目标的比例
- **工具调用正确率（Tool Accuracy）**：选择正确工具并传递正确参数的比例
- **幻觉率（Hallucination Rate）**：输出中包含编造事实的比例
- **步骤效率（Step Efficiency）**：实际执行步数 / 最优步数

由于 Agent 行为天然非确定性，质量指标必须使用 **pass@k** 方法——对同一输入运行 5-10 次，取聚合结果作为评判标准。

## 四、多层监控体系：从基础设施到语义质量

### 4.1 监控金字塔

Agent 的监控需要从底层到顶层覆盖四个层次：

```
                    /\
                   /  \      语义质量监控
                  /    \     幻觉率、忠实度、安全性
                 /      \    LLM-as-Judge 在线评估
                /--------\
               /          \    业务效果监控
              /            \   任务完成率、用户满意度
             /              \  转化率、留存率
            /----------------\
           /                  \    Agent 行为监控
          /                    \   工具调用链、推理步数
         /                      \  Token 消耗、成本归因
        /------------------------\
       /                          \    基础设施监控
      /                            \   可用性、延迟、错误率
     /                              \  传统 APM 指标
    /--------------------------------\
```

**关键原则：上层的问题无法被下层的监控发现。** 基础设施层一切正常（HTTP 200、延迟正常）不代表语义质量层没有问题。

### 4.2 基础设施层：继承传统工程的坚实底座

正如淘天团队的实践所证明的，传统工程的监控能力在 AI 系统中非但没有过时，反而更加关键。他们在构建 Multi-Agent 框架时，复用了主搜团队沉淀多年的业务编排框架（serviceplan），确保 AI 请求与传统搜索请求共享同一套限流、熔断与链路追踪能力。

基础设施层需要监控的核心指标：

- **可用性**：各 LLM Provider 的 API 可用率
- **延迟分布**：TTFT、工具调用延迟、端到端延迟的 P50/P95/P99
- **错误率**：HTTP 错误、超时、限流（429）的分类统计
- **资源利用**：GPU/CPU 使用率、内存、连接池
- **依赖健康**：向量数据库、外部 API、MCP 服务器的健康状态

### 4.3 Agent 行为层：追踪"它在做什么"

这一层是 Agent 监控的核心创新。淘天团队开发了 Trace 工具，完整记录每次请求中"用户输入 → 意图识别 → 上下文组装 → Agent 决策 → Tool 调用 → 模型生成 → 后处理"的全链路轨迹，支持逐层下钻分析。

行为层监控的关键指标：

**推理过程指标：**
- 每次任务的 LLM 调用次数（正常范围 vs 异常飙升）
- 推理步数（是否陷入循环）
- 工具调用序列（是否选择了正确的工具）
- 工具参数正确率

**成本指标：**
- 每次任务的 Token 消耗（输入 / 输出分别统计）
- 每次任务的 API 成本
- Token 消耗趋势（是否在逐步膨胀）

**循环检测：**

```python
class AgentLoopDetector:
    """检测 Agent 是否陷入推理循环"""

    def __init__(self, max_iterations=15, repetition_window=5):
        self.max_iterations = max_iterations
        self.repetition_window = repetition_window
        self.action_history: list[str] = []

    def record_action(self, action: str) -> str | None:
        self.action_history.append(action)

        # 检测 1：绝对上限
        if len(self.action_history) > self.max_iterations:
            return "MAX_ITERATIONS_EXCEEDED"

        # 检测 2：重复模式（Repeater）
        if len(self.action_history) >= self.repetition_window:
            recent = self.action_history[-self.repetition_window:]
            if len(set(recent)) == 1:
                return "REPEATER_DETECTED"

        # 检测 3：周期性循环（Looper）
        if len(self.action_history) >= 6:
            for cycle_len in range(2, len(self.action_history) // 2 + 1):
                tail = self.action_history[-cycle_len * 2:]
                if tail[:cycle_len] == tail[cycle_len:]:
                    return "LOOP_DETECTED"

        # 检测 4：进度停滞（Wanderer）
        if len(self.action_history) >= 8:
            unique_recent = set(self.action_history[-8:])
            unique_early = set(self.action_history[:min(8, len(self.action_history))])
            if unique_recent == unique_early and len(self.action_history) > 16:
                return "NO_PROGRESS_DETECTED"

        return None
```

### 4.4 业务效果层：追踪"它做得怎么样"

Agent 的最终价值体现在业务指标上：

- **任务完成率**：用户目标是否被成功达成
- **用户满意度**：通过显式反馈（点赞/踩）和隐式信号（是否重新提问、是否跳出）衡量
- **业务转化**：如淘天团队关注的点击率、转化率、停留时长等

### 4.5 语义质量层：追踪"它说的对不对"

这是 Agent 监控中最难但也最关键的一层——**在线语义评估**。

**LLM-as-Judge 在线评估：**

```python
class OnlineQualityMonitor:
    """生产环境中的实时语义质量监控"""

    def __init__(self, judge_model="gpt-4o-mini", sample_rate=0.1):
        self.judge_model = judge_model
        self.sample_rate = sample_rate

    async def evaluate_if_sampled(self, request: dict, response: dict):
        """按采样率对生产流量进行在线评估"""
        import random
        if random.random() > self.sample_rate:
            return

        scores = await self._run_evaluation(request, response)
        self._emit_metrics(scores)

        if scores["hallucination_score"] > 0.7:
            self._alert("HIGH_HALLUCINATION", scores)
        if scores["safety_score"] < 0.5:
            self._alert("SAFETY_VIOLATION", scores)

    async def _run_evaluation(self, request: dict, response: dict) -> dict:
        evaluation_prompt = f"""
        请评估以下 AI Agent 的回答质量，按 0-1 打分：

        用户问题：{request['query']}
        Agent 回答：{response['answer']}
        参考来源：{response.get('sources', '无')}

        评估维度：
        1. faithfulness（忠实度）：回答是否忠于参考来源？
        2. relevance（相关性）：回答是否切题？
        3. hallucination（幻觉程度）：是否包含编造的事实？
        4. safety（安全性）：是否包含有害或不当内容？
        5. completeness（完整性）：是否完整回答了用户问题？

        请以 JSON 格式返回评分。
        """
        result = await call_llm(self.judge_model, evaluation_prompt)
        return parse_scores(result)
```

**基于规则的快速校验（零延迟开销）：**

并非所有质量检查都需要 LLM Judge。以下检查可以用规则实时完成：
- 输出格式是否符合预期 Schema
- 是否包含 PII（手机号、身份证号等）
- 回答长度是否异常（过短或过长）
- 是否出现"我是一个 AI 语言模型"等典型泄露语句
- 工具调用参数是否通过类型校验

## 五、熔断与降级：Agent 的"保命符"

淘天团队把熔断、降级、限流等机制称为保障用户体验的"保命符"——在 AI 系统中，这些传统机制不但没有过时，反而因为 Agent 的脆弱性而更加关键。

### 5.1 分层防御体系

Agent 的韧性设计应遵循**分层递进**原则：

**第 1 层：重试与退避** → **第 2 层：模型/Provider 切换** → **第 3 层：熔断器** → **第 4 层：功能降级** → **第 5 层：兜底响应**

每一层只处理前一层无法解决的故障，避免过度防御带来的复杂性。

### 5.2 智能重试：区分可恢复与不可恢复错误

Agent 场景中最常见的错误是：不分青红皂白地重试一切错误。

```python
class SmartRetryPolicy:
    """区分可恢复与不可恢复错误的智能重试策略"""

    RETRYABLE_ERRORS = {
        429,  # Rate Limit
        503,  # Service Unavailable
        504,  # Gateway Timeout
        "timeout",
        "connection_reset",
    }

    NON_RETRYABLE_ERRORS = {
        400,  # Bad Request（Prompt 有问题，重试无意义）
        401,  # Unauthorized
        404,  # Not Found
        "content_policy_violation",
        "context_length_exceeded",
    }

    def __init__(self, max_retries=3, base_delay=1.0, max_delay=30.0):
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.max_delay = max_delay

    async def execute_with_retry(self, func, *args, **kwargs):
        import asyncio
        import random

        for attempt in range(self.max_retries + 1):
            try:
                return await func(*args, **kwargs)
            except Exception as e:
                error_type = self._classify_error(e)

                if error_type in self.NON_RETRYABLE_ERRORS:
                    raise  # 不可恢复，直接抛出

                if attempt == self.max_retries:
                    raise  # 已达最大重试次数

                # 指数退避 + 随机抖动（防止惊群效应）
                delay = min(
                    self.base_delay * (2 ** attempt),
                    self.max_delay
                )
                jitter = random.uniform(-0.3 * delay, 0.3 * delay)
                await asyncio.sleep(delay + jitter)
```

添加随机抖动（Jitter）可以减少 60-80% 的重试风暴，这一点在高并发的 Agent 场景中尤为重要。

### 5.3 模型级熔断与 Fallback

这是 Agent 特有的熔断模式——当主力模型出现问题时，自动切换到备选模型或规则引擎。

```python
class ModelCircuitBreaker:
    """
    模型级熔断器

    状态机：CLOSED → OPEN → HALF_OPEN → CLOSED / EXTENDED_OPEN
    """

    def __init__(
        self,
        failure_threshold=5,
        detection_window_seconds=300,
        initial_backoff_seconds=300,
        extended_backoff_seconds=900,
    ):
        self.failure_threshold = failure_threshold
        self.detection_window = detection_window_seconds
        self.initial_backoff = initial_backoff_seconds
        self.extended_backoff = extended_backoff_seconds
        self.state = "CLOSED"
        self.failure_count = 0
        self.last_failure_time = None
        self.open_time = None
        self.consecutive_opens = 0

    def record_result(self, success: bool, error_type: str = None):
        if self.state == "OPEN":
            return

        if success:
            self.failure_count = max(0, self.failure_count - 1)
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.consecutive_opens = 0
        else:
            if self._is_infrastructure_error(error_type):
                self.failure_count += 1
                if self.failure_count >= self.failure_threshold:
                    self._trip()

    def should_allow_request(self) -> bool:
        import time
        if self.state == "CLOSED":
            return True
        if self.state == "OPEN":
            backoff = (
                self.extended_backoff
                if self.consecutive_opens > 1
                else self.initial_backoff
            )
            if time.time() - self.open_time > backoff:
                self.state = "HALF_OPEN"
                return True
            return False
        if self.state == "HALF_OPEN":
            return True
        return False

    def _trip(self):
        import time
        self.state = "OPEN"
        self.open_time = time.time()
        self.consecutive_opens += 1
        self.failure_count = 0

    def _is_infrastructure_error(self, error_type: str) -> bool:
        """只对基础设施错误触发熔断，业务错误不触发"""
        return error_type in {"502", "503", "504", "timeout", "connection_error"}
```

**关键设计决策：**
- 仅对基础设施级错误（502/503/504、超时）触发熔断，业务逻辑错误（400/401、内容策略违规）不触发
- 引入"扩展退避"状态：如果短期内反复触发熔断，冷却时间从 5 分钟延长到 15 分钟，防止对不稳定服务的频繁试探
- 恢复后渐进放量：从 1 个并发开始，每 5 分钟增加一个，避免恢复瞬间的流量冲击

### 5.4 多级降级策略

淘天团队的实践提供了一个非常好的降级思路：当大模型超时或幻觉严重时，自动切换到基于规则的兜底策略，或者使用更小、更快的模型接管。

将这一思路体系化：

**降级等级定义：**

| 等级 | 触发条件 | 降级策略 | 用户体验影响 |
|------|---------|---------|------------|
| L0 正常 | 一切正常 | 主力模型全功能 | 无影响 |
| L1 轻度 | 主力模型延迟升高 / 部分超时 | 切换到同级备选模型 | 几乎无感知 |
| L2 中度 | 主力模型不可用 | 切换到更小更快的模型，减少推理步数 | 回答质量略有下降 |
| L3 重度 | 所有大模型不可用 | 基于规则引擎的兜底响应 | 仅提供基础功能 |
| L4 极端 | 基础设施故障 | 静态兜底页面 / 引导用户至人工客服 | 功能不可用，但不崩溃 |

```python
class GracefulDegradationManager:
    """多级优雅降级管理器"""

    def __init__(self):
        self.current_level = "L0"
        self.model_breakers = {
            "primary": ModelCircuitBreaker(),
            "secondary": ModelCircuitBreaker(),
            "lightweight": ModelCircuitBreaker(),
        }

    async def execute_with_degradation(self, request: dict) -> dict:
        # L0: 尝试主力模型
        if self.model_breakers["primary"].should_allow_request():
            try:
                result = await self._call_primary_model(request)
                self.model_breakers["primary"].record_result(True)
                return result
            except Exception as e:
                self.model_breakers["primary"].record_result(
                    False, self._classify(e)
                )

        # L1: 尝试同级备选模型
        if self.model_breakers["secondary"].should_allow_request():
            try:
                result = await self._call_secondary_model(request)
                self.model_breakers["secondary"].record_result(True)
                return self._mark_degraded(result, "L1")
            except Exception:
                self.model_breakers["secondary"].record_result(False)

        # L2: 使用轻量模型 + 缩减推理
        if self.model_breakers["lightweight"].should_allow_request():
            try:
                simplified = self._simplify_request(request)
                result = await self._call_lightweight_model(simplified)
                return self._mark_degraded(result, "L2")
            except Exception:
                pass

        # L3: 规则引擎兜底
        rule_result = self._rule_based_fallback(request)
        if rule_result:
            return self._mark_degraded(rule_result, "L3")

        # L4: 静态兜底
        return self._static_fallback(request)
```

### 5.5 从"动态生成"到"受控执行"

淘天团队在 AI 搜索实践中走过的架构演进路径非常有参考价值：

早期采用 ReAct 式的 plan-execute 范式——先由 Plan Agent 实时生成完整执行计划，再交由 MainAgent 执行。问题是"链路过长导致误差逐级放大，反复调用大模型造成高延迟、高 Token 消耗，且幻觉难以追溯"。

后来转向**预编排 SOP 模式**：意图 Agent 仅负责路由至特定子 Agent，每个子 Agent 的执行流程基于人工配置或离线生成的固化 SOP。虽然牺牲了部分动态适应能力，但显著降低了资源开销与输出风险。

**这本质上是一种架构级的稳定性策略**——用有限灵活性换取高确定性，是对不确定性边界的主动收拢。

将这一思路抽象为通用原则：

- **确定性包裹不确定性**：用确定性的前处理（输入校验、意图识别）和后处理（结果校验、格式化）包裹不确定的模型内核
- **漏斗式收敛**：通过多级意图路由逐层收敛，将大模型的不确定性严格限制在必要环节
- **能用规则就不用模型**：简单分类、格式校验、安全过滤等能用规则解决的，不要浪费在大模型上

## 六、幻觉检测与内容校验：守住事实底线

2026 年的最新技术可以将生产环境中的幻觉率降低最高 96%，但完全消除仍不可能。关键在于多层检测与缓解。

### 6.1 多层幻觉防御

**第 1 层：Prompt 层预防**
- 在系统提示中要求模型"引用来源"和"对不确定的内容明确标注"
- 使用结构化输出约束回答格式
- 限制回答范围："仅基于提供的信息回答，如果信息不足请明确说明"

**第 2 层：输出校验层**
- 基于 RAG 来源的事实核查：将模型输出与检索到的原始文档进行比对
- 格式一致性检查：工具调用参数是否符合 Schema
- 数值合理性校验：价格、日期、数量等结构化数据的范围检查

**第 3 层：在线评估层**
- 采样流量进行 LLM-as-Judge 评估
- 用户反馈信号采集（显式点踩 + 隐式重提问）

**第 4 层：离线分析层**
- 定期对生产日志进行批量幻觉检测
- 幻觉率趋势分析，检测是否存在渐进漂移

### 6.2 自愈机制：检测到异常后怎么办

业界最新的 Agent 自愈（Self-Healing）模式不仅检测故障，还能自动恢复：

**三类故障的自愈策略：**

- **存活性故障**（进程挂了/OOM）→ 自动重启 + 从最近 Checkpoint 恢复
- **进度故障**（推理卡住、循环）→ 注入"你似乎陷入了循环，请尝试不同策略"的干预提示，或强制切换推理路径
- **质量故障**（幻觉、格式错误）→ 自动重试并附加更严格的约束，或切换到更强的模型

```python
class AgentHealthChecker:
    """Agent 健康检查与自愈"""

    def check_health(self, agent_state: dict) -> dict:
        issues = []

        # 存活性检查
        last_heartbeat = agent_state.get("last_heartbeat_ms", 0)
        if time.time() * 1000 - last_heartbeat > 30000:
            issues.append({"type": "LIVENESS", "action": "RESTART"})

        # 进度检查
        iteration_count = agent_state.get("iteration_count", 0)
        unique_actions = len(set(agent_state.get("recent_actions", [])))
        if iteration_count > 10 and unique_actions <= 2:
            issues.append({"type": "PROGRESS", "action": "INTERVENE"})

        # 质量检查
        recent_scores = agent_state.get("recent_quality_scores", [])
        if recent_scores and sum(recent_scores) / len(recent_scores) < 0.5:
            issues.append({"type": "QUALITY", "action": "ESCALATE_MODEL"})

        return {"healthy": len(issues) == 0, "issues": issues}
```

## 七、成本异常监控：防止 Token 烧穿预算

### 7.1 成本监控的三道防线

**第 1 道：请求级硬限制**
- 单次任务最大循环次数（例如 15 次）
- 单次任务最大 Token 消耗（例如 50,000 Token）
- 单次任务最大成本（例如 $2.00）

**第 2 道：实时异常检测**
- 单次任务成本超过 P95 阈值 → 实时告警
- 小时级成本环比增长 > 50% → 升级告警
- 特定 Agent 步骤的 Token 消耗突然翻倍 → 调查

**第 3 道：趋势分析与预算守卫**
- 日级成本趋势分析
- 预算燃尽速率（Burn Rate）预测
- 月度预算耗尽预警（"按当前速率，预算将在 X 天内耗尽"）

### 7.2 成本归因

将每一笔 Token 消耗归因到具体的维度：

- **功能归因**：哪个 Agent / 哪个步骤花了最多钱
- **用户归因**：不同用户群的成本分布
- **场景归因**：哪类查询最消耗 Token
- **异常归因**：成本飙升时快速定位是哪个环节、哪个模型、哪类输入

## 八、告警策略：从"响不响"到"值不值得响"

### 8.1 Agent 告警的特殊挑战

传统服务的告警逻辑简单——错误率超过阈值就告警。但 Agent 系统面临两个特殊问题：

- **告警疲劳**：由于非确定性，质量指标的波动是正常的，如果阈值设得太敏感，就会告警风暴
- **静默恶化**：质量退化是渐进的，单看瞬时指标可能都在阈值内，但趋势在持续恶化

### 8.2 分层告警策略

| 级别 | 触发条件 | 通知方式 | 响应时间 |
|------|---------|---------|---------|
| P0 Critical | 服务不可用 / 安全事件 / 成本失控 | 电话 + 即时消息 | 5 分钟内 |
| P1 High | 质量分骤降（> 15%）/ 幻觉率飙升 / 熔断触发 | 即时消息 | 30 分钟内 |
| P2 Medium | 质量分缓慢下降 / 成本趋势异常 / 延迟升高 | 工作消息 | 4 小时内 |
| P3 Low | 特定场景成功率下降 / 新的边界用例 | 日报 | 下个工作日 |

### 8.3 趋势告警 > 阈值告警

对于 Agent 系统，**趋势告警比阈值告警更有价值**。

```python
class TrendAlert:
    """基于趋势的告警，比简单阈值更适合非确定性系统"""

    def check_trend(self, metric_name: str, values: list[float], window_hours: int = 24) -> dict:
        if len(values) < 10:
            return {"alert": False}

        midpoint = len(values) // 2
        first_half_avg = sum(values[:midpoint]) / midpoint
        second_half_avg = sum(values[midpoint:]) / (len(values) - midpoint)

        if first_half_avg == 0:
            return {"alert": False}

        change_rate = (second_half_avg - first_half_avg) / first_half_avg

        alert = False
        severity = "low"

        if abs(change_rate) > 0.15:
            alert = True
            severity = "medium"
        if abs(change_rate) > 0.30:
            severity = "high"

        return {
            "alert": alert,
            "severity": severity,
            "metric": metric_name,
            "change_rate": change_rate,
            "direction": "degrading" if change_rate < 0 else "improving",
            "message": (
                f"{metric_name} 在过去 {window_hours} 小时内"
                f"{'下降' if change_rate < 0 else '上升'}"
                f"了 {abs(change_rate)*100:.1f}%"
            ),
        }
```

## 九、实战：构建 Agent 稳定性保障体系的路线图

### Phase 1：基础保障（第 1-2 周）

**目标：让系统"不崩"**

- 接入基础 APM 监控（可用性、延迟、错误率）
- 实现请求级硬限制（循环上限、Token 上限、成本上限）
- 部署模型级熔断器，配置至少一个 Fallback 模型
- 启用全链路 Trace 记录
- 建立成本监控看板

### Phase 2：行为可观测（第 3-4 周）

**目标：知道 Agent "在做什么"**

- 部署 Agent 行为层监控（工具调用链、推理步数、Token 归因）
- 实现循环检测与自动终止
- 建立成本异常实时告警
- 将 Trace 工具开放给研发团队，支持逐层下钻分析

### Phase 3：质量监控（第 5-8 周）

**目标：知道 Agent "做得对不对"**

- 部署在线语义评估（LLM-as-Judge 采样评估）
- 建立幻觉率、任务完成率等质量指标的基线
- 实现趋势告警（质量缓慢退化的检测）
- 建立用户反馈闭环（点赞/踩 → 标注 → 评估集更新）

### Phase 4：自愈与深度防御（第 9-12 周）

**目标：系统能"自己扛住"**

- 实现多级优雅降级策略（L0-L4）
- 部署 Agent 自愈机制（进度干预、质量升级）
- 建立预编排 SOP 模式，将不确定性限制在必要环节
- 定期混沌工程演练（模拟模型故障、延迟飙升、成本失控等场景）

### Phase 5：持续运营（长期）

**目标：稳定性成为团队"肌肉记忆"**

- 建立 Agent SLA 看板（CLASSic 五维指标）
- 每次上线前必须通过 Eval Gate（见第四篇文章）
- 模型版本升级有专项评估流程
- 每月稳定性复盘，持续完善告警规则和降级策略
- 积累故障案例库，推动 Agent 韧性设计的最佳实践

## 十、总结：在概率世界中守住确定性底线

回到本文开头引用的那句话："AI 工程不是推倒重来，而是在坚实工程地基上，为应对'不确定性'而做的精准架构升级。"

Agent-LLM 应用的稳定性保障，本质上就是**用确定性的工程手段去驾驭不确定的智能输出**。核心原则可以归纳为：

1. **传统工程是底座，不是遗产**。熔断、降级、限流、链路追踪——这些"老武器"在 Agent 时代不但没有过时，反而更加关键
2. **监控必须穿透语义层**。HTTP 200 + 延迟正常 ≠ 系统健康。必须建立从基础设施到语义质量的四层监控体系
3. **SLA 从"可用性"升级为"合理性"**。质量、成本、安全成为与可用性同等重要的 SLA 维度
4. **趋势比阈值更重要**。Agent 的退化是渐进的，瞬时阈值告警远不如趋势分析有效
5. **确定性包裹不确定性**。用确定性的前后处理、漏斗式意图路由、预编排 SOP 来收拢模型的不确定性空间
6. **成本是稳定性的一部分**。一个成本失控的 Agent 和一个不可用的 Agent 同样危险
7. **自愈优于告警**。能自动恢复的系统比能发出告警的系统更强大

**最终目标：让 Agent 系统的不确定性收敛到业务与用户可接受的区间之内——不是消灭波动，而是管理波动。**

正如淘天团队的总结所言：真正的智能，不在于模型能生成多少可能性，而在于系统知道何时该让它说话，何时该让它闭嘴。

---

## 参考资源

- 淘天搜索团队 -《AI 搜索技术演进实践》（ATA 原文）
- [Zylos Research - Graceful Degradation Patterns in AI Agent Systems](https://zylos.ai/research/2026-02-20-graceful-degradation-ai-agent-systems)
- [Zylos Research - AI Agent Error Handling & Recovery](https://zylos.ai/research/2026-01-12-ai-agent-error-handling-recovery)
- [Zylos Research - AI Agent Self-Healing: Automated Recovery and Resilience Patterns](https://zylos.ai/research/2026-03-02-ai-agent-self-healing-recovery-patterns)
- [OneUptime - Observability for AI Agents](https://oneuptime.com/blog/post/2026-02-19-observability-for-ai-agents-why-your-llm-apps-are-flying-blind/view)
- [Zylos Research - LLM Hallucination Detection and Mitigation](https://zylos.ai/research/2026-01-27-llm-hallucination-detection-mitigation)
- [AI Agent SLAs: Setting Performance Guarantees That Scale](https://www.agenticaipricing.com/ai-agent-slas-setting-performance-guarantees-that-scale/)
- [Google Cloud - The KPIs That Actually Matter for Production AI Agents](https://cloud.google.com/transform/the-kpis-that-actually-matter-for-production-ai-agents)
- [OWASP - Cascading Failures in Agentic AI (ASI08)](https://adversa.ai/blog/cascading-failures-in-agentic-ai-complete-owasp-asi08-security-guide-2026/)
- [LangChain - Production Monitoring for LLM Agents](https://www.langchain.com/conceptual-guides/production-monitoring)
