# 第 7 章 · LLM 评估

## 7.1 这一步要解决什么问题

Agent 上线之后,怎么知道它回答得好不好?

最朴素的做法是人工一条条看对话记录、打分、写反馈。这在 Demo 阶段行得通,但一旦进入生产,每天可能有成百上千条真实对话,人工评估在时间和人力成本上都不现实——更麻烦的是,人工评估的**及时性**也跟不上迭代节奏:你改了一版 prompt,想知道有没有变好,如果要等人工评审排期,反馈周期可能是几天,而不是几分钟。

这一章要解决的问题,是把"这个回答质量怎么样"从一件依赖人工肉眼判断的事情,变成一件可以**自动化、可重复、可以量化追踪趋势**的事情。我们在 Phase 6 采用的方案是业界现在的主流做法——**LLM-as-judge**:用另一个大模型(裁判模型,通常选用能力更强的模型,比如 Gemini 系列的旗舰版本)去阅读 agent 的回答,依据我们给定的评分标准打分,并给出打分理由。

### 为什么 LLM 裁判能替代大规模人工评估

LLM-as-judge 之所以能work,本质上是利用了大模型在自然语言理解和推理上已经具备的、接近人类水平的判断力,把"阅读一段文本、对照标准做判断"这件事交给模型去规模化执行。具体来说它解决了三个人工评估做不到的问题:

- **规模**:裁判模型跑一条评估通常是秒级的 API 调用,理论上可以覆盖任意规模的样本,而不是抽样几十条意思一下。
- **一致性**:同一套评分标准(rubric),裁判模型在不同批次之间的执行是确定的评分逻辑,不会像不同人工评审员那样因为个人偏好、疲劳程度、当天心情而产生系统性漂移(虽然裁判模型有自己的偏差,见下文)。
- **即时反馈**:可以直接接入 CI 或定时任务,在每次改动 prompt、换模型、调整 RAG 检索策略之后几分钟内就能拿到质量分数,把"评估"从上线前的一次性动作变成迭代过程中随时可用的信号。

### 但裁判模型的判断不是绝对客观的真值

这里必须非常明确地说清楚一件事,也是我们在 Phase 6 里付出过代价才理解透的一点:**LLM 裁判给出的分数是一个"代理信号"(proxy signal),不是客观真值(ground truth)**。裁判模型本身也是一个概率性的语言模型,它的判断会受到下面这些已知偏差的影响:

| 偏差类型 | 表现 | 影响 |
|---|---|---|
| 冗长偏好(verbosity bias) | 裁判模型倾向给更长、更"看起来详尽"的回答打高分,即使内容并不更准确 | 会误导你以为"话说得多"就是"答得好" |
| 位置偏差(position bias) | 在 `PairwiseMetric` 这类 A/B 对比评估中,裁判模型对"先出现的那个回答"或"后出现的那个回答"有系统性偏好 | 影响 A/B 对比结论的可信度,通常需要交换顺序跑两遍来抵消 |
| 自我偏好(self-preference bias) | 如果裁判模型和被评估的生成模型是同一个模型家族,裁判可能对"风格像自己"的回答打分更高 | 削弱跨模型对比评估(比如评估换成另一家模型的 agent)的公正性 |
| 评分标准理解漂移 | 同一个 rubric,裁判模型在不同批次、不同版本的模型上,对"5 分"和"4 分"的边界把握可能不完全一致 | 长期趋势对比时如果中途换过裁判模型版本,分数不能直接跨版本比较 |
| 指标语义不匹配 | 选错内置评估指标,裁判模型会严格按照该指标的字面定义打分,即使这个定义根本不适合你的任务(本章 7.4 会详细复盘这个真实踩坑) | 会把"指标选错了"误判成"agent 真的很差" |

!!! note "怎么理解这件事"
    把 LLM 裁判类比成一个非常勤奋、看得很快、但没有终审权的"初审员"更合适。它能帮你把 95% 的常规评估工作自动化,替你筛出明显有问题的样本、追踪总体趋势,但不能完全替代人工的最终把关——尤其是在评分标准本身有争议、或者裁判模型可能存在系统性偏差的场景下。7.6 节会具体讲怎么用人工抽检去交叉验证裁判的可靠性。

## 7.2 在 GCP 上具体怎么做

GCP 这边用的是 Vertex AI 的 **Gen AI Evaluation Service**,在 SDK 里对应 `vertexai.evaluation` 这个子模块。整体流程分四步:环境准备、定义评分标准、准备评估数据集、跑评估任务。

### 7.2.1 环境准备

```bash
pip install "google-cloud-aiplatform[evaluation]"
```

注意这里装的是带 `[evaluation]` extra 的版本,而不是裸的 `google-cloud-aiplatform`——`vertexai.evaluation` 依赖 `pandas` 等一批评估专用的库,不加这个 extra 装出来的包是没有 `vertexai.evaluation` 这个子模块的,import 时会直接报错。

跑评估之前还需要确认几件环境层面的事情:

```bash
# 1. 确认 Vertex AI API 已启用(如果这个项目已经跑过 Agent Engine 部署,通常已经启用过)
gcloud services enable aiplatform.googleapis.com --project=adk-fde-lab

# 2. 评估任务需要一个 staging bucket 存中间产物(比如大数据集场景下的临时文件),
#    这个 bucket 要提前建好,SDK 不会自动帮你创建
gsutil mb -l us-central1 gs://adk-fde-lab-staging

# 3. 本地跑评估脚本需要有效的应用默认凭证,且该身份需要有 aiplatform.user 或更高权限
gcloud auth application-default login
```

`vertexai.init()` 里的三个参数分别对应:

- `PROJECT_ID`:GCP 项目,决定了这次评估调用计入哪个项目的计费和配额。
- `LOCATION`:区域,决定了裁判模型实际在哪个 region 跑推理——`us-central1` 是 Vertex AI 生成式服务支持最全的区域之一,选它基本不用担心某个新指标模板还没在这个 region 上线。
- `STAGING_BUCKET`:上面建好的 bucket,评估任务用它来暂存中间数据。即使是本章这种几条样本的小规模评估,这个参数也是必填的,SDK 不会因为数据量小就跳过这一步。

### 7.2.2 定义评分标准:PointwiseMetric + PointwiseMetricPromptTemplate

评估的核心是"评分标准怎么定义"。Vertex AI 这边把评分标准封装成两层对象:

- `PointwiseMetric`:声明一个指标的名字(这个名字会出现在最终结果的列名里,比如 `policy_accuracy/mean`),以及它用什么模板去打分。
- `PointwiseMetricPromptTemplate`:真正描述"裁判模型看什么、按什么标准打分"的模板对象,由三部分构成——`criteria`(评估维度)、`rating_rubric`(打分刻度)、`input_variables`(除了默认的 response 之外,还需要把数据集里的哪些列一起喂给裁判模型)。

Phase 6 里我们针对"回答是否准确反映公司政策"这个场景,自定义了这样一个指标:

```python
from vertexai.evaluation import PointwiseMetric, PointwiseMetricPromptTemplate

policy_accuracy_metric = PointwiseMetric(
    metric="policy_accuracy",  # 指标名,会出现在 summary_metrics 的 key 里,比如 "policy_accuracy/mean"
    metric_prompt_template=PointwiseMetricPromptTemplate(
        criteria={
            # criteria 是一个 dict,key 是评估维度的名字,value 是这个维度具体要求裁判模型检查什么。
            # 可以定义多个维度,裁判模型会针对每个维度分别理解,再综合给出一个总分。
            "policy_accuracy": "The response correctly reflects the company policy given in the reference, with no invented or contradicted rules.",
            "completeness": "The response addresses every part of the user's question.",
        },
        rating_rubric={
            # rating_rubric 定义打分刻度:每个分数档位对应什么样的回答表现。
            # 这里用的是 1~5 的五档量表,档位描述越具体,裁判模型给出的分数越稳定、越可复现。
            "5": "Fully accurate and complete; matches policy exactly.",
            "4": "Accurate with minor omissions.",
            "3": "Partially accurate; missing details or minor errors.",
            "2": "Largely inaccurate or contradicts policy meaningfully.",
            "1": "Wrong or fabricates policy.",
        },
        # input_variables 告诉框架:除了永远会传入的 response(被评估的 agent 回答)之外,
        # 数据集里的 prompt 列和 reference 列也要作为模板变量一起喂给裁判模型。
        # 这里的 "reference" 就是对应问题的政策原文——没有这一列,裁判模型根本无从判断回答是否"准确反映政策"。
        input_variables=["prompt", "reference"],
    ),
)
```

对于没有特殊业务语义、只是想评估"回答质量好不好"这种通用场景,不需要自己写 criteria 和 rubric,SDK 内置了一批现成的模板,通过 `MetricPromptTemplateExamples` 直接取用:

```python
from vertexai.evaluation import MetricPromptTemplateExamples

qa_quality_metric = PointwiseMetric(
    metric="question_answering_quality",
    metric_prompt_template=MetricPromptTemplateExamples.get_prompt_template("question_answering_quality"),
)
```

### 7.2.3 rating_rubric 到底是怎么"喂"给裁判模型的

这里有个容易被忽略、但对理解评估结果非常关键的细节:`criteria`、`rating_rubric`、`input_variables` 这三样东西,并不是分别独立地起作用,而是被 SDK 拼装成**一整份完整的评分请求(meta-prompt)**,一次性发给裁判模型。裁判模型收到的不是"这是评分标准,这是要评的内容"两次调用,而是**一个包含了标准定义 + 具体样本内容的单一 prompt**。

拼装出来的结构大致相当于(具体措辞以 SDK 当前版本的内部模板为准,这里是为了说明拼装逻辑做的示意还原):

```text
你是一个专业的评估者。请根据下面的标准,对 AI 回答进行打分。

## 评估维度
- policy_accuracy: The response correctly reflects the company policy given in
  the reference, with no invented or contradicted rules.
- completeness: The response addresses every part of the user's question.

## 打分刻度
5 分: Fully accurate and complete; matches policy exactly.
4 分: Accurate with minor omissions.
3 分: Partially accurate; missing details or minor errors.
2 分: Largely inaccurate or contradicts policy meaningfully.
1 分: Wrong or fabricates policy.

## 用户问题(prompt)
{这条样本的 prompt 内容}

## 参考答案(reference)
{这条样本的 reference 内容,即政策原文}

## 待评估的 AI 回答(response)
{agent 实际生成的回答}

请给出 1-5 的分数,并说明理由。
```

理解了这一点,你就能明白为什么 `input_variables` 这个参数如此关键——它决定了**裁判模型实际能看到哪些信息**。如果某个指标的模板要求的变量(比如 `context`)你的数据集里根本没有对应的列,或者列名对不上,裁判模型在打分时就相当于"缺了一部分材料",打出来的分数自然也就不再是你以为它在评估的那件事。7.4 节复盘的真实踩坑,根源正是这里。

### 7.2.4 eval_dataset 的真实来源

本章展示的代码里为了聚焦评估逻辑本身,`eval_dataset` 是手写的几行 DataFrame。但真实项目里,Phase 6 的 `run_eval.py` 并不是手写样本,而是从一份维护好的 `samples.json` 构建出来的:`samples.json` 里预先整理了一批"用户问题 + 对应的标准答案(政策原文摘录)",脚本读取这份文件后,逐条调用已经部署好的 remote agent(也就是前几章部署到 Agent Engine 的那个 agent)拿到真实回答,把 `prompt` / `response` / `reference` 三列拼成最终的 DataFrame 再传给 `EvalTask`。

```python
import json
import pandas as pd

with open("samples.json") as f:
    samples = json.load(f)  # 每条形如 {"prompt": "...", "reference": "..."}

rows = []
for s in samples:
    response = call_remote_agent(s["prompt"])  # 调用部署好的 agent,拿到它的真实回答
    rows.append({"prompt": s["prompt"], "response": response, "reference": s["reference"]})

eval_dataset = pd.DataFrame(rows)
```

!!! tip "为什么要用真实部署的 agent 而不是本地跑一遍逻辑"
    评估的意义在于反映**线上实际会发生的行为**,包括真实的网络延迟、真实的模型版本、真实的 RAG 检索结果。如果为了图方便在本地用另一套简化逻辑生成 response,评估出来的分数和线上用户实际得到的体验可能对不上。

## 7.3 核心代码:完整的 run_eval.py 与真实运行效果

下面是 Phase 6 `run_eval.py` 的核心逻辑(为了教学目的补全了逐行注释,逻辑与真实脚本一致):

```python
import json
import os
import pandas as pd
import vertexai
from vertexai.evaluation import EvalTask, MetricPromptTemplateExamples, PointwiseMetric, PointwiseMetricPromptTemplate

PROJECT_ID = "adk-fde-lab"
LOCATION = "us-central1"
STAGING_BUCKET = "gs://adk-fde-lab-staging"

# 初始化 Vertex AI SDK 的运行上下文。这一步之后,后面所有 vertexai.evaluation 下的调用
# 才知道该往哪个项目、哪个 region 发请求,以及用哪个 bucket 存中间产物。
vertexai.init(project=PROJECT_ID, location=LOCATION, staging_bucket=STAGING_BUCKET)

# 自定义指标一:政策准确度。这是我们针对"客服 agent 回答是否符合公司政策"这个具体业务场景
# 自己设计的评分标准,内置指标里没有能直接覆盖这个语义的。
policy_accuracy_metric = PointwiseMetric(
    metric="policy_accuracy",
    metric_prompt_template=PointwiseMetricPromptTemplate(
        criteria={
            "policy_accuracy": "The response correctly reflects the company policy given in the reference, with no invented or contradicted rules.",
            "completeness": "The response addresses every part of the user's question.",
        },
        rating_rubric={
            "5": "Fully accurate and complete; matches policy exactly.",
            "4": "Accurate with minor omissions.",
            "3": "Partially accurate; missing details or minor errors.",
            "2": "Largely inaccurate or contradicts policy meaningfully.",
            "1": "Wrong or fabricates policy.",
        },
        input_variables=["prompt", "reference"],
    ),
)

# 自定义指标二:直接复用 SDK 内置的通用问答质量模板,不用自己写 criteria/rubric。
# 这个模板评估的是更泛化的"这是不是一个称职的回答"(相关性、有用性、清晰度等),
# 作为 policy_accuracy 之外的一个通用质量兜底信号。
qa_quality_metric = PointwiseMetric(
    metric="question_answering_quality",
    metric_prompt_template=MetricPromptTemplateExamples.get_prompt_template("question_answering_quality"),
)

# eval_dataset 在真实脚本里从 samples.json 构建(见 7.2.4),这里假定已经拿到了
# 包含 prompt / response / reference 三列的 DataFrame。

# EvalTask 把"要评估的数据集"和"用什么指标评估"绑定在一起。
# experiment 参数会把这次评估记录成 Vertex AI Experiments 里的一次 run,
# 同名 experiment 下多次 evaluate() 调用会自然形成一条可比较的历史记录(这一点在 7.6 的
# 持续评估流水线里会用到)。
eval_task = EvalTask(dataset=eval_dataset, metrics=[policy_accuracy_metric, qa_quality_metric], experiment="agent-policy-eval")

# 真正触发评估:对数据集里的每一行,针对每个 metric 分别调用裁判模型打分。
eval_result = eval_task.evaluate()

print(eval_result.summary_metrics)     # 全量样本每个指标的汇总统计(均值等)
print(eval_result.metrics_table)       # 每一条样本在每个指标上的具体分数 + 裁判模型给出的打分理由
```

三条样本(真实公司政策问答场景)跑完之后的输出:

```text
=== summary_metrics ===
{'row_count': 3, 'policy_accuracy/mean': 5.0, 'question_answering_quality/mean': 4.0}
```

`metrics_table` 里每一条对话都能看到裁判给出的具体打分理由,比如:

```text
"The response accurately extracts the return policy (within 30 days) from the
reference and correctly applies it to answer the user's question..."
```

这条解释本身就是一个很实用的调试信息——它不只是给你一个数字,还告诉你裁判模型"看懂了什么、依据什么判的分",出现异常分数时,第一反应应该是去看这段解释文本,而不是直接怀疑 agent 本身(这正是下一节要复盘的真实教训)。

## 7.4 真实踩过的坑:选错内置指标,分数全是 0

### 第一次报错:指标名字本身就不存在

一开始想省事,直接用官方文档里提到的 `"helpfulness"` 内置指标,结果程序直接抛出:

```text
KeyError: 'helpfulness'
```

排查后发现,当前 SDK 版本支持的内置指标名单和文档描述的对不上——`MetricPromptTemplateExamples` 背后维护的是一份会随 SDK 版本变化的指标列表,不能凭文档里出现过这个名字就认定它在当前环境里一定可用。解决方法是先查一下当前版本实际支持哪些:

```python
from vertexai.evaluation import MetricPromptTemplateExamples

print(MetricPromptTemplateExamples.list_example_metric_names())
```

!!! warning "文档和 SDK 版本可能不同步"
    这是个很值得记住的小教训:遇到内置枚举类的 `KeyError`,第一反应不是怀疑自己代码写错了,而是先用类似 `list_example_metric_names()` 这种自省方法,把当前环境实际支持的选项打印出来看一眼,再决定用哪个名字。

### 第二次翻车:换成 groundedness,三条正确回答全部得 0 分

查到列表后换成了 `groundedness`,结果更离谱:三条我们**已经人工核对过、回答内容完全正确**的对话,`groundedness` 指标全部打了 0 分。第一反应当然是怀疑 agent 出了什么严重问题,但打开 `metrics_table` 看裁判模型给出的具体解释,内容是这样的:

```text
The AI response introduces external information ... which is not present
in the user's prompt. The prompt only asks a question without providing
any context about return policies.
```

翻译过来就是:裁判模型认为这个回答"引入了 prompt 里没有的外部信息"。可这三条回答恰恰是**正确地**从知识库里查到了退货政策(30 天内可退)并据此回答用户——这本来就是 RAG 系统设计的初衷,凭什么反而被判定为"引入了不该有的信息"而扣到 0 分?

### 深挖:groundedness 到底在评估什么,以及为什么它不适合 RAG 问答

顺着这条错误解释往回想,能定位到问题的根源:`groundedness` 这个指标的评估逻辑,是拿**回答**去对照**用户在 prompt 里提供的信息**,检查回答有没有超出 prompt 本身包含的内容。换句话说,它默认的假设是——"回答里出现的每一件事,都应该能在用户自己的问题(或者显式传入的上下文)里找到出处"。

我们当时传给 `EvalTask` 的数据集里,退货政策原文是放在 `reference` 列里的,而不是 `groundedness` 模板期望的那个"上下文"变量位。也就是说,裁判模型在做 groundedness 判断时,手里能看到的只有 `prompt`(用户的原始问题,一句"你们的退货政策是什么")和 `response`(agent 给出的具体政策条款),完全看不到 `reference` 里那份真正支撑这个回答的政策原文。站在裁判模型的角度,它看到的就是:用户只问了一句话、没提供任何政策细节,回答里却凭空冒出了"30 天内可退"这么具体的规则——那可不就是"引入了 prompt 里没有的外部信息"吗?这个判断在 `groundedness` 自己的评估语义下其实是"逻辑自洽"的,只是这个语义从一开始就不该用在这里。

再往本质上说清楚:`groundedness` 这类指标设计的初衷,是给**摘要生成、内容改写**这一类任务用的——比如"把这篇长文章总结成三句话",评估目标是"总结里的每一句话都不能凭空编造,必须能在原文里找到依据"。这种任务里,"不能超出给定文本"恰恰是**质量要求本身**。

而 RAG 问答的评估目标是反过来的:用户的原始问题往往就是一句简短的自然语言提问,**根本不包含**支撑答案所需要的信息;agent 存在的意义,正是要从外部知识库里检索并引入用户没有直接提供的信息,再据此作答。对 RAG 问答场景,"回答有没有超出用户输入"这件事不但不是负面信号,反而是**系统正常工作的证明**。真正该问的问题是"回答有没有正确地反映了检索到的知识",这和 `groundedness` 拿"用户输入"当基准的逻辑,根本不是一回事。

换成语义匹配的 `question_answering_quality`(以及我们自定义的 `policy_accuracy`,它把 `reference` 显式作为对照基准传入)之后,三条回答才拿到了符合预期的高分。

| 指标 | 对照基准是什么 | 适合的任务类型 | 用在 RAG 问答上会怎样 |
|---|---|---|---|
| `groundedness` | 用户在 prompt(或显式 context 变量)里提供的信息 | 摘要、改写、"不能超出原文瞎编"类任务 | 系统性误判——正确利用外部知识的回答会被当成"编造" |
| `question_answering_quality` | 内置的通用问答质量标准(相关性、正确性、有用性等) | 通用问答质量兜底评估 | 可用,但不针对具体业务政策做校验 |
| `policy_accuracy`(自定义) | 显式传入的 `reference`(政策原文) | 需要对照特定知识来源做准确性核验的场景 | 正合适——这正是我们设计它的目的 |

!!! danger "这是本章最重要的一课"
    评估指标的名字听起来"感觉对"不等于语义真的适合你的场景。用之前一定要看一眼它的评分模板具体在向裁判模型提出什么问题、期望从数据集里拿到哪些变量,不然会把"指标选错了"误判成"agent 真的很差",排查方向完全走偏——我们当时如果没有去读 `metrics_table` 里的具体解释文本,很可能会直接开始怀疑 RAG 检索或者 prompt 出了问题,浪费大量排查时间。

## 7.5 Gen AI Evaluation Service 还能做什么

除了本章用到的单点打分(pointwise),Gen AI Evaluation Service 面向的是更完整的评估场景,以下几项是延伸能力:

**1. `PairwiseMetric`:让裁判模型直接对比两个回答,做 A/B 判断**

不是对每条回答单独打分,而是把两个候选回答放在一起给裁判模型看,直接问"哪个更好"。这在做版本对比时比"各自打分再比较均值"更可靠,因为裁判模型是在同一个上下文里做相对判断,而不是两次独立的绝对打分再拼接。典型场景是:换了底层模型之后,想知道新版本是不是真的比旧版本好;或者改了一版 prompt,想验证有没有退步。

```python
# 概念性示意代码,用于说明用法思路,字段名以当前 SDK 版本文档为准
from vertexai.evaluation import EvalTask, PairwiseMetric, PairwiseMetricPromptTemplate

pairwise_policy_metric = PairwiseMetric(
    metric="policy_accuracy_pairwise",
    metric_prompt_template=PairwiseMetricPromptTemplate(
        criteria={
            "policy_accuracy": "Which response more accurately and completely reflects the company policy in the reference?",
        },
        input_variables=["prompt", "reference"],
    ),
)

# dataset 里除了 prompt / reference,还需要两列分别放"候选回答"和"基线回答"
# 比如:response 是新版 agent 的回答,baseline_model_response 是旧版 agent 的回答
ab_dataset = pd.DataFrame({
    "prompt": [...],
    "reference": [...],
    "response": [...],                  # 新版本 agent 的回答
    "baseline_model_response": [...],   # 旧版本 agent 的回答,作为对比基准
})

pairwise_eval_task = EvalTask(dataset=ab_dataset, metrics=[pairwise_policy_metric])
pairwise_result = pairwise_eval_task.evaluate()
# 结果里会给出"候选回答获胜/基线获胜/打平"的统计,以及每条对比的具体理由
```

!!! warning "别忘了位置偏差"
    7.1 节提到过,`PairwiseMetric` 容易受"先出现的回答更容易被偏好"这种位置偏差影响。如果这个对比结论要拿去做重要的版本发布决策,建议把两个回答的顺序交换后再跑一遍,看结论是否稳定,而不要只信一次的结果。

**2. `client.evals`(更新的统一接口):面向完整多轮对话轨迹的评估**

本章用的 `EvalTask` 面向的是"一问一答"这种静态、单轮的评估单元。但真实客服场景往往是多轮对话——用户先问一个模糊的问题,agent 追问澄清,用户补充信息,agent 再给出最终答案,中间可能还夹杂了工具调用。官方文档提到,更新的统一 Gen AI SDK 里的 `evals` 接口是面向这种完整对话轨迹(trajectory)去设计评估指标的,能够评估的不只是"最后一句回答好不好",还包括整个多轮交互过程是不是合理地走向了任务完成(比如有没有绕圈子、有没有该追问的时候没追问、工具调用顺序是否合理)。这比单轮问答评估更贴近真实场景里用户实际体验到的东西,值得在评估体系成熟之后作为下一步演进方向。

**3. 裁判模型本身也需要被校准**

如果担心裁判模型的判断本身不可靠,可以反过来对裁判模型做一次"元评估":找一批有人工标注好的"标准答案"(即人和裁判模型都对同一批样本打分),对比裁判模型的打分和人工标注的一致程度,来评估这个裁判本身在你的具体业务场景下靠不靠谱。如果一致率不理想,通常意味着 rubric 描述得不够具体、或者需要在评分模板里补充一些打分示例(few-shot),而不是简单地怀疑"LLM 裁判不靠谱"就放弃这套方法。7.6 节会具体展开这个校准流程怎么落地。

**4. 针对不同业务维度自定义指标,复用同一套模板模式**

`policy_accuracy` 只是我们针对"政策准确性"这一个维度设计的自定义指标,同样的 `PointwiseMetric` + `PointwiseMetricPromptTemplate` 模式可以用来评估任何你关心的业务维度——比如语气是否符合品牌调性、有没有意外泄露内部信息或 PII、回答是否遵守了某个法规限制。criteria 和 rating_rubric 换成对应的描述即可,不需要额外学新的 API。

**5. 接入自动化流程,作为发布前的回归测试**

官方文档提到,评估任务可以整合进自动化的持续集成/持续部署流程,在每次模型、prompt 或检索策略变更之后自动跑一遍固定的回归数据集,把"这次改动有没有让质量下降"变成发布流程里的一道硬性检查,而不是完全依赖人工事后回溯。

## 7.6 优化方向

**样本规模**:本章为了教学演示只用了 3 条样本,生产场景需要覆盖更全面的问题分布,包括边界 case、恶意或对抗性输入、多轮对话场景,样本量至少要到几十上百条、并且按问题类型分层抽样,才能得出有统计意义的结论,而不是被少数几条样本的偶然结果带偏。

**持续评估流水线**:不要只在上线前跑一次评估,而是把它做成一条常态化运行的流水线:

1. 用 Cloud Scheduler 定时(比如每天或每周)触发一个 Cloud Run job 或 Cloud Function。
2. 这个 job 从生产环境的对话日志(比如导出到 BigQuery 的会话记录)里,随机或按问题类别分层抽样最近一批真实对话。
3. 组装成 `eval_dataset`(有明确标准答案的问题可以直接匹配 `reference`;没有标准答案的问题可以先只跑不依赖 reference 的通用质量指标),复用 7.3 节的 `run_eval.py` 逻辑跑一遍评估。
4. 因为 `EvalTask` 的 `experiment` 参数会把每次 `evaluate()` 记录成同一个 Vertex AI Experiment 下的一条 run,所以只要 experiment 名字保持不变,历史上每一次评估结果自然就会在 Vertex AI Experiments 里积累成一条可比较的序列。在此基础上,把每次的 `summary_metrics`(连同时间戳)额外写一份到 BigQuery 表里,再用 Looker Studio 这类工具把 `policy_accuracy/mean`、`question_answering_quality/mean` 这些指标画成时间序列趋势图,持续盯着质量有没有随着某次改动悄悄退化。
5. 有条件的话,给关键指标设置 Cloud Monitoring 告警阈值——比如连续两次评估的均分低于某个下限就触发通知,把"质量退化"从"某天偶然发现"变成"系统主动报警"。

**把评估结果和护栏/RAG 检索结果关联起来分析**:只看最终回答的分数,定位不到问题出在哪个环节。更有效的做法是在真实对话链路里,把每次 RAG 检索到的 context 片段、护栏(guardrail)的拦截/改写决策,都用同一个请求 ID 记录下来。当评估发现某条回答分数很低时,不要止步于"这条回答不合格",而是回头用请求 ID 把当时检索到的 context 一起拉出来对照看:

- 如果检索到的 context 里根本没有覆盖这个问题需要的信息,那是**检索问题**——需要回去看知识库的分块(chunking)策略、embedding 模型选择或者索引是否遗漏了相关文档。
- 如果检索到的 context 其实包含了正确信息,但回答依然给错了或者没有把关键信息用上,那是**生成问题**——需要调整 prompt、加强指令约束,或者考虑换一个更强的生成模型。
- 如果 context 和生成都没问题,但护栏把正确回答拦截或改写了,那是**护栏规则过严**,需要回去调整护栏的判定条件。

这样把低分回答按"检索 / 生成 / 护栏"分类之后,才能把优化资源花在真正的瓶颈环节上,而不是笼统地"再调一调 prompt 试试"。

**结合人工抽检,并且做交叉验证**:本章已经演示过 LLM 裁判在指标语义不对时会给出彻底误导性的结果,所以生产场景不能完全信任自动化评估,需要保留人工抽检作为交叉验证:

- 抽检比例上,建议对自动评估里的低分样本(比如 `policy_accuracy` 低于某个阈值的全部样本)做**全量**人工复核,因为这些恰恰是最可能影响用户体验的样本;再从中高分样本里按一定比例做**随机抽样**复核,作为"裁判是不是把高分也判对了"的基线检查——具体比例可以结合团队人力和业务风险等级调整,风险越高(比如涉及资金、法律条款的问答)抽检比例应该越高。
- 交叉验证的具体方法是:让人工评审员在**看不到裁判模型打分**的情况下,独立对同一批样本按同样的 rubric 打分,然后计算人工分数和裁判分数的一致率(完全一致的比例,或者分差在 1 分以内的比例)。如果一致率明显偏低,先不要急着下"LLM 裁判不可靠"的结论,而是回去检查:是 rubric 的档位描述不够清晰、裁判模型缺少打分示例,还是人工评审员之间本身对标准的理解就有分歧(这个可以通过再找第二个人工评审员复核一遍来排查)。
- 这个校准过程不是做一次就一劳永逸的——每次调整了 rubric 措辞、更换了裁判模型版本,或者业务政策本身发生了变化,都建议重新跑一轮人工-裁判一致性校验,确保自动化评估的信号在这些变化之后依然可信。
