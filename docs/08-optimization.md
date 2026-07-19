# 第 8 章 · 优化方向与附录

## 8.1 全局性优化清单

前面每章都有各自维度的优化建议,这里补充几条**跨模块的、生产化必须考虑**的事情:

### 架构层面

- **安全护栏(第4章)目前只在"本地脚本包一层"的方式下生效**,部署到 Agent Runtime 之后(第6章)并没有跟着一起部署。真实生产系统需要把 Model Armor 检查也纳入部署后的调用链路——可以做成 `before_model_callback`/`after_model_callback`(ADK 内置的回调机制),让护栏跟着 agent 一起被部署,而不是外部再包一层。
- **RAG(第3章)、Model Armor(第4章)、可观测性(第5章)三者目前是独立搭建的**,生产环境应该考虑用 Terraform 把整个环境定义成一份可复现的配置,而不是分散的手工步骤。

### 成本层面

- 这个项目里持续产生费用的资源:Agent Runtime 部署实例、Cloud Run 服务、RAG Corpus(Serverless 模式)。用完记得检查账单,或者直接删除整个 GCP 项目。
- 建议给项目设置 [Budget Alert](https://cloud.google.com/billing/docs/how-to/budgets),超过阈值自动通知,避免意外的持续计费。

### 组织与协作层面

- 真实客户现场往往还有:VPC Service Controls(限制数据出入边界)、组织策略(Org Policy)约束、多团队权限隔离——这些是个人练习项目遇不到、但企业客户几乎必然会问的问题,提前了解概念,现场被问到不会懵。

## 8.2 全局踩坑速查表

按"看到这个报错关键词,大概率是这个原因"整理,方便以后遇到类似问题时快速定位:

| 报错关键词 | 大概率原因 | 对应章节 |
|---|---|---|
| `404 ... Publisher model ... was not found` | 模型别名(如 `-latest`)在 Vertex AI 发布模型目录里不认,换成具体模型 ID | 第 1 章 |
| `PERMISSION_DENIED` + 但 IAM 策略看着是对的 | IAM 绑定传播延迟,等一下重试 | 第 1、6 章 |
| Model Armor / gcloud 命令报"权限拒绝"但语焉不详 | 换成直接调 REST API,报错信息通常更准确 | 第 4 章 |
| RAG corpus 创建报 "Spanner mode ... restricted" | 新项目默认建库模式受白名单限制,显式切 Serverless | 第 3 章 |
| `ModuleNotFoundError: opentelemetry.exporter...` | OTel 导出器包版本/变体选错,注意 HTTP vs gRPC、Cloud Trace vs OTLP 是不同的包 | 第 5 章 |
| 追踪数据在控制台一直查不到 | 批处理导出器只在进程优雅退出时强制刷新,常驻服务需要显式 flush 或等待批处理间隔 | 第 5 章 |
| Cloud Run / Cloud Build 部署报默认服务账号权限不足 | 新项目默认不再自动给计算服务账号 Editor 角色 | 第 6 章 |
| 客户端调用挂起、无异常、最终超时 | 不要等客户端,直接去 Cloud Logging 按资源类型过滤查服务端真实报错 | 第 6 章 |
| 评估分数离谱地全部很低/很高 | 大概率是内置指标语义选错了,不是模型/agent真的有问题 | 第 7 章 |

## 8.3 参考资料

- [ADK 官方文档](https://google.github.io/adk-docs/)
- [Gemini Enterprise Agent Platform(原 Vertex AI)](https://cloud.google.com/products/gemini-enterprise-agent-platform)
- [Vertex AI RAG Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/rag-engine/rag-overview)
- [Model Armor](https://cloud.google.com/security/products/model-armor)
- [Gen AI Evaluation Service](https://cloud.google.com/vertex-ai/generative-ai/docs/models/evaluate-judge-model)
- [OpenTelemetry GenAI Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)

## 8.4 这份手册之外,还没覆盖到的

如实列出,不装作面面俱到:

- **多轮对话的长期记忆(Memory Bank)**没有深入展开,只在第6章提到 Agent Runtime 内置支持。
- **MCP(Model Context Protocol)工具集成**没有实际动手做,只在架构决策层面讨论了什么时候该用 MCP、什么时候该用 OpenAPIToolset。
- **高并发场景下的性能测试**完全没有涉及,本手册的所有验证都是单用户、低并发的功能性验证,没有做过压力测试。
- **多语言/国际化**没有专门处理,示例里的中英文混用是随手写的,不是特意设计的多语言方案。

这些都是这份手册基础上可以继续深入的方向。
