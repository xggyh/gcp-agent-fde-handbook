# 第 5 章 · 可观测性(Trace / Logging)

## 5.1 这一步要解决什么问题

Agent 出问题时(回答错了、卡住了、慢了),需要能追溯"这次对话到底经历了哪些步骤、每一步耗时多少、每个工具调用传了什么参数"——这就是可观测性存在的意义,而且客户方通常会明确要求"每次对话可审计"这条合规底线。

## 5.2 在 GCP 上具体怎么做

### 两个 CLI 参数,选新的那个

ADK 有两个开关:

| 参数 | 覆盖范围 | 状态 |
|---|---|---|
| `--trace_to_cloud` | 只导出 Cloud Trace | 较老的路径,在 `agent_engine` 部署目标上已标记弃用 |
| `--otel_to_cloud` | 同时导出 Cloud Trace **和** Cloud Logging | 当前推荐,ADK ≥ 1.17 |

```bash
adk web --otel_to_cloud --port 8001 .
```

需要提前 `export GOOGLE_CLOUD_PROJECT=...` 这类环境变量到 **shell 层面**,不能只写在 agent 包内部的 `.env` 里——tracing 的启用检查发生在 web 服务器启动阶段,比每个 agent 自己 `load_dotenv()` 还早。

### 需要装的包

```bash
pip install \
  "opentelemetry-instrumentation-google-genai>=0.4b0" \
  "opentelemetry-instrumentation-vertexai>=2.0b0" \
  opentelemetry-exporter-gcp-logging \
  opentelemetry-exporter-otlp-proto-http \
  opentelemetry-exporter-gcp-trace
```

## 5.3 核心代码 / 排障

### 关键认知:批处理导出器默认只在进程优雅退出时强制刷新

这是本章最容易被忽略的一点——`BatchSpanProcessor`/`BatchLogRecordProcessor` 内部有缓冲队列,如果你的服务是那种"一直跑着不重启"的常驻进程,发一条消息之后立刻去 Cloud Trace 控制台查,大概率**查不到**,不是没导出成功,只是还没刷新:

```bash
# 优雅终止(不是kill -9),触发force_flush
pkill -TERM -f "adk web"
```

### 用 REST API 直接验证(比等控制台刷新更快确认)

```bash
TOKEN=$(gcloud auth print-access-token)
curl -H "Authorization: Bearer $TOKEN" \
  "https://cloudtrace.googleapis.com/v1/projects/PROJECT_ID/traces/TRACE_ID_HEX32"
```

trace_id 需要转成 32 位十六进制(前导零不能省):

```python
print(format(trace_id_int, '032x'))
```

## 5.4 真实运行效果(Cloud Trace 里的调用链)

```text
invocation
└─ invoke_agent root_agent
   └─ call_llm → generate_content → execute_tool transfer_to_agent (分派给order_agent)
└─ invoke_agent order_agent
   └─ call_llm → generate_content → execute_tool get_order_status (真实调用订单API)
   └─ call_llm → generate_content (生成最终回复文本)
```

每个 span 都带真实的开始/结束时间、工具参数、工具返回值、token 用量。Cloud Logging 里的每条日志还会自动带 `trace` 字段,和对应的 Trace 关联,在控制台点开一条 trace 能直接看到"Logs & Events"标签页里的相关日志。

## 5.5 真实踩过的坑

这一章踩坑最密集,按顺序列出(每一个都是真实的包生态断层,不是配置写错了):

| 报错 | 根因 | 解决 |
|---|---|---|
| `ModuleNotFoundError: opentelemetry.exporter.otlp.proto.http` | ADK 源码实际 import 的是 OTLP **HTTP** 变体,不是常见资料里提到的 gRPC 变体 | `pip install opentelemetry-exporter-otlp-proto-http` |
| `ModuleNotFoundError: opentelemetry.exporter.cloud_trace` | `--trace_to_cloud` 走的是另一个独立包 | `pip install opentelemetry-exporter-gcp-trace` |
| `TypeError: Can't instantiate abstract class CloudLoggingExporter ... force_flush` | `opentelemetry-exporter-gcp-logging` 还是 alpha 版本(1.12.0a0),没跟上 `opentelemetry-sdk` 把 `force_flush` 收紧成强制抽象方法这次变化 | 见下方猴子补丁 |

### 猴子补丁修 alpha 包的 ABC 冲突

```python
# sitecustomize.py,放在venv的site-packages目录下,Python解释器启动时自动加载
from opentelemetry.exporter.cloud_logging import CloudLoggingExporter

if "force_flush" in CloudLoggingExporter.__abstractmethods__:
    def _force_flush(self, timeout_millis: int = 30000) -> bool:
        return True
    CloudLoggingExporter.force_flush = _force_flush
    # 只赋值方法不够 —— ABCMeta在类创建时就把__abstractmethods__冻结了,
    # 运行时打补丁必须显式把这个方法名从集合里摘掉
    CloudLoggingExporter.__abstractmethods__ = frozenset(
        m for m in CloudLoggingExporter.__abstractmethods__ if m != "force_flush"
    )
```

这个补丁本身也是一次真实的技术教训:**给类属性赋值一个具体实现,不代表 Python 的 ABC 机制会自动认为这个抽象方法"被满足了"**——`__abstractmethods__` 是类创建时计算并冻结的 frozenset,运行时 monkey-patch 必须连这个集合一起手动更新。

## 5.6 Cloud Trace / Logging 还能做什么

- **Cloud Monitoring 告警**:基于 Trace/Logging 里的指标(比如某个 span 耗时的 P99、某类错误日志的出现频率)设置告警策略,而不是靠人肉盯着控制台。
- **Log-based Metrics**:把结构化日志里的字段(比如 `gen_ai.usage.output_tokens`)转换成可以在 Monitoring 里画图、设告警阈值的指标。
- **跨服务的分布式追踪**:如果 agent 调用的工具本身是另一个独立部署的服务(比如 MCP 服务器),OpenTelemetry 的 `traceparent` 传播能把两边的 trace 串成一条完整链路——不过要注意,ADK 社区已有多个 issue 反馈这个跨服务传播目前并不总是可靠,可能需要手动通过 `header_provider` 注入。

## 5.7 优化方向

- **给追踪数据打上业务标签**,比如客户 ID、对话渠道,方便按业务维度筛选和统计,而不只是按 agent/session 维度。
- **采样策略**:本章为了教学演示是全量导出,真实高流量场景需要考虑采样率,否则 Trace/Logging 的存储和查询成本会显著增加。
- **把"content.capture"的开关想清楚**:`OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT` 控制是否把完整的 prompt/response 文本导出到 Trace/Logging 里——这涉及到用户数据是否落地到可观测性系统这个隐私合规问题,不是默认全开就好,需要和客户的合规要求对齐。
