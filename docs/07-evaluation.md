# 第 7 章 · LLM 评估

## 7.1 这一步要解决什么问题

Agent 上线之后,怎么知道它回答得好不好?人工一条条看对话记录不现实,这一章用 **LLM 当裁判**(另一个模型给回答打分)的方式,把"质量评估"变成一件可以自动化、可以量化追踪趋势的事情。

## 7.2 在 GCP 上具体怎么做

```bash
pip install "google-cloud-aiplatform[evaluation]"
```

自定义评分标准(比如"回答有没有准确反映政策原文"):

```python
from vertexai.evaluation import PointwiseMetric, PointwiseMetricPromptTemplate

policy_accuracy_metric = PointwiseMetric(
    metric="policy_accuracy",
    metric_prompt_template=PointwiseMetricPromptTemplate(
        criteria={
            "policy_accuracy": "回答是否正确反映了reference给出的政策,不能编造或矛盾",
        },
        rating_rubric={"5": "完全准确", "3": "部分准确", "1": "错误或编造"},
        input_variables=["prompt", "reference"],
    ),
)
```

跑评估:

```python
import pandas as pd
from vertexai.evaluation import EvalTask

eval_dataset = pd.DataFrame({
    "prompt": [...],       # 真实用户问题
    "response": [...],     # agent的真实回答(从部署好的remote agent收集)
    "reference": [...],    # 对应的政策原文
})

eval_task = EvalTask(dataset=eval_dataset, metrics=[policy_accuracy_metric])
eval_result = eval_task.evaluate()

print(eval_result.summary_metrics)     # 各项平均分
print(eval_result.metrics_table)       # 每条对话的具体得分+裁判解释
```

## 7.3 真实运行效果

```text
=== summary_metrics ===
{'row_count': 3, 'policy_accuracy/mean': 5.0, 'question_answering_quality/mean': 4.0}
```

每一条对话都能看到裁判给出的具体解释,比如:

> "The response accurately extracts the return policy (within 30 days) from the reference and correctly applies it to answer the user's question..."

## 7.4 真实踩过的坑:选错内置指标,分数全是 0

一开始想用官方文档提到的 `"helpfulness"` 内置指标,直接报错 `KeyError: 'helpfulness'` —— 当前 SDK 版本里这个名字已经不在支持列表里了,需要通过 `MetricPromptTemplateExamples.list_example_metric_names()` 查当前实际支持哪些。

换成 `groundedness` 之后,三条完全正常的回答**全部得了 0 分**——这不是 agent 有问题,是**指标语义没选对**:

```text
The AI response introduces external information ... which is not present
in the user's prompt. The prompt only asks a question without providing
any context about return policies.
```

`groundedness` 衡量的是"回答有没有超出用户输入本身",这个定义适合"总结一段给定文字"这种场景(摘要不能超出原文瞎编),但**完全不适合 RAG 问答**——RAG 问答的本意就是要从外部知识库引入用户没直接提供的信息。换成 `question_answering_quality` 才是语义匹配的指标。

!!! danger "这是本章最重要的一课"
    评估指标的名字听起来"感觉对"不等于语义真的适合你的场景。用之前一定要看一眼它的评分模板具体在问裁判模型什么问题,不然会把"指标选错了"误判成"agent 真的很差",方向完全错。

## 7.5 Gen AI Evaluation Service 还能做什么

- **`PairwiseMetric`**:不只是单条打分,还能让裁判模型对比两个不同版本的回答(比如换模型前后、改 prompt 前后),给出"哪个更好"的判断,适合做 A/B 对比。
- **`client.evals`(更新的统一接口)**:面向完整多轮对话轨迹的评估(`MULTI_TURN_TASK_SUCCESS` 等指标),而不只是单轮的静态问答对,更贴近真实客服场景的评估需求。
- **裁判模型本身也可以被评估/调优**:如果担心裁判模型的判断不可靠,可以用一批人工标注的"标准答案"去校准裁判模型,评估裁判本身的准确度。

## 7.6 优化方向

- **样本规模**:本章只用了 3 条样本作教学演示,生产场景需要覆盖更全面的问题分布(包括边界case、恶意输入、多轮对话),样本量至少要几十上百条才有统计意义。
- **持续评估流水线**:把评估脚本接入定时任务,定期抽样近期真实对话跑评估,画出质量趋势图,而不是只在上线前跑一次。
- **把评估结果和护栏/RAG检索结果关联起来分析**:比如"哪些低分回答是因为检索没找到相关文档",而不是只看最终回答分数,找到问题的真正环节。
- **结合人工抽检**:LLM 裁判不是万能的(本章已经演示过它自己也会在指标语义不对时给出误导性结果),生产场景应该保留一定比例的人工抽检作为交叉验证。
