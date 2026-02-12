# 评测体系设计与工具指南
> 最后更新：2026-02-05（工具/模型示例可能变化，请按最新资料更新）

## 一、评测设计核心原则
1. **指标业务对齐**: 评测分数必须能反映业务价值。
2. **自动化优先**: 尽可能构建自动化评测流水线。
3. **分层评测**: 单元测试(Unit) -> 整合测试(Integration) -> 端到端测试(E2E)。

---

## 二、评测指标体系 (Metrics)

### 2.1 确定性指标 (Deterministic)
适用于能通过代码直接判断的场景。
- **JSON 格式合法性**: 输出是否为标准 JSON。
- **关键词覆盖**: 是否包含了 "退款"、"受理" 等必要词汇。
- **代码可执行性**: 生成的代码能否编译/运行通过。

### 2.2 基于 Model-based 的指标 (LLM-as-a-Judge)
使用更强的模型（如 GPT-4o）来评价小模型或 Agent 的输出。
- **相关性 (Relevance)**: 回答是否切题？(1-5分)
- **准确性 (Faithfulness)**: 回答是否忠实于检索到的上下文，有无幻觉？
- **助人性 (Helpfulness)**: 回答是否解决了用户问题？

### 2.3 业务指标
- **任务完成率 (Success Rate)**: 对话最终是否达成了意图（如订单创建成功）。
- **人工接管率 (Escalation Rate)**: 用户转人工的比例。

---

## 三、推荐评测工具

### 3.1 Promptfoo (推荐 ⭐)
最流行的 CLI 评测工具，支持多种模型和断言方式。
- **官网**: https://promptfoo.dev/
- **特点**: 配置简单 (YAML)，支持 JS 脚本断言，可视化报告。
- **适用**: Prompt 迭代、回归测试。

**配置文件示例 (`promptfooconfig.yaml`)**:
```yaml
prompts: [prompts.txt]
providers: [openai:gpt-4o, anthropic:claude-3-5-sonnet]
tests:
  - description: "测试客服退款场景"
    vars:
      user_input: "我要退款，订单号是123456"
    assert:
      - type: contains
        value: "退款政策"
      - type: llm-rubric
        value: "回答必须礼貌且询问退款原因"
```

### 3.2 Ragas
专门针对 **RAG (检索增强生成)** 应用的评测框架。
- **官网**: https://docs.ragas.io/
- **核心指标**: Context Precision (检索精准度), Context Recall (检索召回率), Faithfulness (信实度), Answer Relevance (答案相关性)。
- **适用**: 优化 RAG 检索链路和生成质量。

### 3.3 Arize Phoenix / LangSmith
全链路追踪与评测平台。
- **Phoenix**: https://phoenix.arize.com/
- **LangSmith**: https://smith.langchain.com/
- **特点**: 可视化 Trace，不仅看结果，还能看中间步骤（Retrieval 搜到了什么，Tool 传了什么参）。
- **适用**: 复杂 Agent 在线监控与调试。

---

## 四、自建评测脚本示例 (Python)

```python
import json
from openai import OpenAI

client = OpenAI()

def evaluate_answer(question, answer, ground_truth):
    """
    使用 LLM 作为裁判来进行评分
    """
    prompt = f"""
    请作为一名公正的裁判，评估AI助手的回答。
    
    问题: {question}
    标准答案: {ground_truth}
    AI回答: {answer}
    
    请打分 (0-10分) 并简述理由。
    返回格式: {{"score": 8, "reason": "..."}}
    """
    
    response = client.chat.completions.create(
        # 使用当前最强的评审模型（示例）
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}],
        response_format={"type": "json_object"}
    )
    return json.loads(response.choices[0].message.content)

# 批量运行
dataset = [
    {"q": "如何重置密码?", "gt": "点击设置-账号安全-重置密码", "ai": "去设置里找找"},
    # ... 更多数据
]

for item in dataset:
    eval_result = evaluate_answer(item['q'], item['ai'], item['gt'])
    print(f"问题: {item['q']} | 分数: {eval_result['score']} | 理由: {eval_result['reason']}")
```

---

## 五、评测流程最佳实践

1. **建立 "Golden Dataset" (金标准数据集)**:
   - 包含 50-100 条覆盖核心场景的高质量问答对。
   - 每次 Prompt 修改或代码更新，必须跑通该数据集。

2. **从 Bad Case 中学习**:
   - 线上发现的问题，立即转化为评测 Case 加入数据集，防止回归 (Regression)。

3. **分级测试**:
   - 开发时不跑全量（太贵），只跑 10 条冒烟测试。
   - 上线前跑全量测试。
