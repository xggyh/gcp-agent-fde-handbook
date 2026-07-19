# 第 3 章 · RAG 知识检索

## 3.1 这一步要解决什么问题

客户提供了一批政策文档(退货/换货/配送政策),需要 agent 能基于这些文档回答问题,而不是靠模型自己的通用知识瞎猜——这就是 RAG(Retrieval-Augmented Generation)存在的意义:先检索相关内容,再基于检索到的内容生成回答。

对客服场景来说,这不是一个"锦上添花"的功能,而是一条硬性底线:政策类问题(能不能退、多久能退、谁承担运费)背后都有实际的money和纠纷成本,模型凭通用知识"编"出一个听起来合理但不准确的政策,对业务方来说比"不知道"更危险。所以本章的 agent 设计从一开始就贯彻一个原则——**policy_agent 只能说 RAG 检索到的内容,检索不到就应该老实说不知道,而不是自由发挥**。这也是为什么下面的代码里指令反复强调"grounded in the retrieved text; do not invent"。

## 3.2 在 GCP 上具体怎么做

### 三种方案怎么选

GCP 上做"给 agent 一个知识库"这件事,大致有三条路,选型本质是在"上手成本"和"长期灵活性/规模上限"之间做取舍:

| 维度 | RAG Engine(本章选用) | Vertex AI Search(原 Search & Conversation) | 手动 Vector Search |
|---|---|---|---|
| 适合场景 | 中小规模文档集,和 ADK/Agent 集成最紧密,快速接入 | 大规模/异构文档,需要企业级检索体验(排序、摘要、生成式回答可以作为独立产品对外) | 需要完全自定义 ANN 策略、超大规模、非纯文本的检索场景 |
| 上手成本 | 低——直接建 corpus、上传文件,不需要自己建 GCS 桶 | 中——要建 GCS 桶(或接入其他数据源)、data store、search app/engine | 高——自己生成向量、建索引、部署 endpoint |
| 文档导入方式 | `upload_file` 单文件上传 + `import_files` 批量导入(GCS、Google Drive) | 支持 GCS、网站抓取、BigQuery 等多种数据源连接器(具体清单以当前官方文档为准) | 完全自己写数据管道,从抽取到写入索引都要自己实现 |
| 内置 UI/前端 | 没有面向最终用户的现成 UI,检索结果需要 agent 自己组织成回答 | 有——可以直接拿到带高亮、摘要卡片的搜索结果页,也可以作为独立的"企业内部搜索"产品单独使用,不一定非要接 agent | 没有,纯后端能力,前端展示完全自己搭 |
| 计费模式 | 按 corpus 存储量 + 检索调用量计费(具体单价以当前 Vertex AI 定价页面为准) | 按数据规模 + 查询量计费,企业级功能对应的整体成本通常比 RAG Engine 高一档 | 按 endpoint 常驻部署时长 + 存储计费——**哪怕没有一次查询,部署着的 endpoint 也在持续计费** |
| 首次可用速度 | 分钟级(建 corpus、上传文件基本是同步返回) | 通常是分钟到小时级(建索引、部署 data store 需要时间) | 20-30 分钟起(实测的首次部署耗时),索引重建同样要等 |
| 混合检索/重排序 | 支持(Hybrid Search + `LlmRanker`,见 3.5) | 原生自带企业级排序、摘要生成能力 | 需要自己实现 |

3-5 份文档这种量级,RAG Engine 完胜,不需要为了"可能用不上的灵活性"多付部署和维护成本。

!!! tip "什么时候该重新考虑这个选型"
    这个表格不是一次性的决定,而是一个需要在项目演进过程中反复回看的判断依据。真正触发"该换方案了"的信号通常是:文档量级从几份涨到几万份、需要给非 agent 场景(比如内部员工的搜索门户)也复用同一套检索能力、或者客户明确要求"检索结果需要带高亮摘要的可视化界面"。这几个信号出现任何一个,就值得评估切到 Vertex AI Search,而不是硬着头皮在 RAG Engine 上堆功能。

### 环境准备:开通 API 与认证

在跑摄取脚本之前,项目需要先具备两个前提条件——这两步看起来是常规操作,但在 3.4 会看到,第二个 API 不开通的话,会在流程中段报一个不太好定位的错误:

```bash
# 确认/切换到目标项目
gcloud config set project adk-fde-lab

# RAG Engine 底层跑在 Vertex AI 之上,这个是基础依赖
gcloud services enable aiplatform.googleapis.com

# Serverless 模式的 RAG Engine 底层依赖 Vector Search API
# 这个依赖关系官方文档没有直接点明,是实测报错倒推出来的(详见 3.4 坑 2)
gcloud services enable vectorsearch.googleapis.com

# 本地开发机需要应用默认凭据,SDK 调用 vertexai.init() 时会用到
gcloud auth application-default login
```

依赖的 Python 包只需要一个较新版本的 `google-cloud-aiplatform`(`vertexai.preview.rag` 模块随包提供,不需要单独装扩展包,但版本要够新才有 `RagManagedDbConfig`/`Serverless` 这些相对新的 API)加上 `python-dotenv` 用来读写 `.env`:

```bash
pip install --upgrade google-cloud-aiplatform python-dotenv
```

准备好之后就可以跑摄取脚本了:

```bash
python ingest_rag.py
```

预期的终端输出大致是这样(文件名以 `policy_docs/` 目录下实际放的政策文档为准,这里仅作示例):

```text
已切换 RAG Engine 为 Serverless 模式
Created corpus: projects/adk-fde-lab/locations/us-central1/ragCorpora/4611686018427387904
Uploaded: returns_policy.txt
Uploaded: shipping_policy.txt
Uploaded: exchange_policy.txt
RAG_CORPUS 已写入 .env: projects/adk-fde-lab/locations/us-central1/ragCorpora/4611686018427387904
```

值得留意的是,目前 RAG Engine 的日常管理(建 corpus、上传/导入文件、查看文件列表)主要走 Vertex AI SDK 和 Cloud Console,还没有像 `gcloud compute` 那样成熟完整的 `gcloud` 子命令族。想确认 corpus 和文件是否真的建成功,除了看脚本自身的 `print` 输出,更可靠的方式是用 SDK 自带的查询接口(比如 `rag.list_corpora()`、`rag.list_files(corpus_name=...)`)或者去 Cloud Console 里对应的 Vertex AI 知识库管理页面核对——具体入口路径随 Console 版本会有调整,以当前实际界面为准。

### 建 Corpus 并上传文档:`ingest_rag.py` 逐段解读

这是本章 Phase 2 实际跑过的完整摄取脚本,一次性把"建库模式、建 corpus、上传文档、把 corpus 名字交给 agent"这条链路走完。下面按脚本的自然顺序逐段拆解——**顺序本身是有意义的**,不是随便排列的。

**第一段:初始化**

```python
import os
import vertexai
from vertexai.preview import rag

PROJECT_ID = "adk-fde-lab"
LOCATION = "us-central1"

vertexai.init(project=PROJECT_ID, location=LOCATION)
```

`vertexai.init()` 必须在任何 `rag.*` 调用之前执行,它把后续所有 RAG Engine SDK 调用都锁定在这个 project/location 上。这里 `PROJECT_ID`/`LOCATION` 直接硬编码在脚本顶部,对一次性的摄取脚本来说没问题,但如果这套脚本要被多个环境(dev/staging/prod)复用,应该改成读环境变量而不是写死——这个点我们放到 3.6 优化方向里细说。

**第二段:切到 Serverless 模式——为什么要放在最前面**

```python
rag.update_rag_engine_config(
    rag.RagEngineConfig(
        name=f"projects/{PROJECT_ID}/locations/{LOCATION}/ragEngineConfig",
        rag_managed_db_config=rag.RagManagedDbConfig(mode=rag.Serverless()),
    )
)
print("已切换 RAG Engine 为 Serverless 模式")
```

`ragEngineConfig` 是一个**项目 + region 级别的单例资源**——不是每个 corpus 各有一份配置,而是整个 project/location 下所有 RAG Engine 的底层存储模式共用同一份配置,所以这里调用的是 `update_rag_engine_config` 而不是 `create`。

这一步之所以被特意放在 `create_corpus` 之前,是因为这个配置必须在**第一次创建 corpus 之前**生效——新项目在部分热门 region 默认会尝试走 Spanner 存储模式建库,而这个模式目前对未经白名单的新项目是拒绝的(完整报错信息和排查过程见 3.4 坑 1)。把这一行固定写在脚本最开头,意味着这套脚本换到任何新项目、新 region 上跑,都能一次性绕开这个坑,而不用每次都重新报错、重新排查。

`rag.Serverless()` 是 `RagManagedDbConfig` 里可选的其中一种模式(错误信息里提到的默认模式是 Spanner);具体 SDK 里 Spanner 对应的配置类叫什么名字,这里没有实际用到,不做无根据的猜测,以官方文档或 SDK 源码里的实际定义为准。

**第三段:Embedding 模型配置**

```python
embedding_model_config = rag.EmbeddingModelConfig(
    publisher_model="publishers/google/models/text-embedding-005"
)
```

`publisher_model` 这个字符串遵循 Vertex AI Model Garden 统一的资源命名规则——`publishers/<发布方>/models/<模型 ID>`,和调用 Gemini 模型时用到的资源路径是同一套约定,只是这里指向的是一个 embedding 模型而不是生成模型。

**`text-embedding-005` 是什么**:它是 Google 在 Vertex AI Model Garden 上提供的一代文本嵌入模型,定位是把文本(这里是政策文档的段落)映射成向量,供后续做相似度检索用。它是 `text-embedding-*` 这个模型家族里相对新的一个版本,针对英语和代码类内容做了优化,同时仍然可用于中文等其他语言的文本(实际检索效果会因语言而异,建议针对客户真实文档语言做一次抽样验证,而不是默认它对所有语言都同样好用)。至于它具体的输出向量维度、最大输入长度、是否支持通过 `task_type` 区分"文档侧"和"查询侧"编码这些细节,建议直接查阅当前 Vertex AI 的模型卡片确认——这类具体数字迭代较快,写在手册里容易过时,不如养成"用之前查一眼模型卡片"的习惯。

有一点在选型时就要想清楚:**embedding 模型是绑定在 corpus 创建时刻的**,后续想换一个 embedding 模型,通常意味着要把 corpus 里已有的文档重新走一遍向量化流程,而不是简单地"改个配置就切换"。所以不要抱着"先随便选一个,以后再调"的心态。

**第四段:创建 Corpus**

```python
corpus = rag.create_corpus(
    display_name="acme_policy_docs",
    description="Acme 零售的退换货/配送政策文档",
    embedding_model_config=embedding_model_config,
)
print("Created corpus:", corpus.name)
```

`display_name`/`description` 只是给人看的元数据,方便在 Console 里区分多个 corpus;真正在代码里被引用、决定"agent 到底连到哪个知识库"的是 `corpus.name`——这是一个类似 `projects/.../locations/.../ragCorpora/<数字 ID>` 的完整资源路径,`<数字 ID>` 是创建时动态生成的,没法提前写死在代码里。这正是为什么脚本最后要把它写进 `.env`。

**第五段:批量上传文档**

```python
_here = os.path.dirname(__file__)
docs_dir = os.path.join(_here, "policy_docs")
for fname in sorted(os.listdir(docs_dir)):
    if fname.endswith(".txt"):
        rag_file = rag.upload_file(
            corpus_name=corpus.name,
            path=os.path.join(docs_dir, fname),
            display_name=fname,
            description=f"Policy doc: {fname}",
        )
        print("Uploaded:", rag_file.display_name)
```

这里用 `sorted()` 遍历目录不是功能上的必须,单纯是为了让每次运行脚本时上传顺序、日志输出是确定性的,方便对比不同批次的运行日志。`if fname.endswith(".txt")` 把范围限制在纯文本政策文档上,和真实客户场景里常见的 PDF/Word 文档不是一回事——那类文档的切片注意事项会在 3.6 单独展开。

这段循环也是一个"教学脚本 vs 生产脚本"的分界点:它没有对 `upload_file` 失败做任何异常捕获或重试,某一份文档上传失败会直接让整个脚本崩溃、后面的文档也传不上去。3-5 份文档量级下手动重跑一次成本不高,但如果这套脚本要接入自动化流水线(见 3.6 增量更新部分),这里就必须补上错误处理。

**`upload_file` 和 `import_files` 的区别**,这是实际选型时最容易搞混的一对 API:

- **`upload_file`**:同步、单文件、本地路径。调用一次上传一个文件,直接返回对应的 `RagFile` 对象,适合"文档在本地磁盘上,数量不多,需要立刻知道每个文件传成功没有"的场景——本章正是这种场景,3-5 份 txt 文件放在 agent 项目自己的 `policy_docs/` 目录里。
- **`import_files`**:批量、异步、面向"文档已经在云端某个位置"的场景,典型输入是一个 GCS 路径或 Google Drive 文件夹。它是一个长时间运行的操作(long-running operation),可以一次性配置大批文件,并且**可以在导入时传入 `ChunkingConfig` 控制切片策略**(这一点 `upload_file` 没有暴露同等的切片参数,细节见 3.5)。真实客户场景里,如果政策文档已经维护在客户自己的 GCS 桶或者共享 Drive 文件夹里,`import_files` 几乎总是更合适的选择。

简单说:本章因为文档从零开始、数量少、还在本地,所以选 `upload_file`;一旦文档来源变成"客户云端已有的文档库",就应该切到 `import_files`。

**第六段:把 corpus 名字交给 agent**

```python
env_path = os.path.join(_here, ".env")
with open(env_path, "a") as f:
    f.write(f"\nRAG_CORPUS={corpus.name}\n")
print("RAG_CORPUS 已写入 .env:", corpus.name)
```

因为 `corpus.name` 里的数字 ID 是运行时才生成的,没法写死在 `agent.py` 里,所以摄取脚本负责把它落盘到 `.env`,`agent.py` 再通过 `load_dotenv` + `os.environ["RAG_CORPUS"]` 读出来(下一节详细展开)。

这里用的是追加写模式(`"a"`),有一个需要留意的真实细节:如果这个脚本被重复执行(比如调试时跑了两次),`.env` 里会累积出多行 `RAG_CORPUS=...`,虽然 `python-dotenv` 解析同名 key 时后出现的值会覆盖前面的,实际生效的还是最后一次写入的 corpus,但文件本身会变得越来越"脏"、不便于人工核对。更稳妥的写法是先检查 `.env` 里是否已有 `RAG_CORPUS`、清理旧值再写入,脚本目前没有做这层检查,是它作为一次性教学/搭建脚本的简化之处。

## 3.3 核心代码

`agent.py` 里和 RAG 相关的部分——检索工具 `ask_policy_docs` 的定义,以及挂载它的 `policy_agent`:

```python
import os

from dotenv import load_dotenv
from google.adk.agents import LlmAgent
from google.adk.tools.retrieval.vertex_ai_rag_retrieval import VertexAiRagRetrieval
from vertexai.preview import rag

MODEL = "gemini-2.5-flash"

_HERE = os.path.dirname(__file__)
load_dotenv(os.path.join(_HERE, ".env"))  # 读取 ingest_rag.py 写入的 RAG_CORPUS

ask_policy_docs = VertexAiRagRetrieval(
    name="retrieve_policy_docs",
    description=(
        "Use this tool to retrieve relevant passages from Acme Retail's "
        "return, exchange, and shipping policy documents to answer the user's question."
    ),
    rag_resources=[rag.RagResource(rag_corpus=os.environ["RAG_CORPUS"])],
    similarity_top_k=10,
    vector_distance_threshold=0.6,
)

policy_agent = LlmAgent(
    name="policy_agent",
    model=MODEL,
    description="Answers questions about Acme Retail's return, exchange, and shipping policies.",
    instruction=(
        "You answer questions about Acme Retail's return/exchange/shipping policy "
        "using the retrieve_policy_docs tool. Always ground your answer in the "
        "retrieved text; do not invent policy details. Answer in Chinese."
    ),
    tools=[ask_policy_docs],
)
```

再往上一层,`root_agent` 根据用户意图把请求分派给 `order_agent`(第 2 章的订单查询,依赖 `OpenAPIToolset`)或者 `policy_agent`:

```python
root_agent = LlmAgent(
    name="root_agent",
    model=MODEL,
    description="Acme 零售客服总控 agent，负责判断用户意图并分派给对应的专职子 agent。",
    instruction=(
        "You are the customer service router for Acme Retail. "
        "If the user asks about an order's status or wants to cancel an order, "
        "delegate to order_agent. "
        "If the user asks about return/exchange/shipping policy, delegate to policy_agent. "
        "Be concise and helpful."
    ),
    sub_agents=[order_agent, policy_agent],
)
```

逐个说明关键参数:

- **`name="retrieve_policy_docs"`**:这是暴露给模型看到的工具名,`policy_agent` 的 `instruction` 里直接点名了这个工具,让模型知道"回答政策问题时应该调用它",而不是凭空生成。
- **`rag_resources=[rag.RagResource(rag_corpus=...)]`**:注意这是一个**列表**。当前只挂了一个 corpus,但这个设计天然支持未来把多个 corpus(比如按品类拆分的政策文档、或者不同语言版本的政策文档)一起挂到同一个检索工具上,不需要为每个 corpus 单独建一个工具、也不需要模型自己判断该查哪个库。

**`similarity_top_k` 控制的是"检索返回多少个候选段落"**——这里设成 10,意味着每次调用这个工具,最多会拿回 10 个最相似的文本片段(chunk)交给生成模型参考。这个数字本质是在"召回"和"上下文噪音/成本"之间做取舍:

- 设得太小(比如 2-3),遇到答案分散在多个段落里的问题(比如"退货政策"和它的"例外情况"分别在两段),很可能漏掉其中一段,回答不完整。
- 设得太大,会把大量相关度较低的内容也塞进模型的上下文,一方面拉长 prompt、增加调用成本和延迟,另一方面容易在生成阶段引入噪音,让模型东拼西凑出不准确的回答。

**`vector_distance_threshold` 控制的是"距离在多远之内的候选才算数"**——即便某个候选段落排进了 top_k 的名次,如果它和查询之间的向量距离超过这个阈值,也会被过滤掉,不会传给生成模型。这里设成 0.6,反映的是一个明确的取舍倾向:**宁可有的问题检索不到东西、让 agent 老实说不确定,也不要把明显不相关的内容硬塞进去凑数**。调这个参数本质是在召回率(recall)和精确率(precision)之间平衡:

- 阈值调大(放宽),会有更多候选段落通过筛选,召回率提高,但混入不相关内容的风险也上升,容易诱发"看似言之有据、实际上下文不对题"的回答。
- 阈值调小(收紧),筛选更严格,精确率提高,但可能出现"检索不到任何满足条件的段落",导致 agent 对本该能回答的问题也说不知道。

这两个参数是本章能直接调的两个旋钮,具体数值(10 和 0.6)是针对政策文档这种"篇幅短、条目相对独立"的场景定的,换成别的文档集合(比如很长的产品手册)大概率需要重新实测调整,而不是照搬。

!!! warning "硬限制:RAG 工具不能和其他工具混用"
    `VertexAiRagRetrieval` **不能**和别的工具挂在同一个 `LlmAgent` 实例上——ADK 对 Gemini 2.x+ 模型会把它转成模型原生的 "Vertex AI RAG Store" grounding 能力,这种模式下不支持再加别的 function tool。这就是为什么本手册的架构里 `policy_agent` 是独立子 agent,不能和 `order_agent` 的订单查询工具合并成一个 agent——即便从业务逻辑上看,"查订单"和"问政策"完全可以是同一个客服角色该做的两件事。这个限制直接决定了整个多 agent 的拆分方式:**工具类型的技术限制,反过来定义了 agent 的组织边界**,这是在做架构设计时容易被低估的一类约束。

## 3.4 真实踩过的坑

### 坑 1:新项目默认建库模式被容量白名单挡住

```text
For new projects, using Spanner mode with RAG Engine in us-central1, us-east1,
and us-east4 is restricted to only allowlisted projects due to capacity limitation.
```

新项目在这几个热门区域默认走 Spanner 模式建库,新项目未经白名单直接被拒。这类"沙盒/实验性质"的新建项目撞上容量限制的概率明显更高——正式和客户签约的合作项目通常已经走过配额申请流程,但快速搭 demo 用的临时项目大概率会第一次就撞上这个报错。解法是显式切到 Serverless 模式(项目/区域级的一次性设置):

```python
rag.update_rag_engine_config(
    rag.RagEngineConfig(
        name=f"projects/{PROJECT_ID}/locations/{LOCATION}/ragEngineConfig",
        rag_managed_db_config=rag.RagManagedDbConfig(mode=rag.Serverless()),
    )
)
```

### 坑 2:切完 Serverless 又报另一个 API 未开通

```text
Vector Search API has not been used in project ... before or it is disabled.
```

Serverless 模式底层依赖 Vector Search API,这个依赖关系官方文档没有直接点明,得靠实际报错发现:

```bash
gcloud services enable vectorsearch.googleapis.com
```

!!! tip "这两个坑本质上是同一个根因"
    追根溯源,坑 1 和坑 2 都是同一件事的两面:**新建的 GCP 项目默认配置没法直接开箱即用 RAG Engine 的默认存储路径**,必须显式做一次一次性的项目级配置调整,而这个配置本身又依赖一个没有在主文档里显式声明的底层 API。排查思路上没有捷径——两次都是"先报错、再顺着报错信息去搜/去试",这也是为什么 3.2 里把 `update_rag_engine_config` 固定写在脚本第一行:把已经踩过的坑,变成脚本自带的免疫力,而不是指望每个新用这套脚本的人都重新踩一遍。

## 3.5 RAG Engine 还能做什么

- **`rag.import_files()` + `ChunkingConfig`**:除了 `upload_file` 这种单文件同步上传,`import_files` 支持从 GCS 或 Google Drive 批量导入,并且能在导入时传入 `ChunkingConfig` 显式控制切片策略,核心是 `chunk_size` 和 `chunk_overlap` 这两个旋钮。`chunk_size` 决定每个检索单元大致覆盖多少内容——可以按"一个 chunk 大概对应政策文档里的一个自然段或者半页内容"这个直觉去定初始值,而不是拍脑袋定一个字符数;`chunk_overlap` 决定相邻两个 chunk 之间保留多少重叠内容,目的是给切片边界一个缓冲区,避免关键信息恰好卡在两个 chunk 的分界线上。经验上 overlap 会设成 chunk_size 的一个较小比例(远小于 chunk 本身),太小起不到缓冲作用,太大又会导致检索结果里出现大量重复内容,变相挤占了 `similarity_top_k` 的有效名额。这两个参数具体的默认值和可调范围随 SDK 版本会有变化,真正调优时建议直接实测而不是照搬某个记忆中的数字。
- **混合检索(Hybrid Search)**:结合关键词检索和向量检索,对专有名词、型号、订单编号这类"语义相近但字面完全不同"或者反过来"字面相似但语义不同"的内容,纯向量检索容易漏掉或者检索到不相关的近似词,混合检索能把关键词匹配的确定性和向量检索的语义泛化能力结合起来,比较适合政策文档里夹杂大量 SKU、条款编号的场景。
- **`LlmRanker`**:检索出候选段落后,可以再用一个 LLM 对这批候选做二次排序,把"向量距离近但实际不太相关"的内容排到后面,把真正切题的内容排到前面。这在候选质量比响应速度更重要的场景(比如涉及金额、时限这类容易出错的政策条款)上性价比更高,代价是多了一次模型调用,会增加延迟和成本,需要按场景取舍要不要开启。
- **除了 `text-embedding-005`,还有其他 embedding 模型可选**:Vertex AI 的 Model Garden 在 embedding 这个类别下并不只提供这一个模型,还有面向不同语言覆盖范围、不同定位的其他版本可选。具体当前有哪些型号可用、各自的定位和参数差异,建议直接查阅 Vertex AI 官方文档里 Embeddings 模型的可用列表确认——这类清单更新频率较高,这里不逐一列举以免写出过时或不准确的型号信息。
- **`rag.retrieval_query()` 直接测试检索效果**:SDK 里还提供一个可以直接对某个 corpus 发起检索测试的接口,不需要真的跑一遍完整的 agent 对话流程就能拿到检索到的 chunk 内容和相似度分数。这个能力在调试阶段非常实用——想验证"调整 chunk 策略/embedding 模型/`similarity_top_k` 之后检索效果有没有变好",直接用这个接口跑一批已知问题会比每次都通过 agent 对话来验证快得多,也是 3.6 里"检索质量专项评估"能落地的基础。

## 3.6 优化方向

- **真实长文档(PDF 等)的切片策略需要专门打磨**。本章的政策文档是干净的纯文本,默认切片策略够用;真实客户文档往往是几十页的 PDF,结构远比 txt 复杂——多级标题、表格、脚注、重复出现的页眉页脚,甚至扫描版需要先过 OCR。几个具体要注意的点:
    - 尽量先把 PDF 转成保留标题层级的结构化文本(比如 Markdown),而不是直接把整页文本一股脑塞进导入流程——页眉页脚这类重复出现的噪音文字会污染向量空间,拉低检索的区分度。
    - `chunk_size` 不能只按字符数一刀切,要尽量贴合文档本身的段落/小节边界。切得不好的典型后果是:"退货条件"和紧跟着的"例外情况"被切到两个不同的 chunk 里,检索时可能只召回其中一个,回答就会遗漏关键限制条件,看起来是"检索没问题",实际是切片策略把语义单元切碎了。
    - 价格表、时限对照表这类表格内容尤其容易被简单的字符级切片破坏结构,需要额外处理(比如提前转成自然语言描述,或者使用对表格结构敏感的解析工具),否则切出来的 chunk 可能只剩下一堆脱离上下文的数字。
    - 实操建议:挑几个"答案分散在多个段落里"的真实问题,用不同的 `chunk_size`/`chunk_overlap` 组合分别导入,再用 3.5 提到的 `rag.retrieval_query()` 直接看返回的 chunk 内容和相关性排序,而不是等到最终生成回答阶段才发现"漏了一句话"——这一层调试完全不需要跑生成模型,成本低、反馈快。

- **检索质量需要专项评估,不能只靠第 7 章的端到端回答评分**。第 7 章的 LLM 裁判评估看的是"最终回答对不对",但回答不对可能来自两个完全不同的环节:检索没找到对的段落,或者找到了但生成模型没用好。这两层问题的排查方向和修复方式完全不同,混在一起看容易把责任归错地方。具体做法:
    - 准备一批"问题 → 期望被检索到的正确段落/文档"的标注数据,哪怕只是人工标出"这几个 chunk 应该出现在 top_k 结果里"这个粗粒度的标注也够用。
    - 用 `rag.retrieval_query()` 跑这批问题,对比实际检索结果和期望标注,统计类似 Recall@K(期望的段落有没有被检索到)、Precision@K(检索到的结果里有多少是真正相关的)这类指标——这一层不需要调用生成模型,可以跑得更频繁、成本更低。
    - 把这层检索质量的结果和第 7 章的最终回答打分做交叉分析:如果某一类问题最终回答分数偏低,先看是不是这类问题本身的检索 Recall 就低,而不是急着怀疑 prompt 或者生成模型的能力。
    - 每次调整 chunk 策略、embedding 模型或者 `similarity_top_k`/`vector_distance_threshold` 这类参数之后,都应该把这套检索质量评估当成回归测试跑一遍,而不是只在项目初期验证一次就再也不管。

- **文档更新需要一套增量摄入流水线,而不是每次手动重跑脚本**。本章是一次性手动执行 `ingest_rag.py`,真实场景里客户的政策文档会持续变化(比如促销季节临时调整退货规则),需要设计一套"文档变更 → 自动重新摄入 corpus"的流水线。大致思路:
    - 如果文档源维护在 GCS 桶里,可以给桶配置对象变更通知(通过 Pub/Sub),文件新增或更新时推送一条消息,触发一个 Cloud Function/Cloud Run 服务,在收到事件后调用 `rag.import_files()` 把变更的文件重新导入对应 corpus。RAG Engine 对"同名文件重新导入"具体是覆盖更新还是需要先手动删除旧文件这一点,行为细节以官方文档当前描述为准,落地前要先在测试环境验证一遍,不要假设它一定是幂等的。
    - 如果文档源是 Google Drive 共享文件夹,思路类似,但触发信号大概率要换成定时轮询——用 Cloud Scheduler 定期触发一个 Cloud Function,检查文件的修改时间戳有没有变化,因为 Drive 的变更通知接入复杂度比 GCS 的对象通知要高不少。
    - 这套流水线里应该内嵌上面提到的检索质量回归测试:文档重新摄入完成后,自动跑一遍已标注的问题集,如果 Recall/Precision 明显下降就应该告警,而不是文档一改、corpus 一更新就默默上线,等真实用户反馈"这次回答不对"了才发现质量下降了。
    - 也需要考虑版本和回滚:如果某次导入的文档解析出了问题(比如 PDF 转换失败导致 corpus 里存了一堆乱码),要能快速回滚到上一个可用状态——比如给 corpus 或者文件加版本标记,或者保留旧 corpus 一段观察期再切生产流量,而不是新旧数据混在同一个 corpus 里没法回退。

- **文档量级或者交互形态变化时,重新评估切到 Search 的时机**。如果客户文档量涨到几万份、几十万份,或者产品形态需要面向最终用户提供带高亮、摘要卡片的企业级搜索体验,这时候 Vertex AI Search 提供的现成能力(参见 3.2 的对比表)会比继续在 RAG Engine 上自己拼装 chunk 策略、混合检索、重排序划算得多——这是一个应该在项目规划阶段就预留判断节点的决策,而不是等到 RAG Engine 明显撑不住了才仓促迁移。
