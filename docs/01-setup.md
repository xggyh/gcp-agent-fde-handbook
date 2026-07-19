# 第 1 章 · 环境搭建

## 1.1 这一步要解决什么问题

在写第一行 agent 代码之前,需要有一个干净、隔离的 GCP 环境:独立的项目、开通后面几章都要用到的 API、本地能跑 ADK 的 Python 环境、以及能让 ADK 认证到这个项目的凭证。这四件事听起来都是"环境准备"这种一次性的体力活,但每一件背后都有需要想清楚的决策——而"要不要新建一个独立项目"是其中最容易被跳过、却影响最深远的一个。很多人图省事直接复用手头已有的某个 GCP 项目,短期内看不出问题,但随着章节推进(尤其是第 4 章开始处理隐私数据和护栏、第 6 章开始往云上部署常驻服务),混用项目带来的麻烦会越滚越大,到时候再拆分成本已经晚了。

### 为什么一定要"独立项目",而不是复用现有项目

这不是洁癖,而是三个具体、现实的理由,也是真实 FDE 工作里第一天就要做的判断:

**成本隔离**。GCP 的账单是按项目(以及项目下的资源标签)聚合统计的。如果这次实验性质的 agent 搭建混在一个已经在跑其他业务的项目里,月底账单上"这次 PoC 到底花了多少钱"根本分不清楚——尤其是本手册后面会反复创建、销毁、重新部署 RAG Corpus、Agent Runtime 实例、Cloud Run 服务这类按用量甚至按小时计费的资源,如果和其他工作负载的账单混在一起,连"哪一次调试花了多少冤枉钱"都追溯不出来。对真实 FDE 场景更是如此:你大概率是被派驻到客户已有的 GCP 组织下工作,如果图方便直接在客户的生产项目里做实验,一旦某次测试脚本失控式地反复调用了计费 API,买单的是客户,而且这笔钱很难跟客户原有的正常业务开销剥离开来解释清楚。

**权限隔离**。IAM 策略是挂在项目(或者更高层级的文件夹/组织)上的,新建一个独立项目意味着你在这个项目里拥有的角色,跟你在同一个组织下其他项目里的角色是完全分开的两件事。这带来两个直接的好处:一是可以把自己(以及后面跑 agent 用的服务账号)限定在一个"沙盒"范围内做实验,不会因为一次误操作波及客户现有系统的权限边界;二是方便后续做安全审计——所有跟这个 agent 项目相关的 IAM 绑定变更、Cloud Audit Logs 都聚集在一个项目里,而不是散落在客户组织下几十个项目中的某一个,事后追溯"谁在什么时候给什么身份加了什么权限"会容易得多。这一点在第 6 章会体现得很明显:那一章需要给 Agent Runtime 的运行时身份单独授权,如果项目里混杂着其他系统的 IAM 绑定,排查起来会困难得多。

**清理方便**。PoC 结束、或者这一阶段的探索验证完成之后,一条 `gcloud projects delete adk-fde-lab` 就能把这个项目下所有资源——不管是 Agent Runtime 实例、Cloud Run 服务、RAG Corpus、还是中间建的 Cloud Storage bucket——彻底、一次性地清掉,不需要按资源类型一个个排查有没有漏删的。这个"整个项目一起删"的心智模型,比"记住我到底建过哪些资源、再一个个手动删除"可靠得多,也是避免"PoC 做完忘记关某个计费资源、几个月后收到一笔莫名其妙账单"这类事故最简单直接的办法。

!!! tip "FDE 现场的常见做法"
    真实项目里,给每一次客户 PoC/Demo 开一个独立项目几乎是标准动作——项目命名通常会带上客户名/项目代号和日期,方便后续在几十个项目里快速定位;项目生命周期结束后走内部流程整体归档或删除,不会长期累积"僵尸项目"占着账单和权限。这也是为什么第 1.6 节会把"用 Terraform 管理项目"作为第一条优化方向:一旦这种"建项目 → 用一阵子 → 整体清理"的模式要在多个客户之间重复,手敲命令就不如一份可复用的 IaC 模板划算。

## 1.2 在 GCP 上具体怎么做

### 新建项目并挂账单

```bash
# 创建项目: adk-fde-lab 是 Project ID(全局唯一、创建后不可改),
# --name 是显示名称(可以随时改,只影响控制台展示)
gcloud projects create adk-fde-lab --name="ADK FDE Lab"

# 新项目默认没有关联任何账单账户,这个状态下大部分收费 API 无法启用/调用
# YOUR_BILLING_ACCOUNT_ID 从下面这条命令的输出里找
gcloud billing projects link adk-fde-lab --billing-account=YOUR_BILLING_ACCOUNT_ID
```

`gcloud billing accounts list` 能看到你有哪些账单账户可以挂——个人账号通常只有一个(挂着信用卡的那个),企业组织下可能有多个按部门/项目划分的账单账户,选错了后面财务对账会很麻烦。

几个容易忽略但值得记住的细节:

- Project ID(`adk-fde-lab`)和显示名称(`ADK FDE Lab`)是两个独立的字段,前者是几乎所有 `gcloud`/SDK 调用里真正用到的标识符,后者只是给人看的。真实项目里建议 Project ID 带上有辨识度的前缀(比如客户代号 + 日期),一是避免和别人抢注过的常见词冲突导致创建失败,二是方便日后在一堆项目里靠名字就能认出这是哪次客户交付。
- 如果电脑上同时登录了个人账号和公司/客户账号,建议先用 `gcloud config list account` 或 `gcloud auth list` 确认一下当前 CLI 用的是哪个身份在操作——"建错账号下的项目"是一个挺常见的乌龙,而且不易被立刻发现。

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

`gcloud services enable` 背后调用的是 Service Usage API——每个 GCP 服务在项目层面都有独立的开关,默认全部关闭,这是"最小权限/最小攻击面"设计思路在 API 层面的体现(1.5 节会展开讲这一点)。一次性把 13 个 API 全部列出来开通,而不是需要一个开一个,是为了避免后面某一步突然报 "API not enabled" 又要跑回来补一条命令——但这也意味着你需要提前知道后面章节到底会用到哪些 API,下面把每一个都讲清楚。

**每个 API 具体是做什么的、对应后面哪一章**:

| API(service 名称) | 作用 | 对应章节 / 用途 |
|---|---|---|
| `aiplatform.googleapis.com` | Vertex AI(现称 **Gemini Enterprise Agent Platform**)的核心 API,模型推理调用、RAG Engine 的 corpus 管理与检索、Agent Runtime 部署、Gen AI Evaluation 服务全部挂在这个 API 命名空间下 | 第 2 章(模型调用)、第 3 章(RAG Engine)、第 6 章(Agent Runtime 部署)、第 7 章(评估)——贯穿全书,是最重要的一个 |
| `modelarmor.googleapis.com` | Model Armor 护栏服务自身的 API,提供 `sanitizeUserPrompt`/`sanitizeModelResponse` 等接口,对 agent 的输入输出做统一安全检查 | 第 4 章 |
| `dlp.googleapis.com` | Sensitive Data Protection(原 Cloud DLP)的 API,Model Armor 的 PII 检测、自定义竞品词典检测能力实际上建立在 DLP 的 Inspect Template 之上 | 第 4 章(Model Armor 的底层依赖) |
| `storage.googleapis.com` | Cloud Storage,本项目里主要用作 Agent Runtime 部署时的 staging bucket,暂存打包后的 agent 代码 | 第 6 章 |
| `logging.googleapis.com` | Cloud Logging,OpenTelemetry 日志导出的落地位置,也是部署后排查 agent 服务端报错(按 `resource.type` 过滤)的地方 | 第 5 章(可观测性)、第 6 章(部署后排障) |
| `monitoring.googleapis.com` | Cloud Monitoring,和 Cloud Trace 配合构成完整的可观测性栈,可以基于延迟/错误率等指标配置告警 | 第 5 章;也是 1.6 节提到的预算/资源告警能力的基础设施之一 |
| `cloudtrace.googleapis.com` | Cloud Trace,OpenTelemetry span 数据的落地位置,记录一次对话内每一步(每次模型调用、每次工具调用)的耗时和参数 | 第 5 章 |
| `telemetry.googleapis.com` | 相对较新的遥测摄取 API,是 `--otel_to_cloud` 背后统一遥测数据摄取通道的一部分,配合 Cloud Trace/Logging 把 OTel 数据正确导入 | 第 5 章 |
| `cloudresourcemanager.googleapis.com` | 管理项目本身以及 IAM 策略绑定的 API,`gcloud projects add-iam-policy-binding` 这类命令底层调用的正是它 | 第 1 章(本章的 IAM 授权操作)、第 6 章(修复 Agent Runtime 运行时身份权限) |
| `vectorsearch.googleapis.com` | Vector Search 服务的 API,对应第 3 章方案对比表格里"手动 Vector Search"那条更复杂、需要自己管理 ANN 索引的路径 | 第 3 章(备选方案;本项目最终选用 RAG Engine,未直接依赖它,提前开通是为了不影响后续对比实验) |
| `run.googleapis.com` | Cloud Run,托管客户订单查询 Mock API 的地方 | 第 6 章 |
| `artifactregistry.googleapis.com` | Artifact Registry,`gcloud run deploy --source` 背后 Cloud Build 构建出来的容器镜像存放的仓库 | 第 6 章 |
| `cloudbuild.googleapis.com` | Cloud Build,`--source` 这种部署方式背后真正执行"打包源码 → 构建容器镜像"的服务;也是第 6 章 IAM 踩坑的主角 | 第 6 章 |

可以把这 13 个 API 按功能分成五组来理解,记这五组比记 13 个孤立的名字容易得多:

1. **推理与检索核心**:`aiplatform`(几乎全程都用到)、`vectorsearch`(第 3 章的备选方案)。
2. **安全护栏**:`modelarmor` + `dlp`,第 4 章里两者配合使用,架构图里画得很清楚——Model Armor 是外层调用入口,DLP 是底层实际做检测的引擎。
3. **可观测性三件套加一个新摄取通道**:`logging`、`monitoring`、`cloudtrace`、`telemetry`,共同构成第 5 章的全链路追踪能力。
4. **部署与构建**:`run`、`artifactregistry`、`cloudbuild`、`storage`,第 6 章把 agent 本体和它依赖的 mock API 部署上云时,这四个 API 会依次被调用到(源码打包到 Storage/直接读取 → Cloud Build 构建镜像 → 镜像推到 Artifact Registry → Cloud Run 拉取镜像启动服务)。
5. **治理**:`cloudresourcemanager`,管项目和 IAM 绑定本身,本章和第 6 章都会直接用到它做权限操作。

### Application Default Credentials(ADC)与 gcloud auth login 的区别

这是本章最容易被搞混、也是第一次搭建 GCP 环境时最常见的踩坑点,值得单独讲清楚,而不是一笔带过。

```bash
gcloud auth application-default login
```

表面上看,这条命令和你可能已经跑过的 `gcloud auth login` 几乎一模一样——都是打开浏览器、登录一次 Google 账号、点一下授权。正因为操作过程看起来在"重复",很多人第一次配置环境时会想"我明明已经 `gcloud auth login` 过了,为什么 Python 脚本还是报认证错误",然后卡在这里排查半天。实际上这是两套完全独立的凭证体系,服务的对象不一样:

| | `gcloud auth login` | `gcloud auth application-default login` |
|---|---|---|
| 认证的对象 | **gcloud CLI 工具本身** | **基于 Google Cloud 客户端库的代码**(Python 的 `google-cloud-aiplatform`、`vertexai` SDK,以及 ADK) |
| 凭证存放位置 | gcloud 自己内部的凭证 store | 固定路径的 ADC 文件(`~/.config/gcloud/application_default_credentials.json`) |
| 谁在实际使用它 | 你敲的每一条 `gcloud ...` 命令 | `vertexai.init()`、ADK 内部初始化、任何调用 Google Cloud 客户端库且没有显式传凭证参数的代码 |
| 是否可以互相替代 | 不可以——两者服务不同的调用路径,各自独立 | 同左 |

换句话说,`gcloud auth login` 解决的是"我在终端里敲 `gcloud projects list`/`gcloud run deploy` 这类命令时,gcloud 用谁的身份去调 API";而 `gcloud auth application-default login` 解决的是"我写的 Python 代码(比如 `vertexai.init(project=..., location=...)`,或者 ADK 内部发起的模型调用)在没有被显式告知用哪个凭证时,应该去哪里找一份默认凭证"。ADK 和 `vertexai` SDK 走的正是第二条路径——如果只做了 `gcloud auth login`,`gcloud` 命令行操作一切正常,但第一次跑 agent 代码时大概率会报类似 "Could not automatically determine credentials" 的错误,这就是漏做了 ADC 这一步的典型症状。

!!! warning "两者可以对应不同的身份,这是有意为之的设计"
    ADC 和 gcloud CLI 的登录身份并不要求一致——比如生产环境里,你可能用自己的管理员账号做 `gcloud auth login` 来手动运维,但本地跑代码测试时特意用 `gcloud auth application-default login --impersonate-service-account=...` 让 ADC 对应一个权限缩得很小的服务账号,避免本地调试代码时"手滑"用到了管理员权限。1.6 节的优化方向里会再展开这一点——本手册为了教学方便全程用个人账号的 ADC,生产环境不建议这样做。

### 本地 Python 环境

```bash
python3 -m venv .venv                 # 在项目根目录下创建虚拟环境,目录名 .venv 是社区惯例
source .venv/bin/activate             # 激活;Windows 下对应命令是 .venv\Scripts\activate
pip install --upgrade pip             # 老版本 pip 解析 ADK 的依赖树时偶尔会出兼容性问题,先升级更保险
pip install google-adk
```

ADK 要求 Python ≥ 3.10。

几点值得在正式开始写 agent 代码之前养成习惯的小事:

- **一定要用虚拟环境,不要全局 `pip install`**。ADK 和它依赖的一整套 Google Cloud 客户端库更新频繁,第 5 章还会用到好几个仍处于 beta 阶段的 `opentelemetry-*` 包,版本要求比较挑剔——如果全局安装,很容易和你电脑上其他 Python 项目的依赖产生版本冲突,而且很难排查是哪个包版本对不上。虚拟环境把这一切都限制在项目目录内部,删掉 `.venv` 文件夹就是完全重来,不会污染系统环境。
- `.venv/` 目录应该加进 `.gitignore`,不要提交进版本库——它是纯本地产物,换一台机器或者交给同事,靠 `requirements.txt` 重新安装即可,不需要也不应该把整个虚拟环境目录搬来搬去。
- 建议在环境搭好、能跑通之后执行一次 `pip freeze > requirements.txt`,把当前所有依赖的精确版本号锁定下来。这在团队协作或者几个月后需要在新机器上复现同一套环境时会省很多排查版本差异的功夫。
- 如果你电脑上系统自带的 `python3` 版本低于 3.10,不要动系统 Python,用 `pyenv`(或者类似的版本管理工具)装一个满足要求的版本,再拿它去创建虚拟环境——直接升级系统 Python 容易影响到操作系统自身依赖的脚本。
- 如果用 VS Code 之类的编辑器,记得在编辑器里把 Python 解释器手动切换成 `.venv` 里的那一个(而不是系统默认的),不然编辑器的代码补全和实际运行环境会对不上,看起来"明明装了却提示找不到模块"。

### 环境自检清单

进入第 2 章之前,建议按这个顺序确认一遍——下面任何一项没做,大概率会在后面某一章报出一个乍看和当前操作无关的错误:

- [ ] `gcloud config get-value project` 输出的是 `adk-fde-lab`,不是其他项目
- [ ] `gcloud billing projects describe adk-fde-lab` 能看到已关联的账单账户
- [ ] `gcloud services list --enabled --project=adk-fde-lab` 里能看到本章开通的 13 个 API
- [ ] `gcloud auth application-default print-access-token` 能正常打印出一个 token(证明 ADC 配置成功,而不仅仅是 `gcloud auth login` 成功)
- [ ] `python3 --version` ≥ 3.10,且当前 shell 已经 `source .venv/bin/activate` 激活了虚拟环境
- [ ] `pip show google-adk` 能看到已安装的版本号

## 1.3 核心代码 / 配置

Agent 的 `.env` 文件(放在 agent 包目录里,不是项目根目录——这个目录约定在第 2 章讲 ADK 的项目结构时会具体展开):

```bash
GOOGLE_CLOUD_PROJECT=adk-fde-lab
GOOGLE_CLOUD_LOCATION=us-central1
GOOGLE_GENAI_USE_VERTEXAI=True
```

三个变量各自的作用:

- `GOOGLE_CLOUD_PROJECT`:告诉 ADK/`vertexai` SDK 这次的模型调用、RAG 检索等操作应该记到哪个项目的账单和配额上,对应的正是本章一开始新建的 `adk-fde-lab`。
- `GOOGLE_CLOUD_LOCATION`:Vertex AI 相关资源(模型、RAG Corpus、Agent Runtime 部署)是按区域(region)划分的,`us-central1` 是当前 Vertex AI 功能覆盖最全、最先拿到新特性的区域之一,本手册全程统一用它,避免出现"某个功能在这个区域还没开放"的额外变量。
- `GOOGLE_GENAI_USE_VERTEXAI`:决定底层 `google-genai` SDK 走的是 Vertex AI 这条认证/计费路径,还是直接用 Gemini API Key 那条路径。这个开关的选择会直接影响模型别名的解析行为——1.4 节"踩坑记录"里 `gemini-flash-latest` 404 的问题,根源就是这两条路径背后的模型目录并不完全一致。

!!! note "2026 年的环境变量过渡期"
    部分新文档已经开始用 `GOOGLE_GENAI_USE_ENTERPRISE=True` 替代 `GOOGLE_GENAI_USE_VERTEXAI=True`(对应 Vertex AI → Gemini Enterprise Agent Platform 的改名)。如果你按新文档配了新变量名但发现没生效,大概率是当前装的 ADK 版本还只认旧变量名——两个都写上最保险。

## 1.4 真实踩过的坑

| 现象 | 根因 | 解决 |
|---|---|---|
| `gemini-flash-latest` 模型 404 | 这个别名在 Vertex AI 的发布模型目录里(区别于直接用 API Key 那条路径)解析不出来 | 实测确认 `gemini-2.5-flash` 在项目所在区域可用,直接钉死用这个 |
| 新建的计算服务账号权限不够 | Google 在某个时间点后不再默认给新项目的计算服务账号自动授予 Editor 角色 | 手动 `gcloud projects add-iam-policy-binding` 补上 |
| IAM 权限刚加完还是报 403 | IAM 绑定的传播不是瞬时的,通常要等几十秒到几分钟 | 等一下重试,不要怀疑权限配错了 |

### 深入一层:为什么"新项目默认计算服务账号不再自动获得 Editor 角色"

第二行这个坑看起来只是一句话带过,但它背后的行为变化值得展开讲清楚,因为它不是这一章会真正暴露出来的问题——它会在第 6 章部署 Cloud Run 服务时才真正让你吃一次亏。

**过去的默认行为是什么**。在这个变化之前,每个新建的 GCP 项目都会自动创建两个默认服务账号,其中之一是 **Compute Engine 默认服务账号**(格式形如 `PROJECT_NUMBER-compute@developer.gserviceaccount.com`),这个账号会被自动授予项目级别的 `roles/editor`(Editor)角色。这么设计的初衷是"开箱即用":很多 GCP 服务——Compute Engine 虚拟机、Cloud Functions、Cloud Run 等等——默认情况下会用这个服务账号作为运行时身份或者构建时身份,如果它天然就有 Editor 权限,大部分场景下不需要用户再手动配置任何 IAM,项目一建好、代码一部署就能跑通。

**为什么后来改了**。Editor 角色的权限范围极大——项目里几乎所有资源的读写都覆盖到了。这意味着一旦这个默认服务账号的凭证被滥用(比如某个自动化脚本的 bug 意外把这个身份的权限用去做了不该做的事,或者构建流程配置疏忏导致权限被间接放大使用),影响面几乎等同于整个项目失守。Google 在推行"最小权限默认化"的安全加固方向上做了调整:新建的项目里,这个默认计算服务账号不再自动获得 Editor 角色,而是只保留一个权限小得多的基础角色。这是一个**面向所有新建项目**的默认行为变更——已经存在的老项目如果之前已经有这条绑定,通常不会被追溯撤销,但任何从这次变更之后新建的项目,从创建的那一刻起就没有这个"隐藏的万能权限"了。这个方向跟本章 1.5 节要讲的 IAM 最小权限设计思路完全一致,只是这次被 Google 直接做成了平台默认值,而不是等用户自己去手动收紧。

**为什么这会在第 6 章冒出来,而不是这一章**。问题的关键在于:这个权限缺口不会在你"创建项目"或者"开通 API"的时候暴露出来——项目建得好好的,API 也开通成功了,一切看起来正常。它只会在你**第一次真正触发这个服务账号去执行某个需要较高权限的操作**时才报错。第 6 章里 `gcloud run deploy --source=.` 这条命令背后,Cloud Build 会读取你上传的源码、构建容器镜像、再把镜像推送到 Artifact Registry,这一整条构建流水线默认使用的正是这个计算服务账号身份。没有 Editor 角色兜底之后,Cloud Build 可能因为读不到暂存源码的存储位置、或者没有权限把构建好的镜像推到 Artifact Registry,而报出类似 `PERMISSION_DENIED: Build failed because the default service account is missing required IAM permissions` 这样的错误——这也是为什么这份手册在最初搭建环境这一章,就先把这个背景解释清楚:等你真正走到第 6 章遇到这个报错时,能立刻反应过来"这是权限模型变化导致的,不是我哪一步操作错了",直接去补 IAM 绑定,而不是怀疑自己部署命令写错了参数。

!!! danger "这也是为什么不该依赖默认服务账号的隐式权限"
    即便这个 Editor 角色还在某些老项目里存在,生产环境也不应该依赖它——一个到处都有 Editor 权限的服务账号本身就是安全隐患。正确做法是明确知道这个身份实际需要哪些权限,按最小权限单独授予(第 6 章会具体演示怎么给 Agent Runtime 的运行时身份补 `roles/aiplatform.user` 这一类精确到场景的角色),而不是依赖一个"以前能用是因为权限给得太宽"的历史遗留行为。

## 1.5 这些 GCP 组件还能做什么

### IAM(Identity and Access Management)

本章只用到了最基础的用法——给项目加一条 IAM 绑定。IAM 实际的能力比这丰富得多,下面是几个在真实项目里会用到的延伸能力:

- **专用 Service Account 而不是个人账号**:生产做法是给"跑 agent 的那个身份"单独建一个 Service Account,按最小权限原则单独授权,把"人"和"代码的身份"彻底分开管理——个人账号的权限变化(离职、转岗)不应该影响到线上服务的运行。
- **条件绑定(condition-based binding)**:IAM 的角色绑定可以附加一个 condition 表达式,比如限定"这条绑定只在某个时间窗口内生效",或者"只对满足某个命名前缀的资源生效"。适合给外部顾问/临时协作者开一个有截止日期的访问窗口,或者按资源分组做更精细化的授权,而不需要事后再手动去撤销。
- **服务账号模拟(Service Account Impersonation)**:允许一个已授权的用户或服务账号临时"变成"另一个服务账号去执行操作,而不需要下载、分发那个服务账号的 JSON 密钥文件。密钥文件一旦下载到本地就有泄露风险,用模拟机制可以完全避免这个风险敞口,是本地开发/CI 场景下比"下载 key 文件"更推荐的方式。
- **自定义角色(Custom Role)**:当预定义角色(比如 `roles/aiplatform.user`)的权限粒度不够精确时——例如你只想允许某个身份查询 RAG Corpus,但不想让它有权限创建或删除 Corpus——可以基于具体需要的 permission 组合出一个自定义角色,实现比预定义角色更细的最小权限控制。
- **IAM Recommender 与 Policy Analyzer**:官方文档提到,GCP 会基于身份过去一段时间实际用到的权限,自动给出"可以收紧"的角色调整建议,适合定期审计生产环境是不是存在权限过度授予(over-privileged)的身份;Policy Analyzer/Policy Troubleshooter 则可以用来排查"为什么某个身份对某个资源没有权限",尤其是权限继承自文件夹/组织层级、单看项目内的绑定看不出全貌的时候,比人工一条条翻 IAM 绑定快得多。

### Cloud Billing

- **预算告警(Budget Alert)**:在 Billing 账户或者具体项目层面设置一个预算金额,再配置多个提醒阈值(比如预算的 50%、90%、100%),达到阈值后通过邮件通知负责人,也可以进一步接到 Pub/Sub 主题、触发一个 Cloud Function 自动做出反应(比如自动禁用某个异常调用的服务账号)。生产环境强烈建议配上,防止一个失控的重试循环把账单刷爆才被发现。
- **按标签(Label)拆分账单**:给资源打上标签(比如 `env:dev`、`team:fde-lab`),账单报表就可以按标签维度聚合花费,适合多团队/多环境共用一个账单账户时做成本分摊,不需要为每个环境单独开一个账单账户。
- **导出账单明细到 BigQuery**:官方文档提到可以配置账单数据的详细导出,导出之后就能用 SQL 去分析"哪个 API、哪个 SKU、哪一天"贡献了主要花费,比控制台自带的报表灵活得多,适合做更细粒度的成本归因分析,尤其是当一个项目里同时跑着好几个不同用途的工作负载时。
- **一个账单账户挂多个项目、分别设置独立预算**:企业内常见的做法是一个 Billing 账户下挂 dev/staging/prod 好几个项目,但分别给每个项目设置独立的预算阈值——避免某个环境(比如反复重跑的 dev 环境)的异常消费被总账单的规模"稀释"掉,迟迟没人发现。
- **承诺使用折扣(Committed Use Discount)**:如果某类计算资源已经验证过负载长期稳定,可以购买承诺使用折扣换取更低的单价——这适用于已经过了 PoC 阶段、进入稳定生产运行的场景,本手册这种探索性质的搭建阶段不适用。

### API 服务管理(Service Usage / `gcloud services`)

- **配额(Quota)管理**:每个 API 通常都有默认的用量配额(比如每分钟允许的请求数),对应 API 自己的 Quotas 页面可以查看当前用量和上限,也可以在合理范围内申请提升配额——上线前压测,评估会不会撞到默认限速,是容易被忽略但很重要的一步。
- **定期审计已启用的 API 列表**:定期检查项目里到底启用了哪些 API,关掉不再使用的服务,是缩小项目攻击面的直接手段——被启用的 API 本身就是项目对外暴露的能力面之一,不需要的就不应该开着。
- **按项目单独调整配额(Consumer Quota Override)**:官方文档提到可以针对具体项目单独调整某个 API 的配额上限(在 Google 允许的范围内),比如给一个高频调用 RAG 检索的生产项目单独提升 `aiplatform` API 的请求速率上限,而不用整个组织统一调整。
- **组织级别的 API 管控(Org Policy)**:组织层面可以用 Org Policy 里跟 Service Usage 相关的约束,限定整个组织/文件夹下允许哪些 API 被启用,防止某个团队不小心开通了不符合公司合规要求的服务——这一点会在 1.6 节"组织级别的项目护栏"里再展开。
- **审计每一次 API 开关操作**:每次 `gcloud services enable`/`disable` 都会被 Cloud Audit Logs 记录下来,方便事后追溯"这个 API 是什么时候、被谁开通的",尤其是排查"为什么这个项目账单上突然多出一项之前没有的费用"时很有用。

## 1.6 优化方向

### 用 Terraform 管理这些资源

本章这些步骤——建项目、开 API、配 IAM——全部可以写成 Terraform 配置,而不是手敲一长串 `gcloud` 命令。真实 FDE 项目里,手敲命令通常只在最初的探索阶段有意义(快速试错、随时调整),一旦方案确定、需要在多个客户环境之间复用,或者需要团队协作评审变更,就应该固化成 IaC(Infrastructure as Code)。判断"什么时候该切换到 Terraform"的一个简单标准是:如果这套环境搭建步骤需要被**执行第二次**(哪怕只是给下一个客户 PoC 复用一遍),手动操作积累的心智负担就已经值得用一次性的 Terraform 化投入去换掉。

下面是一个示意片段,说明本章"建项目 + 开 API"这两步用 Terraform 大概会长什么样子——具体的资源参数名、必填字段请以 [Terraform Google Provider 官方文档](https://registry.terraform.io/providers/hashicorp/google/latest/docs) 为准,这里只是给出思路,不保证逐字可直接 `terraform apply`:

```hcl
# main.tf —— 示意代码,展示思路而非保证可直接运行

resource "google_project" "adk_fde_lab" {
  name            = "ADK FDE Lab"
  project_id      = "adk-fde-lab"
  org_id          = var.org_id            # 挂在组织下;如果是挂在文件夹下,对应换成 folder_id
  billing_account = var.billing_account_id # 对应 gcloud billing projects link 那一步
}

# 用 for_each 把本章那一长串 gcloud services enable 变成声明式的资源列表,
# 增删一个 API 只需要改这个 set,而不是重新拼一条命令
resource "google_project_service" "enabled_apis" {
  for_each = toset([
    "aiplatform.googleapis.com",
    "modelarmor.googleapis.com",
    "dlp.googleapis.com",
    "storage.googleapis.com",
    "logging.googleapis.com",
    "monitoring.googleapis.com",
    "cloudtrace.googleapis.com",
    "telemetry.googleapis.com",
    "cloudresourcemanager.googleapis.com",
    "vectorsearch.googleapis.com",
    "run.googleapis.com",
    "artifactregistry.googleapis.com",
    "cloudbuild.googleapis.com",
  ])

  project = google_project.adk_fde_lab.project_id
  service = each.value

  # 销毁这个 Terraform 资源(比如整个环境要清理)时不要连带关闭 API,
  # 避免误伤这个项目里其他还依赖同一个 API 的资源
  disable_on_destroy = false
}

# 对应 1.4 节踩坑记录里补的那条 IAM 绑定,提前在 Terraform 里声明好,
# 就不需要等第 6 章真的踩到 403 才手动补
resource "google_project_iam_member" "compute_sa_aiplatform_user" {
  project = google_project.adk_fde_lab.project_id
  role    = "roles/aiplatform.user"
  member  = "serviceAccount:${google_project.adk_fde_lab.number}-compute@developer.gserviceaccount.com"
}
```

几个配套的工程实践,一旦引入 Terraform 就应该一起考虑:

- **远程 state**:把 `.tfstate` 存到一个专门的 Cloud Storage bucket 里(而不是本地磁盘),多人协作时靠 state locking 避免并发 `apply` 冲突。
- **多环境用不同的 `.tfvars`,而不是复制一整份配置**:同一份模块,dev/staging/prod 只是传入的变量(项目 ID、区域、预算阈值等)不同,避免出现三份配置文件慢慢"各自演化、互相不一致"的情况。
- **IAM 相关的变更走代码评审**:IAM 绑定是安全敏感变更,团队协作场景下应该像评审业务代码一样评审"谁给谁加了什么权限"的 Terraform diff,而不是任何人都能直接在控制台点几下就改掉。
- **可复用成模块**:如果 FDE 工作经常需要给不同客户重复"建一个隔离的 agent 沙盒项目"这个动作,把 1.2 节这一整套操作封装成一个 Terraform module,下一次交付只需要传入客户代号、区域这些变量,而不是从头再敲一遍命令。

### 不要用个人账号的凭证跑生产

本手册为了教学方便,全程用的是个人 Google 账号的 ADC(即 1.2 节 `gcloud auth application-default login` 生成的那份凭证)。生产环境应该用专门的 Service Account,并且遵循最小权限——比如运行时身份只给 `aiplatform.user`,不要给 `owner`。如果是自动化流水线(CI/CD)在调用 GCP,进一步的优化方向是用 **Workload Identity Federation**,让 CI 系统凭借它自己的身份换取一个短期的 GCP 访问令牌,彻底不需要下载、保管任何服务账号的 JSON 密钥文件——密钥文件是这类场景下最常见的泄露源头之一。

### 组织级别的项目护栏

如果是在一个公司/团队内,新建项目通常还需要遵守组织策略(Org Policy)的约束,这些是个人练习项目不会遇到、但真实企业客户现场几乎必然会碰到的额外维度,提前了解概念,现场被问到不至于懵:

- 比如 `constraints/iam.allowedPolicyMemberDomains` 这类约束可以限定只有指定域名的账号才能被 IAM 绑定授权,防止误把权限授予给公司域名之外的账号。
- 计算资源相关的约束(比如禁止分配外部 IP)可以在组织层面统一强制,不需要每个项目各自记得配置。
- **VPC Service Controls**:在网络边界层面限制数据能不能流出一个安全边界,是很多金融/政务类客户现场的硬性要求。
- 组织策略通常还会强制要求新建资源打上特定标签(方便成本归因和合规追踪),或者禁止启用某些被认定为高风险的 API——这跟 1.5 节提到的"API 管控"是同一个思路在组织层面的延伸。

这些护栏在个人练习阶段感受不到,但一旦项目建在客户已有的 GCP 组织下,这些策略往往是提前设好、不由你决定的约束条件,理解它们的存在和大致作用,是 FDE 现场沟通时避免"为什么我这条命令跑不通"式困惑的重要背景知识。
