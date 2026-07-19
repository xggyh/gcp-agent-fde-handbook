# 第 1 章 · 环境搭建

## 1.1 这一步要解决什么问题

在写第一行 agent 代码之前,需要有一个干净、隔离的 GCP 环境:独立的项目(不和其他工作混在一起,方便单独看花费、方便用完整个删掉)、开通后面几章都要用到的 API、本地能跑 ADK 的 Python 环境、以及能让 ADK 认证到这个项目的凭证。

## 1.2 在 GCP 上具体怎么做

### 新建项目并挂账单

```bash
gcloud projects create adk-fde-lab --name="ADK FDE Lab"
gcloud billing projects link adk-fde-lab --billing-account=YOUR_BILLING_ACCOUNT_ID
```

`gcloud billing accounts list` 能看到你有哪些账单账户可以挂。

### 开通所需 API

这份清单是"事后诸葛亮"版——实际搭建过程中是踩了坑才逐步补全的(见下面的"踩坑记录")：

```bash
gcloud services enable \
  aiplatform.googleapis.com \
  modelarmor.googleapis.com \
  dlp.googleapis.com \
  storage.googleapis.com \
  logging.googleapis.com \
  monitoring.googleapis.com \
  cloudtrace.googleapis.com \
  telemetry.googleapis.com \
  cloudresourcemanager.googleapis.com \
  vectorsearch.googleapis.com \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  --project=adk-fde-lab
```

### Application Default Credentials(ADC)

这是给 Python SDK 用的凭证,跟 `gcloud auth login`(CLI 自己的登录)是分开的两件事,很多人第一次搭建时会漏掉这一步:

```bash
gcloud auth application-default login
```

### 本地 Python 环境

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install google-adk
```

ADK 要求 Python ≥ 3.10。

## 1.3 核心代码 / 配置

Agent 的 `.env` 文件(放在 agent 包目录里,不是项目根目录):

```bash
GOOGLE_CLOUD_PROJECT=adk-fde-lab
GOOGLE_CLOUD_LOCATION=us-central1
GOOGLE_GENAI_USE_VERTEXAI=True
```

!!! note "2026 年的环境变量过渡期"
    部分新文档已经开始用 `GOOGLE_GENAI_USE_ENTERPRISE=True` 替代 `GOOGLE_GENAI_USE_VERTEXAI=True`(对应 Vertex AI → Gemini Enterprise Agent Platform 的改名)。如果你按新文档配了新变量名但发现没生效,大概率是当前装的 ADK 版本还只认旧变量名——两个都写上最保险。

## 1.4 真实踩过的坑

| 现象 | 根因 | 解决 |
|---|---|---|
| `gemini-flash-latest` 模型 404 | 这个别名在 Vertex AI 的发布模型目录里(区别于直接用 API Key 那条路径)解析不出来 | 实测确认 `gemini-2.5-flash` 在项目所在区域可用,直接钉死用这个 |
| 新建的计算服务账号权限不够 | Google 在某个时间点后不再默认给新项目的计算服务账号自动授予 Editor 角色 | 手动 `gcloud projects add-iam-policy-binding` 补上 |
| IAM 权限刚加完还是报 403 | IAM 绑定的传播不是瞬时的,通常要等几十秒到几分钟 | 等一下重试,不要怀疑权限配错了 |

## 1.5 这些 GCP 组件还能做什么

- **IAM(Identity and Access Management)**:除了给自己账号加权限,更常见的生产做法是给"跑 agent 的那个身份"(比如一个专用 Service Account,而不是个人 Google 账号)按最小权限原则单独授权,人和"代码的身份"分开管理。
- **Cloud Billing**:一个账单账户可以关联多个项目,还能设置预算告警(某项目/某标签花费超过阈值就发通知),生产环境强烈建议配上,防止一个死循环的调用把账单刷爆。
- **API 服务管理(`gcloud services`)**:每个 API 单独开关,不用的服务默认关闭,这是 GCP 的最小权限设计思路的一部分——被开通的 API 也会出现在项目的"攻击面"里,不需要的不要开。

## 1.6 优化方向

- **用 Terraform 管理这些资源**,而不是手敲一长串 `gcloud` 命令——本章这些步骤(建项目、开 API、配 IAM)全部可以写成 Terraform 配置,团队协作、多环境(dev/staging/prod)复制、变更审计都会顺畅很多。真实 FDE 项目里,手敲命令通常只在最初的探索阶段,一旦方案确定就应该固化成 IaC(Infrastructure as Code)。
- **不要用个人账号的凭证跑生产**:本手册为了教学方便,全程用的是个人 Google 账号的 ADC。生产环境应该用专门的 Service Account,并且遵循最小权限——比如运行时身份只给 `aiplatform.user`,不要给 `owner`。
- **组织级别的项目护栏**:如果是在一个公司/团队内,新建项目通常还需要遵守组织策略(Org Policy)的约束,比如强制要求特定标签、禁止某些高风险 API、强制 VPC Service Controls 等,这些是个人练习项目不会遇到、但真实企业客户现场几乎必然会遇到的额外维度。
