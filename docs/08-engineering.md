# 第 8 章 · 工程化:IaC / 测试 / CI

## 8.1 这一步要解决什么问题

前面七章把一个 agent 从零搭到了部署上线,功能上它是完整的。但如果你把它当作一个**要交付给客户、要移交给别的工程师维护**的项目来看,它还缺一层东西——不是某个功能,而是让这套系统"能被可靠地反复重建、能在改动后自动验证、能安全地交到别人手上"的工程化基线。

具体缺三样,它们恰好是一条链:

- **IaC(基础设施即代码)**:整套环境是前几章手敲 `gcloud` + 跑 Python 脚本 + 点控制台混着建出来的。没有任何一份东西完整记录了"这套环境由哪些资源组成"。误删一次、或者要在客户的项目里复制一套,就只能凭记忆重放几十步——第 4 章真踩过的"Model Armor 和 DLP 模板 region 不一致",就是手敲才会犯的错。
- **自动化测试**:一个都没有。每次验证都是手动发一句话、肉眼看回答对不对。一次性、覆盖不全,而且最容易被静默改坏的路由、工具鉴权、RAG grounding,恰恰是肉眼抽查最不容易发现的。
- **CI(持续集成)**:改一行代码到上线,中间是本地手敲 `adk deploy`,没有任何自动门禁。

这一章把这三样真正补上——而且和前面每一章一样,下面的每一段代码都是真实写进项目、真实跑过验证的,不是示意。

!!! note "为什么这三样总被一起提"
    它们是一条链:**IaC** 让环境能被代码定义和复制 → 有了它才能低成本开出**隔离的多套环境** → **CI** 把"改代码→测试→部署"自动串起来 → 其中的**测试**是这条流水线里的质量门禁。四个凑齐,你才从"我一个人能手动让它跑起来"升级到"这套系统能被一个团队可靠地反复交付、还能安全地交接给客户"——而后者才是工业级 FDE 项目的及格线。

## 8.2 在 GCP 上具体怎么做

### IaC:用 Terraform 把手敲的资源声明成代码

Terraform 是行业事实标准的 IaC 工具。它不绑定单一云——靠一层 **provider(提供商插件)** 支持 GCP、AWS、Azure、Kubernetes 等几百种目标,你用同一套语法(HCL)管理不同厂商的东西。这和 GCP 自己的 Deployment Manager、AWS 的 CloudFormation 那种"绑死单一云"的工具是本质区别。

它和 `requirements.txt` / `Dockerfile` 属于同一个思想家族("把该有的东西写成文件,让工具照着搭"),但操作的层不同、也更强:

| | 管什么层 | 声明式还是步骤式 |
|---|---|---|
| requirements.txt | 一台机器**内部**的 Python 库 | 声明式列表 |
| Dockerfile | 打包**一台机器镜像** | 步骤式(RUN/COPY 一步步 build) |
| Terraform | 机器**外部**的云资源(项目、IAM、Cloud Run、数据库……) | **声明式**(描述最终状态,不是执行步骤) |

Terraform 能"一键复原"的关键在于:你写的 `.tf` 文件描述的是**想要的最终状态**,它内部用一个 **state(状态文件)** 记录"我管的资源现在实际什么样",每次 `apply` 把两者一比、只补齐差异。什么都没有就全建、删了一个就只重建那一个、已经一致就什么都不做(幂等)——这是纯 Dockerfile 做不到的,它只会"从零 build",不会"持续让现实趋近于声明"。

项目里新增的 `infra/` 目录就是这套东西。以最能体现价值的三段为例。

**第一段:用 `for_each` 声明所有要开的 API(对应第 1 章手敲的一长串 `gcloud services enable`):**

```hcl
locals {
  services = [
    "aiplatform.googleapis.com",   # 第2/3/6/7章
    "modelarmor.googleapis.com",   # 第4章
    "dlp.googleapis.com",          # 第4章
    "vectorsearch.googleapis.com", # 第3章: RAG Serverless 的底层依赖(踩坑补开的)
    "run.googleapis.com",          # 第6章
    # ...共 13 个
  ]
}

resource "google_project_service" "enabled" {
  for_each           = toset(local.services)
  project            = var.project_id
  service            = each.value
  disable_on_destroy = false   # 拆 TF 时不要顺手把 API 全关了
}
```

`for_each` 让"一个集合里的每一项各建一个资源"只写一次,而不是复制粘贴 13 个几乎一样的资源块。

**第二段:把前几章"部署时才报错、手动补"的两个 IAM 坑固化下来。** 这是 IaC 最直接的价值——让"新项目默认权限不够"从"部署失败才发现"变成"`apply` 时就在标准流程里补好":

```hcl
# 第6章第一个坑: Cloud Build 计算 SA 缺 Editor,gcloud run deploy --source 报权限错
resource "google_project_iam_member" "cloudbuild_compute_sa_editor" {
  project = var.project_id
  role    = "roles/editor"
  member  = "serviceAccount:${var.project_number}-compute@developer.gserviceaccount.com"
}

# 第6章第二个坑: Agent Runtime 运行时 SA 缺 RAG 查询权限,policy_agent 挂起180秒才报403
resource "google_project_iam_member" "agent_runtime_rag_access" {
  project = var.project_id
  role    = "roles/aiplatform.user"
  member  = "serviceAccount:service-${var.project_number}@gcp-sa-aiplatform-re.iam.gserviceaccount.com"
}
```

注意这里用了 `var.project_number`(项目号)而不是 `project_id`——这两个是不同的东西,很多服务账号的名字里用的是项目号,这也是前面踩坑时容易搞混的点,IaC 化之后把它固定成了一个显式变量。

**第三段:把 `--allow-unauthenticated` 这个安全隐患在代码里显式呈现出来。** 生产就绪评审里,"Cloud Run 公网匿名可调"是个 blocker 级安全项。在 Terraform 里它不再是一个藏在部署命令里的 flag,而是一段清清楚楚、能被 code review 看到的授权:

```hcl
# 【安全对照】--allow-unauthenticated 的真身: 给 allUsers 授 run.invoker = 公网匿名可调
resource "google_cloud_run_v2_service_iam_member" "order_api_public" {
  name     = google_cloud_run_v2_service.order_api.name
  location = google_cloud_run_v2_service.order_api.location
  role     = "roles/run.invoker"
  member   = "allUsers"   # ← 问题就在这一行,code review 一眼能看见
}
```

这正是 IaC 的一个隐性好处:**把"点一下就开了"的危险配置变成了必须写进代码、必须过 review 的东西**。`infra/main.tf` 里紧挨着这段就给了安全做法(把 `allUsers` 换成 agent 的运行时 SA),默认注释掉,让对照一目了然。

### 存量环境怎么纳管:import

本手册的读者大概率已经跟着前几章**手动建过一遍**这些资源了。这种情况下直接 `terraform apply` 会报"资源已存在"——因为 Terraform 的 state 是空的,它以为都还没建。正确做法是先 `terraform import` 把已存在的资源"认领"进管理:

```bash
terraform import google_storage_bucket.staging adk-fde-lab-staging
terraform import 'google_project_service.enabled["run.googleapis.com"]' adk-fde-lab/run.googleapis.com
```

**IaC 不是只能用在全新环境,存量环境靠 import 一样能纳管**——这一步最常用也最容易被忽略。

### 测试:分成"不碰云的单测"和"要凭证的集成测试"两层

关键的工程决策是把测试分两层,因为它们的运行条件完全不同:

- **单元测试**:不碰云、不需要凭证、几秒钟跑完。CI 里每次改动都能跑。
- **集成测试**:需要真实 GCP 凭证 + 线上资源,几十秒,用 `-m integration` 显式触发。

用 `pytest.ini` 把这个区分固化下来,默认只跑单测:

```ini
[pytest]
markers =
    integration: 需要真实 GCP 凭证(ADC)和已部署资源才能跑的集成测试
addopts = -m "not integration"
```

## 8.3 核心代码:测试到底测什么

测试的价值不在数量,在于**守住那些最容易被静默改坏、又最难用肉眼发现的东西**。项目里真实写的测试挑几条最有代表性的。

**单测:守住那条"RAG 工具不能和别的工具混挂"的硬约束(第 3 章的架构根因)。**

```python
def test_rag_tool_is_isolated_on_its_own_agent():
    for agent in (order_agent, policy_agent, root_agent):
        rag_tools = [t for t in agent.tools if isinstance(t, VertexAiRagRetrieval)]
        if rag_tools:
            assert len(agent.tools) == 1, (
                f"{agent.name} 上的 RAG 工具必须独占一个 agent,不能和别的工具混挂"
            )
```

这条约束如果被违反,是运行时才报错的;这个测试让它在 CI 阶段、几秒钟内就暴露,而不是等下次部署上线才炸。

**单测:守住订单 API 的工具契约不漂移(第 2、6 章反复强调的坑)。**

```python
def test_spec_declares_expected_operations():
    spec = _load_spec()
    operation_ids = {op["operationId"]
                     for path in spec["paths"].values()
                     for op in path.values() if "operationId" in op}
    assert operation_ids == {"get_order_status", "cancel_order"}
```

`openapi.json` 是导出时的快照,如果真实后端接口变了而快照没跟上,agent 的工具就会和后端静默错位。这条测试把"工具契约"变成可回归的——接口一旦对不上,CI 就红。

**集成测试:守住意图路由的正确性——这是最容易被"改一句 instruction"静默破坏的东西。**

```python
@pytest.mark.integration
def test_order_question_routes_to_order_agent():
    authors, text = _run_and_collect_authors("帮我查一下订单 1001 的状态")
    assert "order_agent" in authors      # 订单问题必须路由给 order_agent
    assert "policy_agent" not in authors # 且不该惊动 policy_agent
```

它真的用 `Runner` 在本地把 agent 跑起来,发一句订单问题,断言参与应答的是 `order_agent`。一个新工程师稍微改动 root 的 instruction 就可能把订单问题错误分派给政策 agent——这在少量手动测试里几乎发现不了,但这条断言会拦住。

这些不是纸面代码。写完实际跑了一遍,**9 条单元测试(2.5 秒,没碰云)+ 6 条集成测试(38 秒,真实调了 Gemini、订单 API、RAG、Model Armor)全部通过**。

### CI:把"测试→部署"串成有门禁的流水线

`.github/workflows/ci.yml` 定义三个 job:

```yaml
jobs:
  test:        # 1. 跑单元测试(不需要凭证)
    steps:
      - run: pip install -r requirements-dev.txt
      - run: pytest -v

  terraform:   # 2. 把 IaC 也纳入门禁: 格式/字段不对就红
    steps:
      - run: terraform fmt -check -recursive
      - run: terraform init -backend=false
      - run: terraform validate

  deploy:      # 3. 只在 main、且 test 通过后才跑 —— 这就是"门禁"
    needs: test
    if: false  # 配好 WIF、清掉硬编码密钥后改成 github.ref == 'refs/heads/main'
    steps:
      - uses: google-github-actions/auth@v2   # 用 WIF 认证,不下载密钥文件
        with:
          workload_identity_provider: ${{ secrets.WIF_PROVIDER }}
          service_account: ${{ secrets.DEPLOY_SERVICE_ACCOUNT }}
      - run: adk deploy agent_engine --project=... --otel_to_cloud ./customer_service_agent
```

`needs: test` 是关键词——**测试没过,`deploy` 这个 job 根本不会开始,坏代码上不了线**。这就是"门禁"和"手动部署"的本质区别。认证用的是 Workload Identity Federation,让 CI 不需要下载服务账号密钥文件(第 1 章优化方向提过的做法)。

!!! danger "启用 CI 之前必须先做两件事"
    这条流水线是"能用的模板",但真正 push 到 GitHub 启用前,有两件事必须先做——**它们本身就来自这个项目的生产就绪评审**:

    1. **清掉源码里硬编码的密钥**(`secret-key-123` 在 `main.py` 和 `agent.py` 里都有)。`.gitignore` 挡得住 `.env`,挡不住写死在 `.py` 里的字符串——一 push,密钥就随代码进了 git 历史。
    2. **配好 Workload Identity Federation**,让 CI 不需要下载 SA 密钥文件就能认证。

    在这两件事做好之前,`test` 和 `terraform` 两个 job 可以安全运行(它们不需要 GCP 凭证);`deploy` job 用 `if: false` 默认关着。

## 8.4 这些工具还能做什么

- **Terraform 远端 state + 加锁**:本地 `terraform.tfstate` 只适合一个人玩。团队协作必须把 state 放到 GCS(带锁),否则两个人同时 `apply` 会互相覆盖。`infra/versions.tf` 里已留了 GCS backend 的占位配置。
- **`terraform plan` 作为 PR 检查**:可以在 CI 里对 PR 自动跑 `plan` 并把"这次会改动哪些基础设施"贴回 PR 评论,让基础设施变更和代码变更一样经过 review,而不是谁在本地 `apply` 了别人都不知道。
- **pytest 的参数化与夹具(fixture)**:路由测试可以用 `@pytest.mark.parametrize` 一次性喂几十个不同问法,验证路由在各种表述下都稳定;夹具可以把"起一个 Runner"这种准备工作复用。
- **把评估接进 CI 当门禁**:第 7 章的评估目前是手动跑。可以把它做成一个 job,给关键指标设一个通过阈值(比如 `policy_accuracy` 均值低于 4.0 就 fail),让"质量退化"也变成一道自动门禁——不过要先解决第 7 章讲的评估样本太少、reference 与语料循环自证的问题,否则这道门禁本身不可信。
- **多环境用同一份 TF + 不同 tfvars**:`terraform apply -var-file=prod.tfvars` / `staging.tfvars`,用同一份基础设施定义复制出隔离的多套环境——这就是"环境隔离"能低成本落地的前提。

## 8.5 优化方向

- **收紧 IaC 里如实复刻的宽权限**:`main.tf` 里 `roles/editor`(给 Cloud Build)和 `roles/aiplatform.user`(给运行时 SA)都是当时快速解封的临时解法。生产环境应该用 `google_project_iam_custom_role` 定义只含所需 permission 的自定义角色——比如运行时 SA 其实只需要 `aiplatform.ragCorpora.query` 这一个权限,而不是整个 `aiplatform.user`。
- **把"没进 TF"的资源逐步纳管**:RAG Corpus、Model Armor / DLP 模板这类较新的产品,写作时 provider 原生资源覆盖还不稳定,`infra/README.md` 里给的过渡方案是用 `null_resource` + `local-exec` 把现有脚本包一层,至少纳入 `apply` 流程。等 provider 支持成熟了换成原生资源。**同时记住:IaC 复原的是基础设施骨架,不是数据**——corpus 能被重建成空的,里面那 3 份政策文档得靠单独的摄取流水线再灌一遍。
- **给集成测试也配一条独立的 CI**:当前 CI 只跑单测(不需要凭证)。可以再开一条定时或手动触发的流水线,配好 GCP 认证后跑集成测试 + 评估,作为"上线前的完整验证",而不是只靠开发者本地记得跑。
- **补上环境隔离**:测试和 CI 都就绪之后,下一步就是把 prod 拆成独立的 GCP 项目,用同一份 TF + 不同 tfvars 复制出来,让开发调试再也不会碰到客户在用的生产数据——这也是把前面几章所有"个人 Owner 账号、单项目"的风险一次性收敛的地方。
