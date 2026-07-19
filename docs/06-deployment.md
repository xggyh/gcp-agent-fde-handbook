# 第 6 章 · 部署上线

## 6.1 这一步要解决什么问题

前几章都是本地跑 `adk web`/`adk run` 调试,这一章要把 agent 变成一个客户能真正调用的、托管在云上的服务——同时要解决一个容易被忽略的架构问题:**本地依赖的 mock API 也得先部署到云端**,不然部署后的 agent 连不到它。

这个问题之所以"容易被忽略",是因为它在本地开发的每一个环节都表现得完全正常:

- 本地跑 `adk web` 的时候,ADK 的开发服务器和 Mock API(比如订单查询服务)都在同一台机器上,agent 代码里写的 `http://127.0.0.1:8000` 指向的就是"我自己"这台电脑,请求畅通无阻。
- 单元测试、集成测试、给同事做 demo 演示——只要这些场景都在同一台开发机上完成,`127.0.0.1` 这个地址永远有效,整条链路看起来天衣无缝。
- 真正的坑出现在"部署"这一步:一旦把 agent 部署到 Agent Runtime,它运行的地方变成了 Google 管理的托管容器,和你的笔记本电脑物理上完全是两个不同的网络环境。这时候 agent 代码里那个 `127.0.0.1:8000` 依然会被解析,但它现在指向的是 **agent 自己所在的容器**,而不是开发者的电脑——那个容器上根本没有监听 8000 端口的服务,于是所有请求都会连接失败或者超时。

更麻烦的是,这类问题暴露出来的报错信息往往很含糊——连接超时、连接被拒绝——第一反应容易去怀疑是不是权限没配对、代码是不是有 bug,而不会立刻想到"是网络拓扑变了"。等排查了一圈发现是这个原因,往往已经浪费了不少时间。

!!! note "这个教训在真实客户项目里非常普遍"
    这不是本章 Mock API 这个个例才有的问题,而是 agent 从"本地开发"迈向"云端部署"这一步几乎必然会撞上的一类架构问题。在真实客户项目里,agent 要调用的下游依赖几乎从来不会和 agent 部署在同一台机器上——它们可能是客户的 CRM、工单系统、订单系统,这些系统本身往往还部署在客户的私有网络(VPC、专线、防火墙后面)里。

    核心原则是:**agent 一旦部署,它的网络位置就变了,所有关于下游依赖地址的假设都需要重新审视**——不能假设开发环境里能连通的网络拓扑,在生产环境里依然成立。FDE 在做客户项目排期时,经常会低估"把测试桩/下游依赖打通网络可达性"这一步的工作量,因为它看起来像是"顺便的事",但如果客户的下游系统本身还没有一个云端/公网可达的测试环境,FDE 往往得先帮客户搭一个能被 agent 访问到的接口(哪怕只是一个部署在 Cloud Run 上的 mock/代理),才能验证整条链路真的能跑通。这一步不做,后面所有关于 agent 效果的验证都建立在沙上。

## 6.2 在 GCP 上具体怎么做

### 前置工作:把依赖的 API 也部署上云

本章的订单查询工具连的是 `http://127.0.0.1:8000`——这是本机地址,部署到 Agent Runtime 之后,托管环境根本访问不到开发者自己的电脑。所以要先把这个 Mock API 部署到 Cloud Run:

```bash
gcloud run deploy order-status-api \
  --source=. \
  --region=us-central1 \
  --allow-unauthenticated \
  --port=8080
```

把每个 flag 拆开看:

- **`--source=.`**:告诉 Cloud Run 直接从当前目录的源码构建镜像,不需要开发者自己写 Dockerfile。背后走的是 **Buildpacks** 自动构建流程——Cloud Build 会先探测目录里的语言特征(比如有没有 `requirements.txt`/`pyproject.toml`),自动选择合适的运行时基础镜像,把依赖装好、把应用打包成容器镜像,再推送到 Artifact Registry,整个过程对开发者是黑盒的。这对 Mock API 这种小工具服务非常合适——没必要为了一个几十行代码的服务专门维护一份 Dockerfile。
- **`--region=us-central1`**:决定这个 Cloud Run 服务部署在哪个区域。这里特意和后面 Agent Runtime 部署用的区域保持一致,一方面降低跨区域调用的延迟,另一方面很多 GCP 资源之间的访问(比如后续如果要接 VPC 网络)本身也有同区域的便利性。
- **`--allow-unauthenticated`**:允许任何人在不携带 GCP 身份令牌的情况下直接访问这个服务的公网 URL。也就是说,只要知道这个 URL,任何人都能调用它——这是为了在本章的实验环境里省掉认证配置的麻烦,让 agent 能直接用一个裸 URL 访问到 Mock API。
- **`--port=8080`**:告诉 Cloud Run 容器内部实际监听的端口。Cloud Run 默认假设容器监听 8080 端口,如果 Mock API(比如用 `uvicorn`/`fastapi` 起的服务)监听的是别的端口,就需要显式声明,否则 Cloud Run 的请求路由不到容器内的进程,服务会一直显示"未就绪"。

!!! warning "--allow-unauthenticated 在生产环境的取舍"
    本章为了简化演示,直接把 Mock API 开放成公网可匿名访问,这在生产环境里是不可接受的——任何人拿到 URL 就能调用你的下游服务,没有任何访问控制。真实客户项目里,这一步至少要做以下两者之一:

    - 去掉 `--allow-unauthenticated`,改成默认的"需要 IAM 身份"模式,再通过 `gcloud run services add-iam-policy-binding` 把调用权限(`roles/run.invoker`)只授予 agent 运行时用的服务账号,做到"只有我的 agent 能调这个服务";
    - 或者在服务前面套一层认证层(API Key、OAuth2、mTLS 等),按客户对接系统本身的安全要求来定,而不是简单地"开放访问"。

部署完拿到形如 `https://order-status-api-xxx.us-central1.run.app` 的地址,回去把 agent 代码里硬编码的 `servers` URL 改成这个——这一步很容易漏掉,因为如果不改,agent 依然会去连本地地址,表现和"完全没部署 Mock API"一模一样,容易误导排查方向。

### 部署 agent 本体到 Agent Runtime

```bash
adk deploy agent_engine \
  --project=adk-fde-lab \
  --region=us-central1 \
  --staging_bucket=gs://adk-fde-lab-staging \
  --otel_to_cloud \
  ./customer_service_agent
```

同样把关键 flag 拆开讲清楚:

- **`--staging_bucket=gs://adk-fde-lab-staging`**:Agent Runtime 的部署流程不是直接把本地代码"推"进某个运行时,而是先把 agent 的代码、依赖打包,上传到这个 GCS bucket 里作为**中转存储**,然后 Vertex AI 服务端再从这个 bucket 里把包拉过去,构建出真正运行的 agent 环境。这个 bucket 需要提前创建好,并且执行部署的身份(通常是你本地 `gcloud auth` 登录的账号,或者 CI/CD 里用的服务账号)必须对这个 bucket 有写权限,否则部署会在上传阶段就失败。
- **`--otel_to_cloud`**:开启 OpenTelemetry 数据自动导出到 Cloud Trace / Cloud Logging。为什么这个参数要在**部署时**就带上,而不是部署完之后再补?因为可观测性的 exporter 是在 agent 运行时初始化的阶段就要接好线的——它决定了这个 agent 实例内部,每一次工具调用、每一次模型调用产生的 trace/span 是否会被采集并推送到 Cloud 端。这不是一个运行期间可以动态打开的开关,如果部署时没带这个 flag,之后想要补上可观测性,通常意味着要重新走一次部署,而不是"调个配置项"就行。带上它之后,才能在后面 6.5 节遇到"客户端挂起没有异常"这种问题时,第一时间去 Cloud Logging/Cloud Trace 里查到服务端真实发生了什么。

拿到的资源名形如:

```text
projects/PROJECT_NUMBER/locations/us-central1/reasoningEngines/RESOURCE_ID
```

!!! note "改名提醒"
    2026 年 4 月起,"Agent Engine" 改名为 **"Agent Runtime"**,底层的 API 资源类型没变,还是 `reasoningEngines`。本章沿用真实项目里的命令和资源命名,`adk deploy agent_engine` 这个子命令名本身也还没有跟着改名同步调整。

## 6.3 核心代码:调用部署好的 agent

部署完成后,真实项目里用来跟这个远程 agent 对话的是一个可以持续多轮交互的命令行小工具 `talk_to_agent.py`:

```python
import vertexai
from vertexai import agent_engines

# 初始化 Vertex AI SDK:声明接下来所有操作对应哪个项目、哪个区域。
# project/location 必须和部署 agent 时用的完全一致,否则下面 get() 会找不到资源。
vertexai.init(project="adk-fde-lab", location="us-central1")

# 通过完整资源名拿到这个已部署 agent 的远程引用。
# 资源名结构是 projects/{PROJECT_NUMBER}/locations/{LOCATION}/reasoningEngines/{RESOURCE_ID}——
# 注意这里用的是数字形式的 PROJECT_NUMBER(372239021300),而不是项目别名 adk-fde-lab。
remote_agent = agent_engines.get(
    "projects/372239021300/locations/us-central1/reasoningEngines/2487818230924574720"
)

# 创建一个会话(session)。多轮对话的上下文全靠这个 session["id"] 维持——
# 后面每一次 stream_query 调用只要带上同一个 session_id,
# agent 就能记住这轮对话之前说过什么、查过哪个订单;
# 换一个新的 session_id,等于开启一段全新的、不带历史上下文的对话。
session = remote_agent.create_session(user_id="my_user")

while True:
    text = input("你: ")
    if text.strip().lower() == "exit":
        break
    # stream_query 返回的是一个生成器(流式接口),会逐个产出服务端生成的 event。
    # 一次用户输入背后,agent 内部可能经过好几轮"模型推理 -> 调用工具 -> 再推理"的循环,
    # 每一轮都可能产生一个或多个 event,而不是等全部处理完才一次性返回。
    for event in remote_agent.stream_query(
        user_id="my_user", session_id=session["id"], message=text
    ):
        content = event.get("content", {})
        for part in content.get("parts", []):
            # 一个 part 不一定是纯文本——它也可能是模型发起的函数调用(function_call)
            # 或者工具返回的结果(function_response)这类中间态。
            # 这里只挑出带 "text" 字段的 part 打印出来,过滤掉工具调用的原始结构化内容。
            if "text" in part:
                print(f"[{event.get('author')}]: {part['text']}")
```

几个值得展开的点:

- **event 的结构**:`stream_query` 每次 yield 出来的 `event` 大致长这样——一个字典,里面有 `content`(本轮说了什么)、`author`(这句话是谁说的,可能是 agent 本体,也可能是某个 sub_agent 或工具执行结果)等字段。`content.parts` 是一个列表,因为一轮输出可能同时包含多段内容——既有模型对用户说的自然语言文本,也可能夹杂着中间的函数调用/函数返回。示例代码里 `if "text" in part` 这一判断,就是在一堆可能的 part 类型里,只把"说给用户听的那部分"挑出来打印,工具调用本身的 JSON 结构不会直接展示给用户。
- **session 的作用**:`create_session` 拿到的 `session["id"]` 是维持多轮对话上下文的关键。它不是一个客户端本地维护的变量,而是服务端(Agent Runtime 内置的 `VertexAiSessionService`)真实持久化的一个会话对象——同一个 `session_id` 下的历史消息、工具调用记录都会被服务端记住,agent 在处理新一轮输入时能带着这些历史一起推理。这也是为什么示例代码把 `session` 创建放在 `while True` 循环外面:只创建一次,后面所有轮次的对话都复用同一个 `session_id`,才能让"帮我查订单1001"和后面追问"那什么时候能到"这种指代关系被正确理解。

## 6.4 三种给客户/最终用户使用的方式

1. **GCP 控制台 Playground**:部署成功后控制台会给一个网页聊天框链接,不用写代码,适合快速演示,也适合非工程背景的客户干系人自己上手试用。
2. **Python SDK**,也就是上面完整展示的 `talk_to_agent.py`——适合写自动化脚本、集成进已有 Python 系统,或者作为 FDE 自己验证部署是否成功的排障工具(比 Playground 更方便快速地复现一个具体的输入序列)。
3. **REST API**,任何语言都能调,适合客户系统本身不是 Python 技术栈、需要从 Java/Node/Go 等服务里发起调用的场景:

```bash
TOKEN=$(gcloud auth print-access-token)
curl -X POST -H "Authorization: Bearer $TOKEN" \
  "https://us-central1-aiplatform.googleapis.com/v1/projects/PROJECT_ID/locations/us-central1/reasoningEngines/RESOURCE_ID:streamQuery" \
  -d '{"class_method": "stream_query", "input": {"user_id": "someone", "message": "帮我查订单1001"}}'
```

三种方式背后调的是同一个部署好的 `reasoningEngine` 资源,区别只在接入层——选哪种取决于调用方是人(Playground)、Python 脚本(SDK)、还是其他语言的后端服务(REST)。

## 6.5 真实踩过的坑

这一章的坑基本全是 **IAM 权限缺口**,而且都是"新项目默认权限比想象中更少"这一类:

| 报错 | 根因 |
|---|---|
| `PERMISSION_DENIED: Build failed because the default service account is missing required IAM permissions` | Google 在某个时间点后不再默认给项目的计算服务账号自动授予 Editor 角色,导致 Cloud Build 读不到刚上传的源码包 |
| 部署后 `policy_agent` 挂起没有任何报错,180 秒后客户端超时才暴露真正的服务端 403 | Agent Runtime 的默认运行时身份(`service-PROJECT_NUMBER@gcp-sa-aiplatform-re.iam.gserviceaccount.com`)只有最基础的 `reasoningEngineServiceAgent` 角色,没有 `aiplatform.ragCorpora.query` 权限 |

第二个坑值得单独展开讲一下,因为它背后反映的是这类流式接口本身的架构特点,而不是一次性的偶然故障:

!!! danger "为什么"客户端挂起没有异常"是这类接口的常态,而不是意外"
    `stream_query` 本质上是一个**流式/长轮询接口**,不是简单的一次性 HTTP 请求-响应模型。客户端发起调用之后,连接会保持打开,等待服务端陆续推送 event——这意味着服务端在处理过程中,即便内部已经抛出了异常(比如这里的"没有权限查询 RAG 语料库"),这个异常也**不会立刻通过一个明确的 HTTP 状态码传导给客户端**。客户端能感知到的现象,往往只是"迟迟没有新的 event 推送过来",直到客户端自己的超时机制(这里是 180 秒)触发,才会抛出一个超时相关的异常。

    问题在于,这个最终抛出的"客户端超时"异常,跟真正的根因(服务端的 IAM 403)几乎没有直接关联——如果只盯着客户端的报错信息去排查,很容易被引导去怀疑网络问题、SDK 版本问题,而不是第一时间想到"服务端权限不够"。

    所以一个更高效的通用排障顺序是:**一旦发现调用挂起超过合理时间,第一时间去查服务端日志,而不是继续等客户端把超时跑完**。具体做法是去 Cloud Logging 里按 `resource.type="aiplatform.googleapis.com/ReasoningEngine"` 过滤,服务端的真实 traceback(包括这里的 403 权限拒绝详情)基本都在里面,比干等客户端超时快得多——这也是前面部署时坚持带上 `--otel_to_cloud` 的直接收益,没有这个开关,服务端日志可能根本没有导出到你能查到的地方。

补权限:

```bash
gcloud projects add-iam-policy-binding adk-fde-lab \
  --member="serviceAccount:service-PROJECT_NUMBER@gcp-sa-aiplatform-re.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user"
```

## 6.6 Agent Runtime / Cloud Run 还能做什么

Agent Runtime 除了本章用到的"部署 + 流式调用"这条最基本路径,官方文档提到还有几个值得在真实项目里用上的能力:

- **会话持久化(`VertexAiSessionService`)**:内置托管的会话存储,前面 `create_session`/`session_id` 这套机制背后就是它——不需要自己再搭一套数据库或者 Redis 来存对话历史,一个 session 内的短期上下文由它自动维护。
- **跨会话记忆(Memory Bank)**:这是和 session 内的短期记忆**完全不同的一层**。session 维持的是"这一次对话"内的上下文——一旦这次对话结束或者客户端换了一个新的 `session_id`,这些细节默认不会带到下一次会话里。Memory Bank 解决的是"用户下次带着一个全新 session 回来,agent 还能不能认得他"这个问题——官方文档提到它可以从历史会话里提炼、沉淀出关于某个用户的长期信息(比如偏好、过往交互的要点摘要),即便开了一个全新的 session,agent 也能取用这些跨会话积累下来的记忆。这在多次造访的客服场景里很有价值:用户几天后再回来问同一个订单的后续进展,不需要每次都从头自我介绍一遍背景。
- **长时间运行的操作**:官方文档提到 Agent Runtime 支持较长时间跨度的运行(可达数天级别),适合那些内部需要长时间等待外部系统响应、或者需要跑异步长任务的 agent,不需要自己额外搭一套任务队列/轮询机制来绕开传统请求-响应模型的超时限制。
- **内置可观测性集成**:结合部署时的 `--otel_to_cloud`,Agent Runtime 能把内部的调用链路自动导出到 Cloud Trace 和 Cloud Logging,不需要自己在 agent 代码里手动打点接入监控栈,这也是 6.5 节排障能够走"先查服务端日志"这条路的前提。
- **和 IAM 深度集成的访问控制**:调用一个部署好的 reasoningEngine 本身走的是标准的 GCP IAM 鉴权,可以通过 IAM 角色/policy 精确控制"谁能调用这个 agent",不需要在 agent 应用层自己再写一套认证逻辑。

再往外看,agent 的部署目标不是只有 Agent Runtime 一个选择:

- **Cloud Run 部署 agent 本体**(`adk deploy cloud_run`):除了本章用它托管 Mock API,Cloud Run 本身也是部署 agent 的备选路径。什么时候更适合走这条路?当团队需要对 Web 服务层有更多底层控制的时候——比如要接入自定义的鉴权中间件、自定义路由规则、自定义的 Web UI,或者团队本身已经有一整套围绕 Cloud Run 的运维体系(监控、CI/CD、流量管理习惯),希望 agent 也纳入同一套模型来管理,这时候 Cloud Run 比 Agent Runtime"开箱即用但灵活度较低"的定位更合适。
- **GKE 部署路径**:当规模和复杂度上升到——需要跑非常大规模的并发、多个模型/多个 agent 混合编排、有状态服务和批处理任务混在一起需要精细调度,或者客户本身已经有成熟的 Kubernetes 运维体系、要求所有服务统一纳入 K8s 管理——这些场景下 GKE 能提供最大的灵活度(自定义调度策略、精细的资源隔离、和已有微服务体系统一治理)。代价是运维成本也最高:集群本身的搭建、网络配置、扩缩容策略都需要团队自己维护,不像 Agent Runtime 那样"托管即用"。

## 6.7 优化方向

- **CI/CD 化部署流程**。本章的部署是手动敲命令,生产场景应该接入 CI/CD(比如 Cloud Build 或 GitHub Actions)。概念上的流水线大致是这样一串阶段:

    1. **测试**:代码提交/合并触发流水线,先跑单元测试和针对 Mock API(或者对接的下游依赖)的集成测试,失败就直接终止,不进入后续阶段。
    2. **构建**:分别为 agent 代码和它依赖的下游服务(比如本章的 Mock API)触发构建——延续本章用到的 `--source=.` 这类 Buildpacks 自动构建方式,产出可部署的镜像/包。
    3. **部署到 staging**:用一个独立的 staging 项目或者至少独立的 `--staging_bucket`/资源,把这一版代码部署上去,不直接碰生产资源。
    4. **冒烟测试**:自动化脚本对着 staging 环境跑几条预设好的测试消息(思路上类似 `talk_to_agent.py`,但改成非交互、批量断言输出是否符合预期),验证核心链路——包括跟 Mock API 的联通性——是否正常,再决定要不要往下走。
    5. **部署到生产**:冒烟测试全部通过后,再把同一份构建产物部署到生产的 Agent Runtime 资源,是否需要人工审批取决于变更的风险等级。

    这一串阶段的核心价值在于:把 6.1 节提到的"下游依赖网络可达性"这类问题,尽量前移到 staging 阶段的冒烟测试里暴露,而不是等到生产环境让客户遇到。

- **多环境隔离**(dev/staging/prod 用不同的 GCP 项目,或者至少不同的 `--staging_bucket`/`reasoningEngines` 资源),避免在生产项目里直接调试、避免开发阶段的试验性变更影响到客户正在使用的 agent。

- **灰度发布**。Agent Runtime 目前的部署模型是每次部署产生一个独立的 `reasoningEngine` 资源 ID,并不像 Cloud Run 那样原生支持给同一个服务的不同 revision 按百分比分配流量。所以更现实的灰度做法是在应用层/网关层维护多版本资源、自己实现流量切分:同时保留旧版本和新版本各自的资源 ID,在调用方(比如客户自己的后端服务,或者一层薄的路由层)按某种规则——例如按 `user_id` 哈希分桶,或者一个显式的 feature flag 服务——把一小部分流量先路由到新版本的 resource ID,观察 Cloud Logging/Cloud Trace 里的指标和报错率稳定之后,再逐步扩大新版本的流量比例,最终把旧版本资源下线。如果 agent 走的是 Cloud Run 部署路径,则可以直接借助 Cloud Run 原生支持的按 revision 分配流量百分比的能力(`gcloud run services update-traffic`),做法上比 Agent Runtime 现阶段要更成熟一些。

- **给 Mock API 换成客户真实系统时的鉴权收紧**。本章为了演示用了 `--allow-unauthenticated` 直接开放访问,真实客户系统对接时,这一步必须重新设计:去掉匿名访问,按客户的安全要求接入 IAM invoker 权限控制、mTLS、OAuth2、或者 IP 白名单等机制,并且要提前跟客户确认清楚他们的下游系统本身能不能接受被一个云端 agent 直接访问、需要走什么样的网络路径(公网、VPC 对等连接、Private Service Connect 等)。
