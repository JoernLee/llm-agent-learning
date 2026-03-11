# Agent 敏捷交付与版本管理：当软件工程遇上非确定性系统

> 撰写日期：2026年3月 | 结合业界最新进展

## 一、为什么 Agent 应用的交付逻辑和传统软件完全不同？

如果你有传统前后端的研发经验，那么在转向 Agent 应用时，你会发现一个根本性的差异——**你的"代码"只是系统行为的一部分，而且可能不是最重要的那部分。**

### 传统应用 vs Agent 应用：交付要素对比

| 维度 | 传统前后端应用 | LLM Agent 应用 |
|------|-------------|---------------|
| **行为决定因素** | 代码逻辑 | 代码 + Prompt + 模型 + 工具 + 记忆 + 上下文 |
| **确定性** | 同一输入 → 同一输出 | 同一输入 → **不同输出**（非确定性） |
| **版本化对象** | 源代码 + 配置 | 源代码 + Prompt + 模型版本 + 工具Schema + 嵌入 + 记忆 |
| **测试方式** | 断言：`assert result == expected` | **语义评估**：意思对不对？忠实不忠实？ |
| **回滚** | 回滚代码即可恢复 | 回滚代码 ≠ 恢复行为（有状态） |
| **发布影响** | 功能变化 | **行为漂移**——可能悄无声息 |
| **成本模型** | 基本固定（计算资源） | **动态变化**（Token 按用量计费） |
| **失败模式** | 崩溃、报错、异常 | **静默失败**——HTTP 200 但输出质量下降 |

一句话总结：**传统应用的"部署"是一个离散事件，Agent 应用的"行为"是一个持续演变的过程。**

这导致了一系列连锁反应：你的 CI/CD Pipeline 要重新设计、测试策略要重新定义、版本管理要多维度覆盖、回滚机制需要全新思路。

## 二、Agent 应用的"四层版本模型"

业界已形成共识：Agent 的版本不是一个单一数字，而是一个**四层分类体系**。每一层都可以独立变化，必须独立追踪。

```
Agent 版本 = 认知层 × 模型层 × 知识层 × 工具层

┌─────────────────────────────────────────────────────┐
│  第 1 层：认知层（Cognitive Layer）                    │
│  - System Prompt 版本                                │
│  - Few-shot 示例                                     │
│  - 推理策略（ReAct / CoT / Plan-and-Execute）         │
│  - 输出格式约束                                      │
├─────────────────────────────────────────────────────┤
│  第 2 层：模型层（Model Layer）                        │
│  - 基础模型版本（gpt-4o-2024-11-20 vs 2025-03-01）   │
│  - 模型参数（temperature, top_p, max_tokens）        │
│  - 路由策略版本                                      │
├─────────────────────────────────────────────────────┤
│  第 3 层：知识层（Knowledge Context）                  │
│  - RAG 知识库版本                                    │
│  - 嵌入模型版本                                      │
│  - 向量索引版本                                      │
│  - Agent 长期记忆状态                                │
├─────────────────────────────────────────────────────┤
│  第 4 层：工具层（Tool Contracts）                     │
│  - 工具 Schema 定义版本                               │
│  - MCP 服务器版本                                    │
│  - 外部 API 合约版本                                 │
│  - 工具权限策略版本                                   │
└─────────────────────────────────────────────────────┘
```

### 为什么要分层？

因为**不同层的变更频率和风险等级完全不同**：

| 层级 | 变更频率 | 风险等级 | 典型触发者 |
|------|---------|---------|-----------|
| 认知层（Prompt） | 高（每天/每周） | 中-高 | Prompt 工程师 / 产品经理 |
| 模型层 | 中（每月） | 高 | 模型提供商 / AI 工程师 |
| 知识层 | 高（每天/实时） | 中 | 数据团队 / 自动化管道 |
| 工具层 | 低（每月/每季） | 极高 | 后端工程师 / 平台团队 |

**关键洞察**：业界数据显示，**工具版本变更导致了 60% 的生产 Agent 故障**，模型漂移占 40%。而这两者恰恰是传统 Git 工作流最容易忽略的。

## 三、Prompt 版本管理——像管理代码一样管理 Prompt

### 问题：Prompt Drift（提示词漂移）

MIT 2025 年的一项研究发现，**95% 的企业 AI 试点项目失败**，未受管控的 Prompt 变更是重要原因之一。如果团队还在用 `summarize_v3_FINAL_actually_final.txt` 这种方式管理 Prompt，那基本是在裸奔。

### 方案 1：Git 原生管理（推荐起步方案）

将 Prompt 作为独立文件纳入代码仓库，享受完整的 Git 工作流：

```
项目结构：
my-agent-app/
├── src/
│   ├── agent.py
│   ├── tools/
│   └── ...
├── prompts/                    # Prompt 独立目录
│   ├── system/
│   │   ├── main_v2.1.0.md     # 主系统 Prompt
│   │   └── CHANGELOG.md       # Prompt 变更日志
│   ├── tools/
│   │   ├── search_tool.md
│   │   └── database_tool.md
│   └── few_shot/
│       ├── examples_v1.json
│       └── negative_examples.json
├── evals/                      # 评估数据集
│   ├── golden_set.jsonl        # 黄金测试集
│   ├── regression_set.jsonl    # 回归测试集
│   └── adversarial_set.jsonl   # 对抗测试集
├── configs/
│   ├── model_config.yaml       # 模型配置（版本、参数）
│   ├── tool_schemas.json       # 工具 Schema 定义
│   └── routing_policy.yaml     # 路由策略
└── agent_manifest.yaml         # Agent 版本清单（见下文）
```

#### Agent 版本清单（Agent Manifest）

这是 Agent 应用的核心配置文件，集中声明所有四层的版本：

```yaml
# agent_manifest.yaml
# Agent 版本清单 —— 完整描述 Agent 的行为配置
agent:
  name: "customer-support-agent"
  version: "2.3.1"  # Agent 整体版本（语义化）

cognitive:
  system_prompt: "prompts/system/main_v2.1.0.md"
  few_shot_examples: "prompts/few_shot/examples_v1.json"
  reasoning_strategy: "react"  # react | cot | plan-execute
  output_format: "structured_json"

model:
  primary:
    provider: "anthropic"
    model: "claude-sonnet-4-20250514"  # 精确到日期版本
    temperature: 0.3
    max_tokens: 4096
  routing:
    intent_classification: "gpt-4o-mini-2024-07-18"
    tool_selection: "gpt-4o-mini-2024-07-18"
    complex_reasoning: "claude-sonnet-4-20250514"
    final_synthesis: "gpt-4o-2024-11-20"

knowledge:
  embedding_model: "text-embedding-3-large"
  vector_index_version: "2026-03-01-snapshot"
  knowledge_base_version: "kb-v4.2"

tools:
  schemas_version: "tools-v3.1.0"
  schemas_file: "configs/tool_schemas.json"
  mcp_servers:
    - name: "database-mcp"
      version: "1.2.0"
      url: "https://mcp.internal/database"
    - name: "email-mcp"
      version: "0.9.3"
      url: "https://mcp.internal/email"

guardrails:
  input_filter_version: "guard-v2.0"
  output_filter_version: "guard-v2.0"
  permission_policy: "configs/permissions_v3.yaml"
```

#### Prompt 语义化版本规范

采用 **MAJOR.MINOR.PATCH** 格式：

**MAJOR —— 行为破坏性变更：**
- 改变了 Agent 的核心角色定义
- 移除了某类任务的处理能力
- 改变了输出格式

**MINOR —— 行为增强：**
- 增加了新的任务处理能力
- 优化了特定场景的处理质量
- 增加了新的 few-shot 示例

**PATCH —— 微调：**
- 修正措辞、减少歧义
- 修复已知的边界情况
- 优化格式和结构

#### Prompt 变更的 PR 规范

```markdown
## Prompt Change: system/main v2.0.0 → v2.1.0

### 变更类型
- [x] MINOR: 行为增强

### 变更原因
生产监控发现 Agent 在处理退款场景时成功率仅 72%，需要增强退款流程的指引。

### 具体变更
- 增加了退款场景的 few-shot 示例（3 个正面 + 1 个反面）
- 细化了退款金额计算的步骤指引

### 评估结果
| 指标 | 变更前 | 变更后 | 变化 |
|------|--------|--------|------|
| 退款场景成功率 | 72% | 89% | +17% |
| 整体任务完成率 | 85% | 86% | +1% |
| 平均 Token 消耗 | 2,340 | 2,580 | +10% |

### 回归测试
- [x] Golden Set: 128/130 通过 (98.5%)
- [x] Regression Set: 245/250 通过 (98.0%)
- [x] 对抗测试集: 47/50 通过 (94.0%)
- [x] 成本影响：+10% Token（可接受范围内）

### 回滚计划
恢复 prompts/system/main_v2.0.0.md 并重新部署。
```

### 方案 2：Prompt 管理平台（规模化方案）

当团队超过 5 人或 Prompt 迭代极频繁时，考虑使用专用平台：

| 平台 | 核心能力 | 适用场景 |
|------|---------|---------|
| Langfuse | 开源，Prompt 管理 + 追踪 + 评估一体化 | 全栈 LLMOps |
| PromptLayer | Prompt 注册表 + 版本管理 + A/B 测试 | 专注 Prompt 管理 |
| Braintrust | Prompt 管理 + 评估 + 数据集管理 | 评估驱动开发 |
| Vertex AI Prompt Registry | GCP 原生，企业级权限管控 | Google Cloud 生态 |

**混合方案（推荐）**：Git 作为 Source of Truth（唯一真相源），平台用于部署和运行时 A/B 切换。Prompt 的变更必须经过 Git PR Review，合并后自动同步到平台。

```
Git 仓库（Source of Truth）
    │
    ├── PR Review + Eval Gate ──→ 合并到主分支
    │                                  │
    │                            CI/CD Pipeline
    │                                  │
    │                    ┌─────────────┼─────────────┐
    │                    ↓             ↓             ↓
    │              Staging 环境   Canary 环境    Production
    │              (全量评估)     (5% 流量)     (全量发布)
    │
    └── Prompt 平台（运行时读取、A/B 测试、监控）
```

## 四、Agent 的 CI/CD Pipeline 设计

传统 CI/CD 关注"构建→测试→部署"。Agent CI/CD 必须增加**评估门控（Eval Gates）**——用语义评估替代传统的 Pass/Fail 断言。

### Pipeline 全景

```
┌─────────────────────────────────────────────────────────────┐
│                    Agent CI/CD Pipeline                       │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────┐ │
│  │ 代码检查  │→│ 构建打包  │→│ 单元评估  │→│ 集成评估     │ │
│  │ Lint      │  │ Build    │  │ Unit Eval│  │ Integration  │ │
│  │ Type Check│  │ Docker   │  │          │  │ Eval         │ │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘ │
│       │              │              │              │          │
│       ↓              ↓              ↓              ↓          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────┐ │
│  │ 安全扫描  │  │ 依赖锁定  │  │ 回归评估  │  │ 成本评估    │ │
│  │ Prompt   │  │ Pin All  │  │ Regression│  │ Cost Gate   │ │
│  │ Injection│  │ Versions │  │ Eval      │  │             │ │
│  └──────────┘  └──────────┘  └──────────┘  └─────────────┘ │
│                                    │              │          │
│                                    ↓              ↓          │
│                              ┌──────────────────────┐        │
│                              │   评估门控 (Eval Gate)│        │
│                              │   质量 ≥ 阈值？       │        │
│                              │   成本 ≤ 预算？       │        │
│                              │   无安全回归？        │        │
│                              └──────────┬───────────┘        │
│                                    │ 通过                     │
│                                    ↓                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────┐ │
│  │ Shadow   │→│ Canary   │→│ 渐进放量  │→│ 全量发布     │ │
│  │ 影子部署  │  │ 金丝雀   │  │ 25→50→  │  │ Production   │ │
│  │ 0% 流量  │  │ 5% 流量  │  │ 75→100% │  │ 100% 流量    │ │
│  └──────────┘  └──────────┘  └──────────┘  └─────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 评估门控（Eval Gate）——Agent CI/CD 的核心创新

这是 Agent CI/CD 区别于传统 Pipeline 最关键的环节：

```python
# eval_gate.py — CI/CD 评估门控脚本
import json
import sys
from dataclasses import dataclass

@dataclass
class EvalGateResult:
    passed: bool
    quality_score: float
    regression_detected: bool
    cost_per_task: float
    details: dict

class EvalGate:
    """CI/CD Pipeline 中的评估门控"""

    def __init__(self, config: dict):
        self.quality_threshold = config["quality_threshold"]     # 例: 0.85
        self.regression_tolerance = config["regression_tolerance"] # 例: 0.03
        self.cost_budget = config["cost_budget_per_task"]         # 例: 1.50
        self.baseline_scores = self._load_baseline()

    def run(self, agent_under_test, eval_datasets: dict) -> EvalGateResult:
        results = {}

        # 1. 黄金测试集：核心功能质量
        golden_score = self._run_golden_eval(agent_under_test, eval_datasets["golden"])
        results["golden_score"] = golden_score

        # 2. 回归测试集：对比基线检测退化
        regression = self._run_regression_eval(agent_under_test, eval_datasets["regression"])
        results["regression"] = regression

        # 3. 对抗测试集：安全性验证
        safety_score = self._run_adversarial_eval(agent_under_test, eval_datasets["adversarial"])
        results["safety_score"] = safety_score

        # 4. 成本评估：Token 消耗是否在预算内
        cost = self._estimate_cost(agent_under_test, eval_datasets["golden"])
        results["cost_per_task"] = cost

        # 综合判定
        quality_passed = golden_score >= self.quality_threshold
        no_regression = not regression["has_regression"]
        cost_ok = cost <= self.cost_budget
        safety_ok = safety_score >= 0.95

        passed = quality_passed and no_regression and cost_ok and safety_ok

        return EvalGateResult(
            passed=passed,
            quality_score=golden_score,
            regression_detected=regression["has_regression"],
            cost_per_task=cost,
            details=results,
        )

    def _run_regression_eval(self, agent, dataset) -> dict:
        """对比基线，检测行为退化"""
        current_scores = self._evaluate(agent, dataset)
        baseline = self.baseline_scores

        regressions = []
        for category, score in current_scores.items():
            baseline_score = baseline.get(category, 0)
            if baseline_score - score > self.regression_tolerance:
                regressions.append({
                    "category": category,
                    "baseline": baseline_score,
                    "current": score,
                    "delta": score - baseline_score,
                })

        return {
            "has_regression": len(regressions) > 0,
            "regressions": regressions,
            "current_scores": current_scores,
        }

if __name__ == "__main__":
    gate = EvalGate(config=load_config("eval_gate_config.yaml"))
    result = gate.run(agent_under_test=build_agent(), eval_datasets=load_eval_datasets())

    if not result.passed:
        print(f"❌ Eval Gate FAILED: {json.dumps(result.details, indent=2)}")
        sys.exit(1)
    else:
        print(f"✅ Eval Gate PASSED: quality={result.quality_score:.2f}, cost=${result.cost_per_task:.2f}")
        sys.exit(0)
```

#### GitHub Actions 集成示例

```yaml
# .github/workflows/agent-ci.yml
name: Agent CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
    paths:
      - 'src/**'
      - 'prompts/**'
      - 'configs/**'
      - 'evals/**'

jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install ruff mypy
      - run: ruff check src/
      - run: mypy src/

  prompt-safety-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Scan prompts for injection vulnerabilities
        run: python scripts/scan_prompts.py prompts/

  eval-gate:
    needs: [lint-and-typecheck, prompt-safety-scan]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run Eval Gate
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: python eval_gate.py --config configs/eval_gate_config.yaml

      - name: Upload eval results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: eval-results
          path: eval_results/

  deploy-canary:
    needs: [eval-gate]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to canary (5% traffic)
        run: |
          python scripts/deploy.py \
            --environment canary \
            --traffic-percentage 5 \
            --manifest agent_manifest.yaml

  monitor-canary:
    needs: [deploy-canary]
    runs-on: ubuntu-latest
    steps:
      - name: Monitor canary for 30 minutes
        run: |
          python scripts/monitor_canary.py \
            --duration 1800 \
            --quality-threshold 0.85 \
            --cost-threshold 1.50

  deploy-production:
    needs: [monitor-canary]
    runs-on: ubuntu-latest
    steps:
      - name: Progressive rollout to production
        run: |
          python scripts/deploy.py \
            --environment production \
            --strategy progressive \
            --stages "25,50,75,100" \
            --stage-interval 600
```

### Prompt 变更独立触发评估

一个重要的 CI/CD 设计原则：**Prompt 变更应该触发评估但不需要重新构建应用**。

```yaml
# 当仅 Prompt 文件变更时，跳过构建，直接评估
on:
  push:
    paths:
      - 'prompts/**'

jobs:
  prompt-only-eval:
    steps:
      - name: Load current deployed agent
        run: python scripts/load_agent.py --from-deployment production

      - name: Apply prompt changes
        run: python scripts/apply_prompts.py --source prompts/

      - name: Run evaluation
        run: python eval_gate.py --mode prompt-only
```

## 五、测试策略：从断言到语义评估

### 传统测试 vs Agent 测试

```python
# 传统后端测试 —— 确定性断言
def test_calculate_total():
    result = calculate_total([10, 20, 30])
    assert result == 60  # 精确匹配

# Agent 测试 —— 语义评估（无法精确匹配）
def test_agent_refund_handling():
    result = agent.run("我买的手机屏幕碎了，想退货")

    # ❌ 这样不行——Agent 的措辞每次都不同
    # assert result == "好的，我来帮您处理退货..."

    # ✅ 语义评估：检查意图和关键要素
    eval_result = semantic_evaluate(
        output=result,
        criteria={
            "acknowledged_issue": "是否确认了用户的问题",
            "offered_solution": "是否提供了退货/退款方案",
            "asked_order_info": "是否询问了订单信息",
            "tone_appropriate": "语气是否专业友好",
            "no_hallucination": "是否未编造不存在的政策",
        }
    )
    assert eval_result.overall_score >= 0.85
```

### Agent 测试金字塔

```
                    /\
                   /  \      生产监控（Production Monitoring）
                  /    \     - 实时质量评分
                 /      \    - 用户反馈采集
                /--------\   - 漂移检测
               /          \
              /   端到端    \  端到端场景测试（E2E Eval）
             /   场景测试   \  - 完整任务流测试
            /              \  - 多轮对话场景
           /----------------\  - 30-100 个黄金用例
          /                  \
         /    集成评估        \  集成评估（Integration Eval）
        /                    \  - 工具调用正确性
       /                      \  - RAG 检索质量
      /------------------------\  - 多 Agent 协作
     /                          \
    /       单元评估              \ 单元评估（Unit Eval）
   /                              \ - 单个 Prompt 质量
  /                                \ - 单个工具调用验证
 /                                  \ - 输出格式合规
/------------------------------------\
```

### AgentAssay：非确定性系统的回归测试

2026 年 3 月发布的 AgentAssay 框架（arXiv: 2603.02601）专门解决了 Agent 回归测试的核心难题：

**核心创新：**

| 特性 | 传统回归测试 | AgentAssay |
|------|------------|-----------|
| 判定结果 | 二元（PASS/FAIL） | 三值（PASS/FAIL/**INCONCLUSIVE**） |
| 行为表征 | 字符串匹配 | **行为指纹（Behavioral Fingerprint）**：执行轨迹→紧凑向量 |
| 资源消耗 | 固定 | **自适应预算**：根据置信度动态调整试验次数 |
| 检测能力 | 确定性回归 | 概率性回归（传统方法检测率 0% 的场景达到 86%） |

**关键思想：** 由于 Agent 行为天然非确定性，单次运行结果无法判断是否存在回归。AgentAssay 使用序贯概率比检验（Sequential Probability Ratio Test），在统计显著性和 Token 成本之间取得平衡——试验次数减少 78%，同时保持统计保证。

### 评估数据集管理

```yaml
# evals/dataset_manifest.yaml
datasets:
  golden:
    description: "黄金测试集：核心场景覆盖"
    file: "golden_set.jsonl"
    size: 130
    categories:
      - name: "订单查询"
        count: 30
      - name: "退款处理"
        count: 25
      - name: "产品咨询"
        count: 35
      - name: "投诉处理"
        count: 20
      - name: "边界情况"
        count: 20
    update_policy: "每月审查，由领域专家维护"

  regression:
    description: "回归测试集：历史 Bug 场景"
    file: "regression_set.jsonl"
    size: 250
    source: "生产 Bug 记录 + 用户反馈"
    update_policy: "每个 Bug 修复后追加对应用例"

  adversarial:
    description: "对抗测试集：安全与鲁棒性"
    file: "adversarial_set.jsonl"
    size: 50
    categories:
      - "Prompt 注入尝试"
      - "角色扮演攻击"
      - "信息窃取尝试"
      - "越权操作尝试"
    update_policy: "红队测试后更新"
```

**数据集规模建议：**

| 阶段 | 黄金集大小 | 回归集大小 | 对抗集大小 |
|------|-----------|-----------|-----------|
| MVP / 早期 | 30-50 | 50-100 | 20-30 |
| 成长期 | 100-200 | 200-500 | 50-100 |
| 成熟期 | 300-500 | 500-1000 | 100-200 |

## 六、部署与发布策略

### 为什么 Agent 不能直接蓝绿切换？

传统蓝绿部署的假设是**无状态**——切换流量即切换版本。但 Agent 是有状态的：

**进行中的会话（Conversation State）：**
- 多轮对话上下文
- 当前任务的中间状态

**长期记忆（Long-term Memory）：**
- 用户偏好
- 历史交互摘要

**工具执行副作用（Side Effects）：**
- 已发送的邮件
- 已修改的数据库记录
- 已触发的外部操作

**向量索引（Vector Index）：**
- RAG 知识库版本
- 嵌入模型版本对应关系

**直接切换的风险：**
- 进行中的会话中断，用户需要重新开始
- 新版本 Agent 读取旧版本记忆格式可能不兼容
- 已完成一半的工作流无法在新版本中恢复
- 向量索引版本不匹配导致检索质量骤降

### 推荐：渐进式交付（Progressive Delivery）

```
阶段 1: Shadow（影子模式）—— 0% 用户流量
┌───────────────────────────────────────┐
│ 生产流量 ──→ V1 (当前版本) ──→ 响应用户 │
│       └──→ V2 (新版本) ──→ 仅记录结果  │
│                                       │
│ 对比 V1 vs V2 的输出差异               │
│ 无风险地检测行为偏差                    │
└───────────────────────────────────────┘
        │ 行为偏差可接受
        ↓
阶段 2: Canary（金丝雀）—— 5% 用户流量
┌───────────────────────────────────────┐
│ 95% 流量 ──→ V1                       │
│  5% 流量 ──→ V2                       │
│                                       │
│ 实时对比 V1 vs V2 的质量、成本、延迟    │
│ 异常则自动回滚                         │
└───────────────────────────────────────┘
        │ 指标正常 × 30分钟
        ↓
阶段 3: Progressive（渐进放量）—— 25→50→75→100%
┌───────────────────────────────────────┐
│ 每个阶段持续 10-15 分钟               │
│ 每个阶段检查：                        │
│ - 质量分 ≥ 基线 - 3%                  │
│ - 成本 ≤ 预算 × 1.2                  │
│ - 无安全事件                          │
│ 任一检查失败则暂停并回滚               │
└───────────────────────────────────────┘
        │ 全部通过
        ↓
阶段 4: Full Rollout —— 100% 流量
```

### "前滚"而非"回滚"（Roll Forward, Not Back）

对于具有以下特征的 Agent，**应该前滚修复而不是回滚**：

| 场景 | 原因 |
|------|------|
| 长期记忆已被新版本修改 | 回滚代码但记忆格式不兼容 |
| 已产生外部副作用（邮件、交易） | 回滚会导致重复操作 |
| 向量索引已更新 | 旧版本代码无法正确使用新索引 |
| 下游系统已接收新版本的输出 | 回滚可能导致数据不一致 |

```python
class RollbackDecisionEngine:
    """决定回滚还是前滚"""

    def decide(self, incident: dict) -> str:
        has_memory_mutations = incident.get("memory_mutated", False)
        has_side_effects = incident.get("external_side_effects", False)
        has_index_changes = incident.get("vector_index_updated", False)
        severity = incident.get("severity", "low")

        if severity == "critical" and not has_side_effects:
            return "ROLLBACK"

        if has_memory_mutations or has_side_effects or has_index_changes:
            return "ROLL_FORWARD"

        return "ROLLBACK"
```

## 七、依赖管理——Agent 的隐藏地雷

### 模型版本漂移

这是传统后端不存在的问题：**你的代码没变，但模型提供商更新了模型，行为就变了。**

举一个真实场景：
- 2026-01-15：使用 `gpt-4o-2024-11-20`，一切正常
- 2026-02-01：OpenAI 默认切换到 `gpt-4o-2025-02-01`，用户投诉增加
- 2026-02-03：排查发现是模型版本变化导致 Tool Calling 格式微变

**防御措施：**

```yaml
# ❌ 危险：使用别名，会被自动升级
model: "gpt-4o"  # 随时可能指向不同版本

# ✅ 安全：精确到日期版本
model: "gpt-4o-2024-11-20"

# ✅ 更安全：在 manifest 中集中管理，并有升级计划
model:
  name: "gpt-4o-2024-11-20"
  upgrade_reviewed: false  # 标记是否已评估新版本
  next_review_date: "2026-04-01"
```

### 依赖锁定清单

Agent 需要锁定的依赖（传统后端不需要关心的部分）：

- 模型版本（精确到日期）
- 嵌入模型版本（影响向量索引兼容性）
- LLM SDK 版本（langchain, openai, anthropic 包版本）
- 工具 Schema 版本（JSON Schema hash）
- MCP 服务器版本
- 向量数据库客户端版本
- Prompt 文件版本（Git commit hash）
- 评估数据集版本
- Guardrails 模型/规则版本

### Docker 镜像固定

```dockerfile
# ❌ 危险：latest 标签
FROM python:latest

# ✅ 安全：精确版本 + digest
FROM python:3.12.3-slim@sha256:abc123...

# 依赖锁定
COPY requirements.lock ./
RUN pip install --no-deps -r requirements.lock

# Prompt 和配置作为构建层
COPY prompts/ /app/prompts/
COPY configs/ /app/configs/
COPY agent_manifest.yaml /app/
```

## 八、敏捷迭代节奏：Agent 的 Sprint 怎么做？

### 传统 Sprint vs Agent Sprint

| 活动 | 传统 Sprint | Agent Sprint |
|------|------------|-------------|
| 需求定义 | 用户故事 + 验收标准 | 用户故事 + **评估用例 + 质量阈值** |
| 开发 | 写代码 | 写代码 + **Prompt 工程 + 工具集成** |
| 测试 | 单元测试 + 集成测试 | 评估数据集 + **Eval Gate + 人工评审** |
| Code Review | 审查代码 | 审查代码 + **审查 Prompt 变更 + 审查模型配置** |
| 发布 | 部署上线 | Shadow → Canary → **渐进放量** |
| 监控 | 错误率、延迟 | 错误率、延迟 + **质量分、Token 成本、漂移检测** |
| 回顾 | 效率、质量 | 效率、质量 + **Eval 覆盖率、误报率、成本趋势** |

### 推荐的迭代节奏

**Week 1：开发与内部评估**
- Day 1-2：Prompt 实验 + 本地评估
- Day 3-4：代码开发 + 工具集成
- Day 5：PR Review（代码 + Prompt + 配置 联合审查）

**Week 2：评估与发布**
- Day 1：CI 评估通过 + 修复回归
- Day 2：Shadow 部署 + 行为对比分析
- Day 3：Canary 发布（5%）+ 实时监控
- Day 4：渐进放量至 100%
- Day 5：生产监控确认 + Sprint 回顾

### 新增角色：Prompt 审查者

在传统的 Code Review 之外，Agent 团队需要 **Prompt Review**。这不是同一件事：

**Code Review 关注：**
- 逻辑正确性
- 性能
- 安全漏洞
- 代码规范

**Prompt Review 关注：**
- 指令清晰度和无歧义性
- Few-shot 示例覆盖率
- 注入攻击抵抗力
- 与模型版本的兼容性
- Token 效率
- 输出格式约束的完整性
- 边界情况处理指引

## 九、企业落地的常见陷阱与应对

### 陷阱 1：把 Agent 当传统微服务管理

**症状**：只版本化代码，忽略 Prompt 和模型版本。生产出问题后无法复现，因为本地环境的模型版本已经更新了。

**应对**：引入 Agent Manifest，集中声明所有四层版本。任何变更都必须更新 Manifest 并触发评估。

### 陷阱 2：用传统断言测试 Agent

**症状**：写了大量 `assert response.text == "..."` 的测试，全部因为 Agent 措辞不同而失败。团队放弃了自动化测试。

**应对**：转向语义评估。使用 LLM-as-Judge、DeepEval 等框架评估意图和质量，而非精确匹配。

### 陷阱 3：Prompt 变更不经 Review 直接上生产

**症状**：产品经理直接在平台上修改了 Prompt，导致质量骤降。没有变更记录，无法回溯。

**应对**：Git 作为 Prompt 的 Source of Truth，所有变更必须经过 PR Review + Eval Gate。

### 陷阱 4：没有成本门控，Agent 烧穿预算

**症状**：某个边界场景导致 Agent 陷入无限重试循环，一夜之间 API 费用翻了 10 倍。

**应对**：CI/CD Pipeline 中加入成本评估。生产环境设置循环上限、Token 上限和费用告警。

### 陷阱 5：模型提供商升级导致"无声故障"

**症状**：OpenAI/Anthropic 更新了模型默认版本。HTTP 状态码都是 200，但 Agent 的工具调用格式微变，导致 30% 的任务静默失败。

**应对**：精确锁定模型版本到日期。建立模型版本升级的专项评估流程。订阅提供商的变更通知。

### 陷阱 6：回滚了代码但没回滚记忆

**症状**：回滚到上一版本后，Agent 读取了新版本写入的记忆格式，导致更严重的错误。

**应对**：建立 Roll Forward 优先的策略。如果必须回滚，确保记忆层的兼容性或同步清理。

## 十、总结：Agent 交付的核心原则

1. **版本化一切**：不只是代码，还有 Prompt、模型、工具、知识
2. **评估即测试**：用语义评估替代精确断言，Eval Gate 是 CI 核心
3. **渐进式发布**：Shadow → Canary → Progressive → Full
4. **锁定依赖**：精确到模型日期版本、Docker digest、SDK 版本
5. **成本是指标**：CI 中加入成本门控，生产中设置预算告警
6. **Prompt 即代码**：Git 管理、PR 审查、语义化版本、变更日志
7. **前滚优于回滚**：有状态系统慎用回滚，优先前滚修复
8. **持续监控**：行为漂移是渐进的，不是瞬间的，需要持续检测

**最后一条忠告：Agent 开发不是一次性的软件交付，而是一个持续的"行为工程"过程。你交付的不是代码，而是一个不断演化的决策系统。用管理活的系统的心态去管理它。**

---

## 参考资源

- [Zylos Research - AI Agent Version Management: Safe Upgrade Patterns](https://zylos.ai/research/2026-03-06-ai-agent-version-management-safe-upgrade-patterns)
- [Supergood Solutions - Prompt Version Control: Treat Your Prompts Like Production Code](https://supergood.solutions/blog/systems-sunday-prompt-version-control-2026/)
- [Progressive Delivery for Agents: Shadow Tests, Eval Gates, and Fast Rollbacks](https://appropri8-astro.pages.dev/blog/2026/01/30/progressive-delivery-agents-shadow-canary-rollback/)
- [AgentAssay: Token-Efficient Regression Testing (arXiv: 2603.02601)](https://arxiv.org/abs/2603.02601)
- [Auxiliobits - Versioning & Rollbacks in Modern Agent Deployments](https://www.auxiliobits.com/blog/versioning-and-rollbacks-in-agent-deployments/)
- [Braintrust - What is Prompt Management?](https://www.braintrust.dev/articles/what-is-prompt-management)
- [LLMOps Complete Guide 2026 - Calmops](https://calmops.com/ai/llmops-complete-guide-2026/)
- [Microsoft - LLMOps: Operational Management of LLMs](https://learn.microsoft.com/en-us/ai/playbook/technology-guidance/generative-ai/mlops-in-openai/)
