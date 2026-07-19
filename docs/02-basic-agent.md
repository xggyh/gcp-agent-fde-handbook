# 第 2 章 · 基础 Agent 与工具集成

## 2.1 这一步要解决什么问题

FDE 工作里最常见的场景就是这个:**客户已经有一套现成的业务系统(REST API),需要让 agent 能调用它**。这一章要做两件事:搭出第一个能跑的 ADK agent,并且把客户的 API(这里用一个 Mock 的订单查询系统模拟)包装成 agent 能用的工具。

## 2.2 在 GCP 上具体怎么做

### ADK 期望的项目结构

```text
adk_fde_project/                    <- adk web/run 从这里启动
└── customer_service_agent/         <- 这个文件夹名字就是agent的包名
    ├── __init__.py                 <- 必须写: from . import agent
    ├── agent.py                    <- 必须定义模块级的 root_agent
    └── .env
```

`__init__.py` 不是可以省略的样板文件——如果它不存在或者没有 `from . import agent`,ADK 的自动发现机制会直接跳过这个文件夹,不会报错,只是"什么都没找到"。

### 工具从哪来:OpenAPI 规范

客户的 API 只要能生成 OpenAPI(Swagger)规范,就可以用 `OpenAPIToolset` **自动**生成工具,不用一个接口一个接口手写包装函数:

```python
from fastapi import FastAPI, Security
from fastapi.security.api_key import APIKeyHeader

app = FastAPI(
    servers=[{"url": "http://127.0.0.1:8000"}],  # 关键:决定生成的spec里base URL是什么
)

@app.get(
    "/orders/{order_id}",
    operation_id="get_order_status",   # 决定生成的工具名叫什么
    summary="Look up the status of an order",
    dependencies=[Security(APIKeyHeader(name="X-API-Key"))],
)
def get_order_status(order_id: str):
    ...
```

跑起来后导出规范:

```bash
curl http://127.0.0.1:8000/openapi.json -o openapi.json
```

## 2.3 核心代码

```python
import json
from google.adk.agents import LlmAgent
from google.adk.tools.openapi_tool.auth.auth_helpers import token_to_scheme_credential
from google.adk.tools.openapi_tool.openapi_spec_parser.openapi_toolset import OpenAPIToolset

with open("order_api/openapi.json") as f:
    _order_spec = json.load(f)

# 认证信息:spec里只能描述"这里需要一个API Key",真正的密钥值必须单独传
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
    model="gemini-2.5-flash",
    instruction="You help users check the status of their orders...",
    tools=[order_api_toolset],
)

root_agent = LlmAgent(
    name="root_agent",
    model="gemini-2.5-flash",
    instruction="If the user asks about order status, delegate to order_agent...",
    sub_agents=[order_agent],
)
```

`sub_agents=[...]` 是 ADK 的多智能体分派机制——不需要自己写路由逻辑,`root_agent` 的模型会根据每个子 agent 的 `description` 自动判断该转给谁。

## 2.4 真实运行效果

```text
[user]: 帮我查一下订单 1001 的状态
[order_agent]: 您的订单 1001 状态为已发货，预计 2 天后到达。
```

## 2.5 真实踩过的坑

`OpenAPIToolset` 根据 `spec["servers"][0]["url"]` 拼接实际请求的 base URL——如果 FastAPI 没显式设置 `servers=[...]`,导出的 spec 里根本不带这个字段,工具调用会失败。这个参数容易被忽略,因为本地测试时 FastAPI 的交互文档(`/docs`)不需要这个字段也能正常显示。

## 2.6 OpenAPIToolset(以及 ADK 工具体系)还能做什么

ADK 里工具不止 `OpenAPIToolset` 一种,选型是个真实的架构决策:

| 工具类型 | 适合场景 |
|---|---|
| `FunctionTool`(原生 Python 函数) | 简单逻辑、和 agent 强耦合、对延迟敏感 |
| `OpenAPIToolset` | **客户已有文档化的 REST API**(本章场景) |
| `McpToolset` | 工具需要被多个不同 agent/框架复用,值得为此多付出一层部署成本 |

`OpenAPIToolset` 本身还支持:

- `tool_filter`:只暴露 spec 里的部分接口,不是全部自动开放
- `tool_name_prefix`:同时接入多个不同的 API spec 时避免工具名冲突
- OAuth2/OIDC/Google Service Account 等认证方式,不只是 API Key

## 2.7 优化方向

- **鉴权信息不要硬编码在代码里**(本章示例为了教学简化直接写了 `"secret-key-123"`)。生产环境应该用 [Secret Manager](https://cloud.google.com/security/products/secret-manager) 存储真实密钥,代码里只读引用。
- **给工具加超时和重试策略**。客户的 API 不一定稳定,agent 调用工具时如果没有超时设置,一次慢请求可能拖垮整条对话链路。
- **多个客户 API 的场景下,评估要不要升级到 MCP**。如果这个订单查询工具将来要被另一个团队做的另一个 agent 复用,提前规划成独立部署的 MCP 服务会比事后重构轻松得多(详见此手册对应主题的进一步讨论)。
