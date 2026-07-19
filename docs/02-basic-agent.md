# 第 2 章 · 基础 Agent 与工具集成

## 2.1 这一步要解决什么问题

FDE 工作里最常见的场景就是这个:**客户已经有一套现成的业务系统(REST API),需要让 agent 能调用它**。客户不会因为你要做一个 agent 就重写他们的订单系统、CRM 或者工单系统——你的工作是在不改动客户后端的前提下,把这些系统"接"进 agent 的工具体系里,让 LLM 能够可靠地调用它们、拿到结构化的结果。

这是本手册所记录的真实项目(Acme 零售客服 agent)Phase 1 的范围,目标拆成两件具体的事:

1. 搭出第一个能跑起来的 ADK agent——哪怕它现在只会做一件事(查订单),也要先把"agent 能被发现、能被启动、能对话"这条链路跑通。
2. 把客户的订单查询系统(这里用一个 Mock 的 FastAPI 服务模拟真实的 Acme 订单系统)包装成 agent 能调用的工具,并且验证 agent 真的能通过对话触发一次真实的 HTTP 调用、拿到真实的数据。

Phase 1 结束时,这个 agent 只有一个专职子 agent——`order_agent`,以及负责把用户请求路由过去的 `root_agent`。第 3 章会在此基础上接入基于 Vertex AI RAG 引擎的 `policy_agent`,处理退换货政策类的问题;这一章我们把范围严格限定在订单查询/取消这条链路上,把地基打扎实。

## 2.2 在 GCP 上具体怎么做

### ADK 期望的项目结构

```text
adk_fde_project/                    <- adk web/run 从这里启动
└── customer_service_agent/         <- 这个文件夹名字就是agent的包名
    ├── __init__.py                 <- 必须写: from . import agent
    ├── agent.py                    <- 必须定义模块级的 root_agent
    ├── .env                        <- API Key、RAG_CORPUS 等环境变量(RAG_CORPUS 第3章才会用到)
    └── order_api/                  <- Mock 订单系统:FastAPI 源码 + 导出的 OpenAPI 规范
        ├── main.py
        └── openapi.json
```

`__init__.py` 不是可以省略的样板文件——如果它不存在或者没有 `from . import agent`,ADK 的自动发现机制会直接跳过这个文件夹,不会报错,只是"什么都没找到"。这是一个非常容易在排查时被忽略的细节:你在 `adk web` 里死活看不到自己的 agent,第一反应往往是去查 agent 定义本身有没有问题,而不是去查这个一行代码的包初始化文件。

`order_api/` 这个子目录之所以放在 agent 包内部而不是外面单独一个仓库,是因为 `agent.py` 用相对路径读取它:

```python
_HERE = os.path.dirname(__file__)
with open(os.path.join(_HERE, "order_api", "openapi.json")) as f:
    _order_spec = json.load(f)
```

这样无论 `adk web` 从哪个工作目录启动,只要 `customer_service_agent/` 这个包本身完整,`openapi.json` 就总能被找到,不依赖调用方的当前工作目录。这是 FDE 项目里一个很实用的小习惯:**凡是 agent 运行时要读的本地文件,路径都相对 `__file__` 而不是相对 cwd 去拼。**

### 第一步:把 Mock 订单系统本地跑起来

在写 agent 之前,先要有一个能被包装的 API。本章场景里这是一个模拟 Acme 零售订单系统的 FastAPI 应用(完整代码见 2.3 节)。本地起服务:

```bash
pip install fastapi uvicorn
uvicorn order_api.main:app --reload --port 8000
```

- `--reload`:开发阶段热重载,改代码不用手动重启进程。生产/部署环境不会带这个参数。
- `--port 8000`:和 `main.py` 里 `servers=[{"url": "http://127.0.0.1:8000"}]` 保持一致——这个端口号后面会被写进导出的 OpenAPI 规范里,不是随便选的。

跑起来之后,访问 `http://127.0.0.1:8000/docs` 能看到 FastAPI 自动生成的交互式文档,可以先用它手动点一遍 `get_order_status`、`cancel_order` 两个接口,确认业务逻辑本身没问题,再往下走工具集成这一步。

### 第二步:导出 OpenAPI 规范

```bash
curl http://127.0.0.1:8000/openapi.json -o order_api/openapi.json
```

这一步产出的 `openapi.json` 会被提交进代码仓库,后续 `agent.py` 是从磁盘直接读这个文件,而不是每次启动 agent 都重新请求线上 API 的 `/openapi.json` 端点。这么做的好处是 agent 启动不依赖后端 API 当时是否可访问;代价是这份规范是一个"快照",后端接口如果发生变化而没有重新导出、重新提交,agent 侧看到的工具 schema 就会跟真实接口不一致(这一点在 2.5 节的坑里会具体展开)。

### 第三步:把 Mock API 部署到 Cloud Run

本地跑通只是验证逻辑,真实项目里这个 Mock API(以及未来客户的真实后端)需要一个稳定、可从公网(或者 agent 运行环境所在网络)访问的地址。这也是为什么 `agent.py` 里最终把 `servers` 覆盖成了一个 Cloud Run 地址:

```text
https://order-status-api-372239021300.us-central1.run.app
```

部署命令:

```bash
gcloud run deploy order-status-api \
  --source . \
  --region us-central1 \
  --allow-unauthenticated \
  --port 8000
```

逐个参数说明:

- `order-status-api`:Cloud Run 服务名,会直接体现在生成的默认域名里(`order-status-api-<hash>-<region>.run.app`),这也是为什么最终 URL 长这个样子。
- `--source .`:不手写 Dockerfile,让 Cloud Build 用 Buildpacks 自动检测项目类型(发现 `requirements.txt` 里有 fastapi/uvicorn)并打包镜像。这是 FDE 场景下加速交付的常用做法——客户环境往往没有专职的容器化工程师配合,能少写一个 Dockerfile 就少一处需要维护的东西。
- `--region us-central1`:选区域时通常和项目里其他 GCP 资源(Vertex AI 模型调用、未来的 RAG Corpus 等)保持同一区域,减少跨区域延迟,也方便统一看 Cloud Logging/Cloud Trace。
- `--allow-unauthenticated`:允许公网直接访问,不要求调用方带 Google 签发的身份令牌。这里做了一个明确的架构选择:**认证收敛在应用层(`X-API-Key` 头),而不是 Cloud Run/IAM 层**。如果不加这个参数,Cloud Run 默认要求调用方是经过 IAM 授权的身份(比如一个服务账号的 ID Token),而这份 `OpenAPIToolset` 的配置只准备了 API Key,请求会在还没到应用代码之前就被 Cloud Run 拦成 403。是否要换成 IAM 层认证,是 2.6 节要展开的一个真实选型点。
- `--port 8000`:告诉 Cloud Run 容器内监听的端口是哪个,要和应用实际监听的端口一致。

!!! note "Buildpacks 部署 FastAPI 需要一个入口说明文件"
    用 `--source .` 走 Buildpacks 路线时,Buildpacks 需要知道怎么启动这个 ASGI 应用。常见做法是在项目根目录放一个 `Procfile`:

    ```text
    web: uvicorn main:app --host 0.0.0.0 --port $PORT
    ```

    注意这里必须监听 `0.0.0.0` 而不是 `127.0.0.1`,且端口要用 Cloud Run 注入的环境变量 `$PORT`,不能写死 `8000`——本地开发用固定端口,线上部署要读环境变量,这是两个不同的运行模式,很容易在从本地搬到 Cloud Run 时漏改。如果不想依赖 Buildpacks 的自动检测,写一个显式的 Dockerfile 也是完全可以的,只是本章示例项目选择了更轻量的路径。

### 第四步:验证部署后的 API 行为

部署完成后,先用 `curl` 手动验证一遍认证逻辑是否符合预期,再把它接进 agent——这一步能帮你把"API 本身的问题"和"agent 侧工具调用的问题"提前分开排查:

```bash
# 不带 API Key:应该被拒绝
curl -i https://order-status-api-372239021300.us-central1.run.app/orders/1001

# 带错误的 API Key:应该是 401
curl -i -H "X-API-Key: wrong-key" \
  https://order-status-api-372239021300.us-central1.run.app/orders/1001

# 带正确的 API Key:应该拿到订单数据
curl -i -H "X-API-Key: secret-key-123" \
  https://order-status-api-372239021300.us-central1.run.app/orders/1001
```

### 第五步:本地起 ADK 开发界面测试 agent

```bash
cd adk_fde_project
adk web
```

`adk web` 会扫描当前目录下所有符合"包内有 `__init__.py` 且导入了 `agent` 模块"规则的文件夹,在网页界面里把它们列成可选的 agent,选中 `customer_service_agent` 就能直接开始对话调试,同时能看到每一轮对话背后调用了哪些工具(这个界面本身也是排查 2.3 节要讲的 `sub_agents` 路由问题最直接的工具)。命令行场景下也可以用 `adk run customer_service_agent` 做单次交互测试。

## 2.3 核心代码

### order_api/main.py:把 Mock 后端写成"OpenAPIToolset 友好"的样子

```python
from fastapi import FastAPI, HTTPException, Security
from fastapi.security.api_key import APIKeyHeader
from pydantic import BaseModel

API_KEY = "secret-key-123"
API_KEY_NAME = "X-API-Key"

app = FastAPI(
    title="Order Status API",
    version="1.0.0",
    description="Mock API for looking up order status (模拟 Acme 零售的订单系统)",
    servers=[{"url": "http://127.0.0.1:8000"}],
)

api_key_header = APIKeyHeader(name=API_KEY_NAME, auto_error=True)

def verify_api_key(api_key: str = Security(api_key_header)) -> str:
    if api_key != API_KEY:
        raise HTTPException(status_code=401, detail="Invalid API Key")
    return api_key

class OrderStatus(BaseModel):
    order_id: str
    status: str
    eta_days: int

class CancelResult(BaseModel):
    order_id: str
    cancelled: bool

FAKE_ORDERS = {
    "1001": {"order_id": "1001", "status": "shipped", "eta_days": 2},
    "1002": {"order_id": "1002", "status": "processing", "eta_days": 5},
    "1003": {"order_id": "1003", "status": "delivered", "eta_days": 0},
}

@app.get(
    "/orders/{order_id}",
    response_model=OrderStatus,
    operation_id="get_order_status",
    summary="Look up the status of an order",
    dependencies=[Security(verify_api_key)],
)
def get_order_status(order_id: str):
    order = FAKE_ORDERS.get(order_id)
    if not order:
        raise HTTPException(status_code=404, detail="Order not found")
    return order

@app.post(
    "/orders/{order_id}/cancel",
    response_model=CancelResult,
    operation_id="cancel_order",
    summary="Cancel an existing order",
    dependencies=[Security(verify_api_key)],
)
def cancel_order(order_id: str):
    if order_id not in FAKE_ORDERS:
        raise HTTPException(status_code=404, detail="Order not found")
    FAKE_ORDERS[order_id]["status"] = "cancelled"
    return {"order_id": order_id, "cancelled": True}
```

逐个关键设计说明为什么要这么写(而不是随手用 FastAPI 默认值):

**`servers=[{"url": "http://127.0.0.1:8000"}]`**——OpenAPI 3.0 规范里 `servers` 字段声明的是"这份接口文档描述的服务实际部署在哪个 base URL 下"。FastAPI 默认不会主动填这个字段,除非你显式传 `servers=[...]` 给 `FastAPI()` 构造函数。它在 `/docs` 交互式文档里其实不是必需的——Swagger UI 会自己用当前访问的域名去发请求,所以本地开发时哪怕不写 `servers`,`/docs` 页面点一点也一切正常,这也是这个参数最容易被漏掉的原因。但 `OpenAPIToolset` 不是浏览器,它没有"当前访问域名"这个概念,只能严格按 `spec["servers"][0]["url"]` 去拼接每次工具调用的真实请求地址——漏了这个字段,或者字段指向的地址是错的,工具调用会直接失败。这个坑在 2.5 节还会再展开一次。

**`operation_id="get_order_status"` / `operation_id="cancel_order"`**——这两个字符串不是随便起的标签,它们直接决定了 `OpenAPIToolset` 生成的工具在 LLM 眼里叫什么名字、以什么函数签名被调用。如果不显式指定,FastAPI 会按"函数名 + 路径 + HTTP 方法"的规则自动生成一个 `operation_id`,类似 `get_order_status_orders__order_id__get_orders__order_id__get` 这种又长又难读的字符串。这种自动生成的名字对人类阅读体验很差,对 LLM 的工具选择也不友好——模型做 function calling 时,函数名本身就是一部分语义信号,一个清晰的 `get_order_status` 远比一串下划线堆砌的自动生成名字更容易被正确调用。所以但凡是要给 agent 用的 API,`operation_id` 都应该手写、明确、见名知意。

**`Security(verify_api_key)` 而不是 `Depends(verify_api_key)`**——这是个很容易被忽略但影响真实行为的细节。`Security()` 是 `Depends()` 的一个特化版本,专门用来声明"这个依赖本身也是一个安全方案(SecurityBase 的实例)"。当依赖链条里包含 `Security()` 包裹的 `SecurityBase` 子类(这里是 `APIKeyHeader`)时,FastAPI 会自动把对应的认证要求写进导出的 OpenAPI 规范的 `components.securitySchemes` 和每个操作的 `security` 字段里;如果只用普通的 `Depends()`,即使运行时效果一样(请求没带对的 key 一样会被拒绝),**生成的 `openapi.json` 里完全不会出现任何"这个接口需要认证"的声明**。而 `OpenAPIToolset` 正是靠解析规范里的 `security` 声明,才知道调用这个工具时需要附加认证信息、以及附加到哪个位置(header/query/cookie)。所以这里用 `Security` 不是风格偏好,是让工具集成能自动感知认证要求的必要条件。

同样值得注意的是,`dependencies=[Security(verify_api_key)]` 是写在**每个路由**上,而不是用 FastAPI 的全局中间件或者给整个 `app` 加统一依赖。这么做保留了后续给某些接口开白名单(比如未来加一个不需要认证的健康检查接口)的灵活性,也让每个操作的认证要求在规范里是显式、独立声明的,而不是"整个服务都要认证"这种笼统的隐含约定。

**`response_model=OrderStatus` / `response_model=CancelResult`**——用 Pydantic 模型声明返回结构,这不仅仅是接口文档好看。这两个模型会被编码进 OpenAPI 规范的 schema 里,`OpenAPIToolset` 依据这份 schema 生成工具的输入输出结构描述,间接影响了 LLM 理解"这个工具会返回什么样的数据"的能力——如果只返回裸 `dict` 不声明 `response_model`,规范里这部分 schema 就会是空的或者非常粗糙。

### agent.py:order_agent 与 root_agent

!!! note "关于这份代码的范围"
    下面贴的是项目里 `agent.py` 的完整内容——包括第 3 章才会详细拆解的 `policy_agent` 和 `ask_policy_docs`(基于 Vertex AI RAG 引擎的检索工具)部分。这一章只逐行解读 `order_agent`、`order_api_toolset` 和 `root_agent` 这几块;`policy_agent` 涉及的 RAG 语料库配置和检索工具的用法留到下一章展开,这里看到它出现在 `sub_agents` 列表里,先有个印象即可。

```python
import json
import os

from dotenv import load_dotenv
from google.adk.agents import LlmAgent
from google.adk.tools.openapi_tool.auth.auth_helpers import token_to_scheme_credential
from google.adk.tools.openapi_tool.openapi_spec_parser.openapi_toolset import OpenAPIToolset
from google.adk.tools.retrieval.vertex_ai_rag_retrieval import VertexAiRagRetrieval
from vertexai.preview import rag

MODEL = "gemini-2.5-flash"

_HERE = os.path.dirname(__file__)
load_dotenv(os.path.join(_HERE, ".env"))

with open(os.path.join(_HERE, "order_api", "openapi.json")) as f:
    _order_spec = json.load(f)

_order_spec["servers"] = [{"url": "https://order-status-api-372239021300.us-central1.run.app"}]

_auth_scheme, _auth_credential = token_to_scheme_credential(
    "apikey", "header", "X-API-Key", "secret-key-123"
)

order_api_toolset = OpenAPIToolset(
    spec_dict=_order_spec,
    auth_scheme=_auth_scheme,
    auth_credential=_auth_credential,
)

order_agent = LlmAgent(
    name="order_agent",
    model=MODEL,
    description="Handles order status lookups and order cancellations.",
    instruction=(
        "You help users check the status of their orders and cancel orders. "
        "Use the get_order_status and cancel_order tools as needed. "
        "Order IDs look like '1001'."
    ),
    tools=[order_api_toolset],
)

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

聚焦订单这条线,逐段拆解:

- `load_dotenv(os.path.join(_HERE, ".env"))`:即使这一章 `order_agent` 本身不需要读任何环境变量,`.env` 依然在 `agent.py` 顶层被加载——因为这是一个模块级的初始化,`policy_agent` 后面要用到的 `RAG_CORPUS` 也是从这里的 `.env` 读出来的。提前把这行放在文件最上面,是为了让第 3 章接入 RAG 时不需要再回头改这部分基础设施代码。
- `_order_spec["servers"] = [...]`:这是本章最值得记住的一行代码。`openapi.json` 是本地开发时 `curl` 下来的快照,里面记录的 `servers` 还是 `http://127.0.0.1:8000`(2.2 节导出时的状态)。与其在导出规范这一步就手动改文件、或者每次环境变化都重新导出,这里选择在 **agent 构建时,用代码把 `servers` 字段覆盖成当前环境该指向的真实地址**。这样"规范的结构从哪来"和"请求实际打到哪里去"就被解耦了:同一份 `openapi.json` 可以配合不同环境的 `.env`/代码分支,分别覆盖成本地地址、测试环境地址、生产 Cloud Run 地址,而不需要为每个环境维护一份不同的规范文件。
- `token_to_scheme_credential("apikey", "header", "X-API-Key", "secret-key-123")`:这个 ADK 提供的辅助函数,把"认证方案"和"认证凭据"这两件事拆开构造——`_auth_scheme` 描述的是"这是一个放在 header 里、名字叫 `X-API-Key` 的 API Key 认证",`_auth_credential` 才是真正的密钥值。之所以要拆成两个对象,是因为同一套 `auth_scheme`(认证方式的形状)在不同环境下可能对应不同的 `auth_credential`(具体密钥值),这也是 2.7 节要讲的 Secret Manager 改造能够干净嵌入的原因——需要替换的只是 `_auth_credential` 那一部分。
- `OpenAPIToolset(spec_dict=_order_spec, auth_scheme=_auth_scheme, auth_credential=_auth_credential)`:这一步做的事情比表面上看起来多得多。它会解析整份 `_order_spec`,为规范里**每一个带 `operation_id` 的操作**生成一个对应的工具,并且把 `auth_scheme`/`auth_credential` 自动应用到所有需要认证的操作上——也就是说 `get_order_status` 和 `cancel_order` 这两个工具,都不需要你手写"往 header 里加 `X-API-Key`"这行代码,`OpenAPIToolset` 会在每次实际发起 HTTP 请求前自动附加。
- `tools=[order_api_toolset]`:注意这里传给 `order_agent` 的 `tools` 列表里只有**一个元素**,但这个元素本身在 ADK 眼里会展开成两个可调用的工具(`get_order_status`、`cancel_order`)。这是 `OpenAPIToolset` 和手写 `FunctionTool` 的一个直观区别:后者是"一个函数对应一个工具",前者是"一整份规范对应一批工具",数量取决于规范里声明了多少个操作。

### sub_agents 机制的工作原理

`root_agent` 里的 `sub_agents=[order_agent, policy_agent]` 是 ADK 多智能体分派机制的核心配置项,但它具体是怎么让 `root_agent` "知道该转给谁"的,值得完整讲清楚,因为这直接决定了你在真实项目里怎么写 `description`、怎么排查路由错误。

**第一步:`description` 不是文档,是会被注入模型上下文的路由信号。** 当一个 `LlmAgent` 声明了 `sub_agents`,ADK 在组装这个 agent 实际发给模型的上下文时,会把每个子 agent 的 `name` 和 `description` 拼进去,相当于告诉 `root_agent` 背后的模型:"你手头有这些专职助手可以调用,他们各自负责什么"。这也是为什么本章示例里 `order_agent` 的 `description` 写的是`"Handles order status lookups and order cancellations."`——这句话不是给人类读者看的注释,是真正参与模型推理、影响路由决策的输入。如果这句话写得含糊(比如写成 `"handles stuff"`),或者两个子 agent 的 `description` 内容高度重叠,模型在路由时就有更高概率选错、或者干脆自己直接回答而不转发。

**第二步:`root_agent` 自己的 `instruction` 提供显式的路由规则,和 `description` 共同起作用。** 本章代码里 `root_agent.instruction` 明确写了"如果用户问订单状态或者要取消订单,转给 order_agent;如果问退换货/物流政策,转给 policy_agent"。这是一层比"靠 description 自己推断"更直接的路由指令,两者叠加使用——`description` 告诉模型"有哪些选项及其职责范围",`instruction` 告诉模型"具体按什么规则去选"。真实项目里这两者最好保持一致、互相印证,而不是各说各话。

**第三步:路由决策真正落地时,ADK 是通过一个内置工具 `transfer_to_agent` 实现的,不是框架内部某种黑箱跳转。** `sub_agents` 配置生效后,`root_agent` 背后的模型除了能调用它自己声明的工具(本章 `root_agent` 没有自己的 `tools`,只有 `sub_agents`)之外,还会被自动附加一个内置工具,它的调用形状大致是"传入一个目标 agent 的名字"。当模型判断这轮对话应该交给 `order_agent` 处理时,它发起的其实是一次**标准的工具调用**——调用 `transfer_to_agent`,参数里带上 `agent_name="order_agent"`。ADK 的 runner 拦截到这次工具调用后,才真正把会话控制权切换到 `order_agent` 这个子 agent,由它去决定要不要调用 `get_order_status`/`cancel_order`、以及怎么组织最终回复。

这件事之所以重要,是因为**它把"路由"这个原本容易被当成框架魔法的环节,还原成了和普通工具调用完全一样的、可观测的一步**:在真实运行的 Cloud Trace 里,这次路由决策会体现为一个名字就叫 `execute_tool transfer_to_agent` 的 span——和 `execute_tool get_order_status` 这类你自己接入的工具调用,在可观测性层面是同一等级的东西,都能在 Trace 里看到入参、耗时、是否报错。这对 FDE 排查"为什么客户反馈 agent 该转给订单助手却没转"这类问题非常关键:你不需要靠猜,直接去 Trace 里看 `root_agent` 那一轮到底有没有发起 `transfer_to_agent` 调用、传的 `agent_name` 参数是不是符合预期。

!!! tip "根据经验判断的一点补充"
    转移发生之后,后续几轮对话如果仍然是同一个领域的请求(比如连着问了两个订单相关的问题),实践中我们观察到不需要每一轮都重新触发一次 `transfer_to_agent`——当前接管对话的子 agent 会继续处理,直到用户的意图明显切换到另一个领域。具体的会话保持策略属于 ADK 会话状态管理的实现细节,建议以你实际部署时在 Trace/日志里观察到的行为为准,而不是假设一个固定不变的规则。

## 2.4 真实运行效果

```text
[user]: 帮我查一下订单 1001 的状态
[order_agent]: 您的订单 1001 状态为已发货，预计 2 天后到达。

[user]: 那把 1002 取消掉吧
[order_agent]: 好的，已为您取消订单 1002。
```

打开这次请求对应的 Cloud Trace 详情页,调用链大致长这样(以下缩进结构只是示意,具体 span 的层级和命名请以你自己环境里实际看到的为准;但 `execute_tool transfer_to_agent` 这个 span 名字,是 ADK 底层路由机制的真实体现,这一点是确定可复现的):

```text
root_agent 处理这一轮请求
 └─ execute_tool transfer_to_agent  {"agent_name": "order_agent"}
     └─ order_agent 接管会话
         └─ execute_tool get_order_status  {"order_id": "1001"}
             └─ 实际发往 https://order-status-api-.../orders/1001 的 HTTP 请求
```

对 FDE 来说,这条链路的可观测性价值在于:客户反馈"agent 答非所问"或者"没查到订单"的时候,你不用靠猜,直接打开对应请求的 Trace,就能确认到底是路由环节出了问题(`root_agent` 根本没转给 `order_agent`,或者转给了错误的子 agent),还是工具调用环节出了问题(转对了,但 `get_order_status` 调用失败、或者传的 `order_id` 参数不对)。这两类问题的排查方向完全不同,而 Trace 能在几秒钟内帮你把它们区分开。

## 2.5 真实踩过的坑

!!! danger "servers 字段缺失或者指错地址"
    `OpenAPIToolset` 根据 `spec["servers"][0]["url"]` 拼接实际请求的 base URL——如果 FastAPI 没显式设置 `servers=[...]`,导出的 spec 里根本不带这个字段,工具调用会直接失败。这个参数容易被忽略,因为本地测试时 FastAPI 的交互文档(`/docs`)不需要这个字段也能正常显示,你很可能在联调阶段"接口文档看着一切正常",直到接进 agent 才第一次发现问题。养成习惯:接入 `OpenAPIToolset` 之前,先打开导出的 `openapi.json` 文件肉眼确认 `servers` 字段存在且地址正确,而不是只看 `/docs` 页面。

!!! warning "auto_error=True 情况下,缺失 Header 和 Header 错误返回的状态码不一样"
    `APIKeyHeader(name=API_KEY_NAME, auto_error=True)` 在请求**完全不带** `X-API-Key` 头时,会由 FastAPI 的安全工具类自身抛出 `403 Forbidden`;而当请求带了这个头、但值不对时,是走到我们自己写的 `verify_api_key` 函数,抛出的是 `401 Unauthorized`。这是两条不同的代码路径产生的两种状态码,如果你在给 agent 或者上层业务写错误处理逻辑时,想当然地认为"认证失败统一是 401",遇到 `403` 分支时可能会被漏判。写集成测试时,这两种场景（完全不带头 / 带了错的头）最好都覆盖到。

!!! warning "本地导出的 openapi.json 是快照,后端接口改了但没重新导出会悄悄错位"
    `agent.py` 是在模块加载时从磁盘读一次 `order_api/openapi.json`,而不是每次都实时向线上 API 请求最新的规范。这样做换来了 agent 启动速度和对后端可用性的解耦,但代价是:如果后端 API 后续新增了字段、改了某个参数类型,而没人记得重新 `curl` 导出、重新提交这份 `openapi.json`,agent 侧看到的工具 schema 就会和线上真实接口悄悄脱节。团队协作时,这份 `openapi.json` 最好当成一份需要跟着后端变更同步更新的"生成产物"来管理,而不是一次性写死的静态文件——比如在后端 API 的 CI 流程里加一步自动重新导出并提交这份文件。

## 2.6 OpenAPIToolset(以及 ADK 工具体系)还能做什么

ADK 里工具不止 `OpenAPIToolset` 一种,选型是个真实的架构决策,值得展开对比:

| 工具类型 | 工作方式 | 适合场景的具体例子 |
|---|---|---|
| `FunctionTool`(原生 Python 函数) | 直接把一个 Python 函数包装成工具,LLM 调用时在 agent 进程内同步执行,没有网络往返 | 比如一个 `calculate_late_fee(days_overdue, order_amount)` 这样纯计算、不需要访问任何外部系统的函数——延迟只取决于本地 CPU,不用为它单独走一次 HTTP 请求,也不需要维护一份 OpenAPI 规范 |
| `OpenAPIToolset` | 指向一份 OpenAPI/Swagger 规范,按 `operation_id` 自动批量生成工具 | 本章场景:**客户已经有一个文档化的 REST API**(哪怕只是一个 Mock/内部系统),不需要为每个接口手写一层 Python 包装函数,规范改了重新生成就行 |
| `McpToolset` | 连接一个独立部署的 MCP(Model Context Protocol)服务,通过标准协议(stdio/SSE/HTTP)获取工具定义并调用 | 比如客户的平台团队要维护一个"客户身份查询"工具,这个工具不仅要给这个 ADK agent 用,还要给另一个团队用 LangChain 写的内部工具、甚至要接入 Claude Desktop 这类外部客户端——把它做成一个独立的 MCP 服务,一次实现多处复用,比在每个消费方各写一份逻辑更值 |

**`FunctionTool`**、**`OpenAPIToolset`**、**`McpToolset`** 三者不是互斥关系,同一个 `LlmAgent` 的 `tools` 列表里完全可以混用——比如给 `order_agent` 再加一个纯本地的 `FunctionTool` 做订单号格式校验,校验通过了再调用 `OpenAPIToolset` 生成的 `get_order_status`,这在真实项目里是很常见的组合方式。

围绕本章用到的 `OpenAPIToolset`,还有几个值得展开的延伸能力:

- **`tool_filter`**:一份规范里可能有几十个接口,但某个 agent 只应该被允许调用其中几个。传入一个只包含目标 `operation_id` 的过滤列表,可以让 `OpenAPIToolset` 只生成被允许的那一部分工具,不是规范里所有接口都自动暴露给 LLM。这在客户的后端 API 本身覆盖面很广、但你只想让某个专职 agent 接触其中一小块能力时非常实用——既减少了模型选错工具的概率,也从工程上限制了这个 agent 能触发的操作范围。
- **`tool_name_prefix`**:如果一个 agent 同时接入多份不同的 OpenAPI 规范(比如订单系统 + 库存系统 + 物流系统各是一个独立的 `OpenAPIToolset`),不同规范里的 `operation_id` 有可能撞名(两边都定义了一个 `get_status`)。给每个 `OpenAPIToolset` 实例加一个前缀,可以在不改动任何一方后端代码的前提下,让生成的工具名各自独立、互不覆盖。
- **多种认证方式**:本章示例用的是最简单的 API Key(`token_to_scheme_credential("apikey", "header", ...)`),但客户的真实后端未必是这种认证方式。官方文档提到 ADK 的 `auth_helpers` 模块里还提供了面向 OAuth2(授权码/客户端凭证等流程,适合后端本身是接在客户已有的 OAuth2 身份提供方比如 Okta、Azure AD 之上的场景)、OIDC(基于 OpenID Connect 的身份令牌,适合后端校验的是符合 OIDC 标准的 ID Token)、以及 GCP 服务账号身份(适合后端本身就是另一个受 IAM 保护的 GCP 服务,比如一个没有开 `--allow-unauthenticated` 的 Cloud Run 服务,这时候用服务账号的身份令牌换取访问权限,比维护一个额外的应用层密钥更贴合 GCP 原生的访问控制模型)等构造方式,概念上都是"生成一对 `auth_scheme`/`auth_credential`"这同一套模式,只是背后对应的认证协议不同。
- **混合工具类型组合使用**:如前面提到的,一个 agent 的 `tools` 列表里可以同时有 `OpenAPIToolset` 生成的远程工具和手写的 `FunctionTool` 本地工具。真实项目里,"先用 `FunctionTool` 做一层输入校验或者业务规则预检查,再调用 `OpenAPIToolset` 生成的远程接口"是一个很实用的模式,可以在不改动客户后端代码的前提下,给这些接口的调用加上一层 agent 侧的业务护栏。

## 2.7 优化方向

**鉴权信息接入 Secret Manager,而不是硬编码在代码里。** 本章示例为了教学简化,直接把 `"secret-key-123"` 写在了 `agent.py` 和 `main.py` 里——这在真实项目里是不可接受的,密钥一旦进了 git 历史就很难彻底清除。概念上的改法是把 `token_to_scheme_credential` 里那个密钥值,换成从 Secret Manager 实时读取:

```bash
# 创建密钥(内容从标准输入读入,不会出现在 shell 历史里更安全的做法是从文件读)
echo -n "secret-key-123" | gcloud secrets create order-api-key --data-file=-

# 只给 agent 实际运行时用的服务账号授予"读取密钥值"这一个最小权限
gcloud secrets add-iam-policy-binding order-api-key \
  --member="serviceAccount:AGENT_RUNTIME_SA@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

```python
from google.cloud import secretmanager

def get_secret(project_id: str, secret_id: str, version: str = "latest") -> str:
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_id}/versions/{version}"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("UTF-8")

# 原来是硬编码的 "secret-key-123",现在改成运行时从 Secret Manager 读取
_order_api_key = get_secret(os.environ["GCP_PROJECT"], "order-api-key")

_auth_scheme, _auth_credential = token_to_scheme_credential(
    "apikey", "header", "X-API-Key", _order_api_key
)
```

这个改法之所以能干净地嵌进已有代码,正是因为 2.3 节提到的设计——`token_to_scheme_credential` 本来就把"认证方案的形状"和"具体密钥值"拆成了两个独立的输入,需要替换的只是密钥值来源,不需要动 `OpenAPIToolset` 的其它配置。授权时只给运行时服务账号绑定 `roles/secretmanager.secretAccessor` 这一个角色,而不是项目级别的更宽泛权限,是最小权限原则在这里的具体体现。

**给工具调用配置超时和重试策略。** 客户的 API 不一定稳定,agent 调用工具时如果没有超时设置,一次慢请求可能拖垮整条对话链路,让用户盯着一个没有响应的对话框等很久。实践中值得明确配置的几个点:

- 在实际发起 HTTP 请求的客户端层面(`OpenAPIToolset` 底层最终依赖某种 HTTP 客户端库发起请求),设置好连接超时和读取超时,让一次异常慢的请求快速失败,而不是无限期挂起整个 agent 回合。
- 只对瞬时性错误(5xx 服务端错误、429 限流)做指数退避重试,不要对 4xx 客户端错误(比如 404 订单不存在、401 认证失败)做重试——这些错误重试再多次结果也不会变,徒增延迟。
- 限制重试的总次数和总耗时上限(比如最多重试 2-3 次,总耗时不超过几秒),避免一个不稳定的下游服务把 agent 的响应时间拖到用户无法接受的程度。

!!! warning "重试要考虑操作是否幂等"
    `cancel_order` 是一个会修改后端状态的 `POST` 接口。这次示例里重复调用它是安全的(订单一旦被标记为 `cancelled`,再调一次结果不变),但真实客户的接口未必都设计成幂等的——如果某个 `POST` 接口每次调用都会产生新的副作用(比如每次都触发一条新的退款流水),对它做自动重试就有可能造成重复扣款、重复下单这类真实的业务事故。给工具调用加重试策略之前,先确认对应接口的幂等性,或者要求客户后端为关键的写操作补上幂等键(idempotency key)机制。

**评估什么时候应该把工具升级到 MCP。** `OpenAPIToolset` 对本章这种"一个客户 API、被一个 agent 使用"的场景已经足够轻量、足够快。但当出现下面这些信号时,提前规划成独立部署的 MCP 服务通常会比事后重构轻松得多:

- 同一套工具逻辑需要被不止一个 agent 框架或者不止一个团队的代码库调用——继续各自维护一份 `OpenAPIToolset` 包装,意味着同样的认证配置、错误处理逻辑要在多处重复维护,一处改了容易忘记同步另一处。
- 这个工具需要独立于任何单个 agent 的部署节奏,单独做版本管理、单独做访问审计和限流——把它拆成一个独立的 MCP 服务,可以有自己的发布周期,不用绑定某一个 agent 项目的上线时间表。
- 需要对接的消费方本身就是支持 MCP 协议的外部客户端(比如 Claude Desktop 或者其他厂商的 agent 运行时),而不只是你自己维护的这一个 ADK 项目——这种情况下重新用各家框架各自的工具包装方式,不如直接暴露一个标准的 MCP 服务来得通用。
- 工具背后要接入的后端系统数量和复杂度持续增长(不再是一个订单系统,而是订单、库存、物流、工单等一整套系统),继续以"每个 agent 各自维护一堆 `OpenAPIToolset` 实例"的方式管理,会让每个 agent 项目的配置越来越臃肿,这时候把工具层整体收敛到一个统一的 MCP 服务,能把认证、可观测性这些横切关注点集中处理一次。
