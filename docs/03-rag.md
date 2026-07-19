# 第 3 章 · RAG 知识检索

## 3.1 这一步要解决什么问题

客户提供了一批政策文档(退货/换货/配送政策),需要 agent 能基于这些文档回答问题,而不是靠模型自己的通用知识瞎猜——这就是 RAG(Retrieval-Augmented Generation)存在的意义:先检索相关内容,再基于检索到的内容生成回答。

## 3.2 在 GCP 上具体怎么做

### 三种方案怎么选

| 方案 | 适合场景 | 上手成本 |
|---|---|---|
| **RAG Engine**(本章选用) | 中小规模文档集,快速接入 | 低——直接建 corpus、上传文件,不需要 GCS 桶 |
| Search(原 Vertex AI Search) | 大规模/异构文档,需要企业级检索 UX(排序、摘要、生成式回答作为独立产品) | 中——要建 GCS 桶、data store、app/engine |
| 手动 Vector Search | 需要完全自定义 ANN 策略、超大规模 | 高——自己生成向量、建索引、部署 endpoint(首次部署 20-30 分钟,常驻计费) |

3-5 份文档这种量级,RAG Engine 完胜,不需要为了"可能用不上的灵活性"多付部署和维护成本。

### 建 Corpus 并上传文档

```python
import vertexai
from vertexai.preview import rag

vertexai.init(project="adk-fde-lab", location="us-central1")

corpus = rag.create_corpus(
    display_name="acme_policy_docs",
    embedding_model_config=rag.EmbeddingModelConfig(
        publisher_model="publishers/google/models/text-embedding-005"
    ),
)

rag.upload_file(
    corpus_name=corpus.name,
    path="policy_docs/returns_policy.txt",
    display_name="returns_policy.txt",
)
```

## 3.3 核心代码

```python
from google.adk.tools.retrieval.vertex_ai_rag_retrieval import VertexAiRagRetrieval
from vertexai.preview import rag

ask_policy_docs = VertexAiRagRetrieval(
    name="retrieve_policy_docs",
    description="Use this tool to retrieve relevant passages from Acme Retail's policy documents...",
    rag_resources=[rag.RagResource(rag_corpus=RAG_CORPUS_NAME)],
    similarity_top_k=10,
    vector_distance_threshold=0.6,
)

policy_agent = LlmAgent(
    name="policy_agent",
    model="gemini-2.5-flash",
    instruction="Always ground your answer in the retrieved text; do not invent policy details.",
    tools=[ask_policy_docs],
)
```

!!! warning "硬限制:RAG 工具不能和其他工具混用"
    `VertexAiRagRetrieval` **不能**和别的工具挂在同一个 `LlmAgent` 实例上——ADK 对 Gemini 2.x+ 模型会把它转成模型原生的 "Vertex AI RAG Store" grounding 能力,这种模式下不支持再加别的 function tool。这就是为什么本手册的架构里 `policy_agent` 是独立子 agent,不能和 `order_agent` 的订单查询工具合并成一个 agent。

## 3.4 真实踩过的坑

### 坑 1:新项目默认建库模式被容量白名单挡住

```text
For new projects, using Spanner mode with RAG Engine in us-central1, us-east1,
and us-east4 is restricted to only allowlisted projects due to capacity limitation.
```

新项目在这几个热门区域默认走 Spanner 模式建库,新项目未经白名单直接被拒。解法是显式切到 Serverless 模式(项目/区域级的一次性设置):

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

## 3.5 RAG Engine 还能做什么

- **`rag.import_files()`**:除了单文件上传,还支持从 GCS 或 Google Drive 批量导入,并且能配置 `ChunkingConfig`(chunk 大小、重叠字数)控制切片策略。
- **混合检索(Hybrid Search)**:结合关键词检索和向量检索,对专有名词、型号、编号这类"向量检索容易漏"的内容效果更好。
- **`LlmRanker`**:检索出候选段落后,可以再用一个 LLM 做二次排序,提升最终喂给生成模型的上下文相关性。

## 3.6 优化方向

- **切片策略需要根据真实文档调优**。本章的政策文档很短,默认切片策略够用;真实客户文档往往是几十页的 PDF,切片大小、重叠字数需要针对文档结构专门调试,切得不好会导致"答案的关键信息被切断在两个 chunk 之间"。
- **增加检索质量的离线评估**,而不是只凭感觉判断"回答得准不准"——第 7 章的 LLM 评估方案可以扩展到专门评估检索环节(而不仅仅是最终回答)。
- **文档更新的自动化流水线**。本章是一次性手动上传,真实场景里客户的政策文档会持续更新,需要一套"文档变更 → 自动重新摄入 corpus"的流水线,而不是每次改动都手动跑脚本。
- **大规模场景下评估 Search**。如果客户文档量级涨到几万份、几十万份,或者需要面向最终用户的企业级搜索体验(高亮、摘要卡片),这时候 Search 提供的现成能力会比自己拼装 RAG Engine 更划算。
