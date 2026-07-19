# 第 6 章 · 部署上线

## 6.1 这一步要解决什么问题

前几章都是本地跑 `adk web`/`adk run` 调试,这一章要把 agent 变成一个客户能真正调用的、托管在云上的服务——同时要解决一个容易被忽略的架构问题:**本地依赖的 mock API 也得先部署到云端**,不然部署后的 agent 连不到它。

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

部署完拿到形如 `https://order-status-api-xxx.us-central1.run.app` 的地址,回去把 agent 代码里硬编码的 `servers` URL 改成这个。

### 部署 agent 本体到 Agent Runtime

```bash
adk deploy agent_engine \
  --project=adk-fde-lab \
  --region=us-central1 \
  --staging_bucket=gs://adk-fde-lab-staging \
  --otel_to_cloud \
  ./customer_service_agent
```

拿到的资源名形如:

```text
projects/PROJECT_NUMBER/locations/us-central1/reasoningEngines/RESOURCE_ID
```

!!! note "改名提醒"
    2026 年 4 月起,"Agent Engine" 改名为 **"Agent Runtime"**,底层的 API 资源类型没变,还是 `reasoningEngines`。

## 6.3 核心代码:调用部署好的 agent

```python
import vertexai
from vertexai import agent_engines

vertexai.init(project="adk-fde-lab", location="us-central1")
remote_agent = agent_engines.get(
    "projects/PROJECT_NUMBER/locations/us-central1/reasoningEngines/RESOURCE_ID"
)

session = remote_agent.create_session(user_id="some_user")
for event in remote_agent.stream_query(
    user_id="some_user", session_id=session["id"], message="帮我查订单1001"
):
    for part in event.get("content", {}).get("parts", []):
        if "text" in part:
            print(part["text"])
```

## 6.4 三种给客户/最终用户使用的方式

1. **GCP 控制台 Playground**:部署成功后控制台会给一个网页聊天框链接,不用写代码,适合快速演示。
2. **Python SDK**(如上),适合写自动化脚本、集成进已有 Python 系统。
3. **REST API**,任何语言都能调:

```bash
TOKEN=$(gcloud auth print-access-token)
curl -X POST -H "Authorization: Bearer $TOKEN" \
  "https://us-central1-aiplatform.googleapis.com/v1/projects/PROJECT_ID/locations/us-central1/reasoningEngines/RESOURCE_ID:streamQuery" \
  -d '{"class_method": "stream_query", "input": {"user_id": "someone", "message": "帮我查订单1001"}}'
```

## 6.5 真实踩过的坑

这一章的坑基本全是 **IAM 权限缺口**,而且都是"新项目默认权限比想象中更少"这一类:

| 报错 | 根因 |
|---|---|
| `PERMISSION_DENIED: Build failed because the default service account is missing required IAM permissions` | Google 在某个时间点后不再默认给项目的计算服务账号自动授予 Editor 角色,导致 Cloud Build 读不到刚上传的源码包 |
| 部署后 `policy_agent` 挂起没有任何报错,180 秒后客户端超时才暴露真正的服务端 403 | Agent Runtime 的默认运行时身份(`service-PROJECT_NUMBER@gcp-sa-aiplatform-re.iam.gserviceaccount.com`)只有最基础的 `reasoningEngineServiceAgent` 角色,没有 `aiplatform.ragCorpora.query` 权限 |

**一个通用排障技巧**:客户端调用挂起、没有任何异常抛出时,不要在客户端一直等——直接去 Cloud Logging 里按 `resource.type="aiplatform.googleapis.com/ReasoningEngine"` 过滤,服务端的真实 traceback 都在那,比等客户端超时快得多。

补权限:

```bash
gcloud projects add-iam-policy-binding adk-fde-lab \
  --member="serviceAccount:service-PROJECT_NUMBER@gcp-sa-aiplatform-re.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user"
```

## 6.6 Agent Runtime / Cloud Run 还能做什么

- **Agent Runtime**:内置会话持久化(`VertexAiSessionService`)和跨会话记忆(Memory Bank),支持长时间运行的操作(官方文档提到可达 7 天),不需要自己另外搭一套会话存储。
- **Cloud Run**:除了托管这里的 Mock API,本身也是部署 agent 的备选目标之一(`adk deploy cloud_run`)——适合想要更多底层控制(自定义 Web UI、自定义路由)的场景,Agent Runtime 更"开箱即用"。
- **GKE**:如果需要跑非常大规模、多模型混合、复杂编排的 agent 系统,GKE 提供最大的灵活度,但运维成本也最高。

## 6.7 优化方向

- **CI/CD 化部署流程**。本章的部署是手动跑命令,生产场景应该接入 CI/CD(比如 Cloud Build 或 GitHub Actions),代码合并到主分支自动跑测试、自动部署,而不是本地手敲命令。
- **多环境隔离**(dev/staging/prod 用不同的 GCP 项目或至少不同的资源),避免在生产项目里直接调试。
- **灰度发布**。Agent Runtime 支持多个部署版本,生产场景应该考虑先给一小部分流量验证新版本,而不是一次性全量切换。
- **给 Mock API 换成客户真实系统时的鉴权收紧**。本章为了演示用了写死的 API Key,真实客户系统对接需要按客户的安全要求(mTLS、OAuth2、IP 白名单等)重新设计认证方案。
