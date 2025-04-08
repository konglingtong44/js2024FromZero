明白了，我将为你整理一份面向初学者、基于你提供的架构图的 AWS 全流程部署练习手册。

这份内容将包括：
- 架构图中所有服务模块的清单（共计40+个）
- 每个模块的搭建顺序建议与原因（例如先建网络 → 安全组 → 计算资源等）
- 控制台“点点点”操作指南：每个资源如何手动创建
- Lambda + API Gateway 实现多种方式（Lambda 调用 Fargate / EC2 / 数据库），以及按钮触发 Lambda 的示例 HTML 页面
- CloudFormation 脚本：至少包含基础网络架构 + 一个 EC2 + Lambda 示例
- 一键关闭脚本（停止 EC2、暂停 RDS、删除 ALB、终止 Fargate 等）
- 成本预警与优化建议：每个花钱的服务是否有更便宜或免费的替代方案（如 Aurora 用 RDS 替代）
- 最后还会附上一份完整的学习路径，建议你依照服务类别学习 AWS 相关技能

输出格式：普通文档 + Markdown 格式版本。

我整理好后尽快提供给你，请稍等。

# AWS 新手部署实战手册：基于 KongLing 架构

## 引言

本手册面向刚接触 AWS 的初学者，目标是**一步步搭建一个包含多种 AWS 服务的示例架构**，以熟悉常用云服务的部署与联动方法。该架构源自用户提供的 *KongLing* 系统示意图，涵盖了网络配置、负载均衡、弹性容器、云服务器、关系数据库、无服务器函数、API 网关、监控告警、网络防火墙、持续集成等模块。我们将主要通过 AWS 管理控制台的“点击式”操作完成部署，并提供相应的 CloudFormation 模板以供学习。同时，我们会标注每项服务是否有免费额度，并提醒哪些服务可能产生较高费用以及相应的低成本替代方案。最后，本手册还附上一键关闭所创建资源的脚本，方便在演示后暂停或终止计费，并提供一个按功能模块划分的**学习路径图**，帮助您逐步掌握 AWS 技能。

请确保您已注册 AWS 账户，并且有足够的权限进行资源创建和管理（建议使用具有管理员权限的 IAM 用户进行操作）。本指南将在 **AWS 东京区域 (ap-northeast-1)** 部署所有资源，请提前在控制台右上方选择该区域。

## 架构概览

 ([image]())*图：KongLing 示例系统在 AWS 云上的架构示意。该架构包含了多可用区的 VPC 网络、Internet Gateway、NAT Gateway、Application Load Balancer、AWS Fargate（ECS 容器）、Amazon EC2 实例、Amazon Aurora 数据库、Amaz ([A Simple Guide To AWS Lambda Pricing And Cost Management](https://www.cloudzero.com/blog/lambda-pricing/#:~:text=request%20or%20invocation%20,invoke%20call)) ([A Simple Guide To AWS Lambda Pricing And Cost Management](https://www.cloudzero.com/blog/lambda-pricing/#:~:text=,compute%20time%20per%20month%20free))API Gateway、AWS WAF、以及多个 AWS 开发者工具（CodePipeline、CodeComm ([Amazon API Gateway Pricing | API Management | Amazon Web Services](https://aws.amazon.com/api-gateway/pricing/#:~:text=The%20API%20Gateway%20free%20tier,for%20up%20to%2012%20months))odeDeploy 等）等模块。*

如上图所示，KongLing 架构将应用系统分布在一个 AWS 虚拟私有云（VPC）内的多个可用区 (Availability Zones) 中，以实现高可用。主要组件包括：

- **网络和安全**：一个自定义 VPC 划分多个子网（公有子网用于对外服务，私有子网用于内部服务和数据库），Internet Gateway 提供公网访问，NAT Gateway 提供私有子 ([Amazon API Gateway Pricing | API Management | Amazon Web Services](https://aws.amazon.com/api-gateway/pricing/#:~:text=The%20Amazon%20API%20Gateway%20free,the%20API%20Gateway%20usage%20rates))(Security Group) 用于控制入出流量。架构中还包括 AWS WAF (Web Application Firewall) 保护 Web 应用免受常见网络攻击。
- **负载均衡**：使用 Application Load Balancer (应用负载均衡，简称 ALB) 将来自互联网的请求分发到后端多台 ECS 容器实例或 EC2 服务器上，实现流量分发和故障切换。
- **计算层**：同时采用了 AWS Fargate（无服务器容器，在 ECS 集群中运行）和 Amazon EC2（弹性云服务器）来运行不同组件的应用程序。例如，某些微服务以容器形式运行在 Fargate 上，另一些传统应用可能运行在 EC2 实例上。
- **数据库与存储**：使用 Amazon Aurora （兼容 MySQL 或 PostgreSQL 的高性能托管数据库）存储核心业务数据。Aurora 集群包含主实例和只读副本，实现高可用和读写分离。架构中还展示了 Amazon S3、Amazon EFS 等存储服务（视具体需求选择）。
- **异步解耦**：示意图中包括 Amazon SQS（简单队列服务）和 Amazon SES/SNS（邮件和通知服务）等，用于异步消息和通知（本手册侧重主要部分，这些服务不在本次部署范围，可视需要自行探索）。
- **无服务器与 API**：采用 AWS Lambda 函数来运行按需的无服务器计算，并通过 Amazon API Gateway 提供统一的 RESTful API 接入。示例中将展示通过在 EC2 上的前端页面点击按钮，触发 API Gateway 调用后端 Lambda 函数并返回结果的流程。
- **监控与运维**：利用 Amazon CloudWatch 收集日志和指标，对各组件进行监控和告警通知。CloudWatch Logs 会记录 Lambda、EC2、ALB 等的日志，CloudWatch Alarms 可设置阈值告警。架构还包括 AWS CloudTrail（记录API操作日志）等保障运维审计安全。
- **持续集成/持续部署 (CI/CD)**：使用 AWS CodePipeline 编排代码从提交到部署的流水线。它可以集成 CodeCommit (源码仓库)、CodeBuild (构建)、ECR (容器镜像仓库)、CodeDeploy (部署到 ECS/EC2 或 Lambda) 等，实现自动化部署。本手册将展示创建一个简单的 CodePipeline，将代码更新部署到目标环境。

总体上，该架构涵盖了**网络、计算、存储、数据库、安全、集成、运维**等各方面功能，为初学者提供了一个全栈 AWS 环境的实践机会。下面，我们将按照合理的顺序逐一部署这些组件，并在过程中解释每个步骤的作用和注意事项。

> **关于部署顺序**：为了保证依赖关系正确，我们将先部署网络基础设施，然后部署数据库等后端服务，接着部署计算资源（EC2、ECS 和 ALB），再部署无服务器功能（Lambda 和 API Gateway）并实现前后端联动，最后配置安全和CI/CD等附加服务。请按照顺序进行，以免遇到资源依赖问题。

## 环境准备

在开始正式部署之前，请确保以下准备工作已完成：

- **AWS账户及权限**：您拥有一个可用的 AWS 账户，并以具有足够权限的身份登录 AWS 管理控制台。建议创建一个 IAM 用户并授予管理员权限用于此次实践，而不要直接使用根账户操作。
- **选择区域**：确认已将控制台区域切换到 **亚太地区（东京）ap-northeast-1**。本指南所有资源将创建在该区域，以利用其免费额度和低延迟。
- **SSH密钥对**：如果计划通过 SSH 连接 EC2 实例，您需要在东京区域预先创建或导入一个 EC2 密钥对 (Key Pair)。在控制台导航到 EC2 服务 -> 密钥对 -> 创建密钥对，并下载保存私钥文件（`.pem`）。稍后创建 EC2 时会用到该密钥对。**（如果不使用 SSH 登入，可忽略此项，但仍建议创建以备需要）**。
- **基础知识**：建议对以下概念有基本了解，将有助于理解操作步骤：VPC 和子网、路由和网关、安全组、Docker 容器、HTTP 协议基础、AWS 服务控制台使用方法。如果不熟悉，也没关系，本手册会在操作中简要解释。

一切准备就绪后，我们将按照步骤开始部署架构中的各个组件。

---

## 1. 网络环境配置 (VPC、子网、路由、网关)

首先，我们创建架构的网络基础，即 **VPC (Virtual Private Cloud，虚拟私有云)**。VPC 相当于您在 AWS 云上隔离出的一个逻辑网络空间，您可以在其中定义IP地址段、子网划分、路由规则等，就像在自有数据中心管理网络一样。所有后续的EC2实例、ECS容器和数据库等都将部署在这个 VPC 内。我们将在东京区域创建一个新的 VPC，并配置公有子网、私有子网、Internet网关和必要的路由，使架构支持对外提供服务和内部组件互通。

**步骤1：创建 VPC**

1. 登录 AWS 管理控制台，进入 “VPC 服务” 控制面板。在左侧导航栏点击“您的 VPC” (Your VPCs)。
2. 点击顶部的“创建 VPC”按钮。在出现的表单中，输入：
   - **名称标签 (Name tag)**：输入一个识别用名称，例如 “KongLing-VPC”。
   - **IPv4 CIDR 块**：输入网段，例如 `10.0.0.0/16`。这个CIDR代表整个VPC的私有地址范围 (我们选择 /16 可以容纳65k地址，足够支撑多个子网)。
   - 其他设置保持默认（暂不启用IPv6，租用默认 TENANCY）。
3. 点击“创建 VPC”。几秒钟后，新的 VPC 将出现在列表中。

> 🔹 *免费额度*: 创建和使用自定义 VPC 本身不收费。VPC 内的子网、路由表、安全组等均免费提供。但是，如果使用了 NAT Gateway 等网络组件会产生计费，我们稍后会讨论替代方案。

**步骤2：创建子网 (Subnets)**

按照最佳实践，我们将 VPC 拆分为**公有子网**（Public Subnet）和**私有子网**（Private Subnet）两类：公有子网中的资源可以直接访问互联网（通常用于放置 ALB、跳板机等需要公网访问的实例），私有子网中的资源没有直接公网访问能力（用于放置数据库、应用服务器等只需内网通信的实例）。

1. 在 VPC控制面板左侧，点击“子网”(Subnets)，然后点击“创建子网”。
2. 在“创建子网”页面，填写：
   - **名称标签**：为每个子网输入名称。例如创建两个公有子网 “KongLing-PublicSubnet-A” 和 “KongLing-PublicSubnet-C”，以及两个私有子网 “KongLing-PrivateSubnet-A” 和 “KongLing-PrivateSubnet-C”。（这里名称中包含AZ后缀A/C，表示分别在不同可用区）。
   - **VPC**：选择刚刚创建的 “KongLing-VPC”。
   - **可用区**：为实现高可用，我们将子网分散到多个 AZ。 ([Pricing - AWS WAF - Amazon Web Services (AWS)](https://aws.amazon.com/waf/pricing/#:~:text=Web%20ACL%20charges%20%3D%20%245,00%2Fmonth)) “ap-northeast-1a”，第二个公有子网在 “ap-northeast-1c”。私有子网同理分别选 1a 和 1c。
   - **IPv4 CIDR 块**：为每个子网指定一个 /24 段（256 地址）。例如：
     - KongLing-PublicSubnet-A -> ` ([AWS WAF vs. Cloudflare | Indusface Blog](https://www.indusface.com/blog/aws-waf-vs-cloudflare/#:~:text=AWS%20WAF%20vs.%20Cloudflare%20,Cloudflare%20provides%20unmetered%20DDoS))
     - KongLing-PublicSubnet-C -> `10.0.2.0/24`
     - KongLing-PrivateSubnet-A -> `10.0.101.0/24`
     - KongLing-PrivateSubnet-C -> `10.0.102.0/24`
     这样公私子网地址不同网段，便于识别（也留出了空间将来添加更多子网）。
3. 确认信息无误后，点击页面底部的“创建子网”按钮，批量创建上述4个子网。

子网创建完毕后，我们需要**开启公有子网自动分配公网IP**功能（使其内的EC2实例自动获得公网IP）。默认新子网不启用该功能。依次执行以下操作：
1. 在子网列表中，选中第一个公有子网 “KongLing-PublicSubnet-A”，在右侧的“子网详细信息”选项卡找到 “自动分配 IPv4 公共地址” (Auto-assign public IPv4) 设置，点击“编辑子网设置”，勾选“自动分配 IPv4 公共地址”，保存。
2. 同样对第二个公有子网 “KongLing-PublicSubnet-C”启用自动分配公网IP。
3. 私有子网则保持禁用公网IP（默认即为否）。

**步骤3：创建 Internet Gateway 并附加到 VPC**

Internet 网关允许 VPC 中的资源访问公网，也是外部网络流量进入 VPC 的通道。一个 VPC 需要关联一个 Internet Gateway 才能实现子网的公网连通。

1. 在左侧导航点击 “Internet 网关”，然后点击 “创建 Internet 网关”。
2. 为网关命名，例如 “KongLing-IGW”，然后点击创建。
3. 创建后，选中该 Internet Gateway，点击“操作”下拉菜单，选择“附加到VPC”，在对话框中选择我们的 “KongLing-VPC”，确认附加。

现在 VPC 有了出入口，但还需为公有子网配置路由，将出站流量引向 Internet Gateway。

**步骤4：设置路由表 (Route Table)**

每个子网关联一个路由表。默认 VPC 会有一个主路由表，但我们可以为公有和私有子网分别建立不同的路由策略。

- **公有子网路由表**：需要有一条将目的地 `0.0.0.0/0`（即所有公网地址）的流量指向 Internet Gateway。
- **私有子网路由表**：默认即可（仅本地VPC内部路由）。若私有子网需要访问外部，还需 NAT 配置，这里暂按无NAT处理。

具体操作：
1. 在左侧点击“路由表”，找到我们的 VPC 下自动创建的路由表（可以通过“VPC 列”确认）。默认名字可能为空或类似 “rtb-xxxxxxxx (KongLing-VPC)”。可以选中它，点击“编辑名称标签”命名为 “KongLing-MainRouteTable”。
2. 选中该路由表，查看下方“子网关联”。默认所有新子网都关联到了主路由表。我们计划将**公有子网**使用这个路由表，并添加IGW路由；而**私有子网**将使用单独路由表来控制不同路由策略。
3. 点击该路由表的“路由”选项卡，点击“编辑路由”。添加一条路由：
   - 目的地 (Destination)：`0.0.0.0/0`
   - 目标 (Target)：选择 Internet Gateway，并在下拉中选择之前创建的 “KongLing-IGW”。
   保存路由更改。这时，该路由表指向IGW的缺省路由生效。**此路由表现在适合作为公有子网的路由表**。
4. **创建私有路由表**：点击“创建路由表”，Name填入 “KongLing-PrivateRouteTable”，VPC选择我们的VPC。创建后，在“子网关联”选项卡，点击“编辑子网关联”，勾选我们两个私有子网 (KongLing-PrivateSubnet-A/C)，保存。这样这两个子网不再使用主路由表，而改用我们新建的私有路由表。
5. 由于私有子网此时没有通往互联网的路由，它们的实例将无法访问公网（除非我们稍后配置 NAT Gateway 或 VPC Endpoint）。本示例暂不为私有子网创建 NAT Gateway（因为 NAT 网关每小时收费且流量计费，在演示环境可能不必要）。若您的私有子网内资源需要访问外网更新软件，可考虑创建 NAT Gateway 或将实例临时放入公有子网。

> 🔸 *费用提示*: Internet Gateway 是免费的，但 **NAT Gateway** 会按时计费且按流量收费，在长期闲置情况下会产生不小的费用。如果只是学习环境，且私有子网内资源不需要主动访问互联网，可不创建 NAT 网 ([amazon web services - Is AWS Fargate included in their "first year for free" plan? - Server Fault](https://serverfault.com/questions/1085498/is-aws-fargate-included-in-their-first-year-for-free-plan#:~:text=4))3】如需私有子网也访问外网，一个低成本替代方案是在公有子网启动一台 NAT 实例 (使用开源 NAT 镜像) 来代替 NAT Gateway，但这需要自行管理且不如官方服务稳定。对于初学者，建议暂不使用 NAT Gateway 以控制成本。

**步骤5：配置安全组 (Security Groups)**

安全组是 VPC 层面的虚拟防火墙，可以附加到EC2、ECS任务、ALB等资源上。我们需要创建若干安全组来控制不同组件的流量：

- ALB 的安全组：允许公网 HTTP/HTTPS 流量进入负载均衡。
- EC2 的安全组：允许来自 ALB 或互联网的HTTP访问网站，以及允许管理访问（如SSH）。
- ECS任务的安全组：允许来自 ALB 的流量（如果 ALB 将流量转发到容器）。
- 数据库的安全组：允许应用服务器或Lambda访问数据库端口。
- Lambda 若在 VPC 内访问数据库，也需要安全组。

我们先创建这些安全组，但稍后在创建对应资源时再具体配置规则：

1. 在控制台顶部搜索进入 “EC2 服务” 面板，然后在左栏找到“网络和安全”下的**“安全组”**。
2. 点击“创建安全组”，创建以下安全组（每次填写完保存再新建下一个）：
   - **ALB-SG**：名称如“KongLing-ALB-SG”，描述填写“Allow HTTP from Internet”，VPC选择我们的 KongLing-VPC。创建后编辑其入站规则，添加规则：“类型”选HTTP，协议TCP，端口80，来源选0.0.0.0/0（允许所有IPv4地址）。这样 ALB 将允许所有来源的HTTP流量。*（如果需要HTTPS，可以类似添加443端口并配置证书，这里为了简单仅示范HTTP。）*
   - **EC2-SG**：名称如“KongLing-EC2-SG”，描述“Allow HTTP from ALB or Internet; SSH from my IP”，VPC同上。创建后添加入站规则两条：一是HTTP (TCP 80) 来源选 “0.0.0.0/0” （表示暂时允许任意来源访问网站。更安全的做法是仅允许ALB-SG的流量，我们稍后说明）；二是SSH (TCP 22) 来源选 “我的IP”（这样只有您的IP能通过SSH连接，保证安全）。
     - *高级*: 更佳安全实践是限制EC2的HTTP仅来自ALB，即来源选择“安全组”，填入刚创建的 ALB-SG。这要求用户不能直接通过EC2的公网IP访问，只能通过ALB访问。不过本次我们也希望直接访问EC2查看网站，所以暂允许所有来源访问80端口。演示完可改为仅 ALB 来源。*
   - **ECS-SG**：名称“KongLing-ECS-SG”，描述“Allow HTTP from ALB to ECS tasks”。入站规则：添加 HTTP (80) 来源选择刚创建的 ALB-SG 安全组。这表示只有来自 ALB 的流量能进到挂此SG的容器/实例。出站规则默认允许全部，可以保留。
   - **DB-SG**：名称“KongLing-DB-SG”，描述“Allow DB access from ECS/Lambda”，VPC同上。入站规则：选择 MySQL/Aurora (TCP 3306) 端口，来源选择我们创建的 ECS-SG 安全组（以及后续如有 Lambda 所用SG）。因为Aurora采用数据库端口3306，允许应用所在SG访问即可。如果稍后Lambda也访问数据库，还需要在此DB-SG添加来源为Lambda的SG（Lambda在VPC时必须关联SG）。

   创建这些安全组后，记下它们的名称或ID，后续创建资源时会用到。

至此，我们的网络环境配置完成。我们已有：**VPC** (10.0.0.0/16)、2个公有+2个私有**子网**跨越两个AZ、* ([CI/CD Pipeline - AWS CodePipeline - AWS](https://aws.amazon.com/codepipeline/#:~:text=One%20free%20active%20pipeline%20per,with%20the%20AWS%20Free%20Tier))way**及路由、以及基本的**安全组**。接下来，我们将部署各项云服务资源，将它们放置在这个网络之中。

---

## 2. 部署 Amazon Aurora 数据库

数据库通常是应用的支撑后端，因此我们优先创建数据库，方便稍后应用程序直接连接。架构中使用 **Amazon Aurora** 数据库，一种由 AWS 托管的高性能关系型数据库。Aurora 提供 MySQL 和 PostgreSQL 兼容引擎，我们可以选择熟悉的一种。Aurora 没有完全免费的套餐，但对小规模使用可以选用较小实例以降低成本。

**注意**：Aurora 数据库将部署在我们刚创建的 VPC 私有子网中，确保其不会暴露在公网上。同时，我们将使用之前创建的 DB-SG 来控制访问，仅允许来自应用服务器的连接请求。

**步骤1：创建 Aurora 集群**

1. 在控制台导航到 “RDS 服务”（Relational Database Service），在左侧菜单选择“数据库”。点击“创建数据库”按钮。
2. **选择引擎**：选中 “Amazon Aurora”。在引擎下拉中可以选择 “Aurora MySQL” 或 “Aurora PostgreSQL”。本示例以 Aurora MySQL 为例。（版本选择默认最新版即可）。
3. **模板**：选择“开发/测试”模板，这样默认值更适合小规模使用（会关闭一些多AZ冗余以节省资源）。
4. **设置**：
   - **集群标识符**：输入数据库集群名字，例如 “KongLing-Aurora-Cluster”。
   - **主用户名**：设置数据库管理员用户名，例如 “admin”。
   - **主用户密码**：输入两次密码，记下此密码（稍后应用需要用到）。
5. **实例配置**：
   - **实例类**：选择一个最小实例规格。如“db.t4g.micro”或“db.t3.small”（具体可用选项取决于当前Aurora版本和区域）。较小实例成本更低。
   - **实例标识符**：给实例起名字，例如 “KongLing-DBInstance-1”.
   - 保持只创建一个 writer 实例，**不**添加只读副本（开发模板默认就只建一个实例）。
6. **存储**：保持默认（Aurora存储会自动扩展，起始10GB）。
7. **连接**：
   - **VPC**：选择我们创建的 “KongLing-VPC”。
   - **子网组**：Aurora 需要一个子网组（包含多个子网以支持多AZ存储）。如果没现成的，点击“创建新的子网组”。在弹窗中输入子网组名称如 “KongLing-DBSubnetGroup”，选择VPC，添加可用区 ap-northeast-1a 和 1c 下的 **私有子网**（KongLing-PrivateSubnet-A/C）。保存子网组。
   - **公共访问**：选择“否”（不允许数据库有公共访问，因为我们放在私有子网，通过应用访问即可）。
   - **VPC安全组**：选择我们创建的 **DB-SG (KongLing-DB-SG)**。
   - **可用区**：可以留为空（自动分配）。Aurora集群的存储跨AZ，无需指定具体AZ。
8. 其他设置保持默认或可选：
   - **数据库名称**：可指定一个初始数据库名如 “KongLingdb”，不填则不会自动创建schema。
   - **密码验证**、**增强性能监控**等可根据需要，当前演示可关闭以节省费用。
   - **自动备份**保留默认（Retention Period 1 day）即可，备份少量不影响成本。
   - **删除保护**：建议**取消**勾选删除保护，方便演示结束后删除资源（否则需要先手动关闭保护才能删除实例）。
9. 点击“创建数据库”。控制台会开始创建Aurora集群和实例，这可能需要几分钟完成。在“数据库”列表中会看到状态从“创建中”变为“可用”。

等待数据库创建的同时，我们说明一些费用注意事项：

> 🔸 *费用提示*: **Aurora 本身没有免费额 ([Aurora Vs. RDS: Which AWS Database Solution Is Best?](https://www.cloudzero.com/blog/aurora-vs-rds/#:~:text=seven%20on%20RDS%20on%20Amazon,Aurora%20Serverless%20costs%20in%20advance))14】。运行一个最小规格的 Aurora 实例 (如 db.t3.small) 也会按小时收费，外加存储费用。不过存储起步10GB成本很低（几元每月级别）。Aurora 提供高可用和性能，但如果您只是学习或小项目，可以考虑使用 **Amazon RDS MySQL** 的免费套餐：新用户可获得一年的 db.t2.micro 实例免 ([Amazon RDS Free Tier - Database - AWS](https://aws.amazon.com/rds/free/#:~:text=Amazon%20RDS%20Free%20Tier%20,in%20the%20cloud%20for%20free))25】。Aurora 相比 MySQL 提供了分布式存储和更快复制，但也更昂贵且无法享受免费期。因此，在生产环境选择时，可权衡成本和性能。如果对高可用要求不高，另一个省钱方案是使用单个小型 EC2 实例自行部署 MySQL 数据库，但这需要自行运维备份，通常不推荐。

Aurora 创建完成后，我们可以在 RDS 控制台查看到**终端节点 (Endpoint)** 地址，它是应用程序连接数据库所需的主机名。可以点击集群名称，在“连接和安全”标签下找到终端节点和端口 (3306)。记下 **终端节点** 和 **端口**，以及之前设置的用户名密码。

稍后我们在应用程序（如部署的EC2或Lambda）中配置数据库连接时，需要用到这些信息。至此，数据库已就绪。

---

## 3. 部署 Amazon EC2 实例 (云服务器)

接下来，我们部署一台 **Amazon EC2 实例**（Elastic Compute Cloud）作为架构中的云服务器。这台  ([CloudWatch Pricing: A Straightforward 2024 Guide](https://www.cloudzero.com/blog/cloudwatch-pricing/#:~:text=match%20at%20L300%20,Virginia))单的网站前端，供用户访问，并提供一个按钮触发后端 Lambda 函数（模拟实际应用中前端调用后端的情形）。同时，我们也可以将其视作传统应用服务器，与Aurora数据库交互（本次不具体安装数据库客户端，但可选）。

我们将选择免费的实例类型并运行最新的 Amazon Linux 操作系统，通过 User Data 脚本自动安装一个简单的 Web 服务。例如，安装 Nginx 或 Apache 并放置一个包含按钮的示例网页。

**步骤1：启动 EC2 实例**

1. 进入 AWS 控制台的 EC2 服务面板。点击上方的“启动实例” (Launch Instance) 按钮。
2. **设置基本配置**：
   - **名称**：例如填入 “KongLing-WebServer”。
   - **AMI**：选择操作系统镜像。这里点击“快速开始”里的 Amazon Linux 2 AMI (64-bit x86)。Amazon Linux 2 是 AWS 提供的常用Linux发行版，免费且与AWS服务集成良好。
   - **实例类型**：选择 “t2.micro”（1 vCPU, 1GB内存）。这是免费套餐允许 ([Free Container Services - AWS](https://aws.amazon.com/free/containers/#:~:text=12%20MONTHS%20FREE))98】。
   - **密钥对**：在 “密钥对(登录)” 下拉中，选择您在准备阶段创建的密钥对名称（如没有，请创建一个新的）。这将用于 SSH 登录EC2。
   - **网络设置**：
     - VPC：选择我们创建的 “KongLing-VPC”。
     - 子网：选择一个 **公有子网**（KongLing-PublicSubnet-A 或 C）。选择其中一个AZ部署即可，比如 1a。
     - 自动分配公有 IP：应为“启用”，如果之前子网设置正确这里会自动启用。
     - **安全组**：选择“选择现有安全组”，勾选我们创建的 “KongLing-EC2-SG”。这样EC2实例将应用我们预设的规则（允许80和22端口访问）。
   - **存储**：默认给8GiB gp2型EBS即可，免费套餐足够涵盖。
3. **高级细节 - 用户数据 (User Data)**：
   向下展开“高级详细信息”。在“用户数据”文本框中，我们可以提供一段脚本，使EC2在首次启动时自动执行。填入以下User Data脚本（选择“文本”模式粘贴脚本）：
   ```bash
   #!/bin/bash
   yum update -y
   amazon-linux-extras install -y nginx1
   systemctl enable nginx
   systemctl start nginx
   # 创建一个简单的网页
   echo "<h1>KongLing Demo Site</h1>
   <p>欢迎访问 KongLing 示例站点。这是部署在 EC2 上的网页。</p>
   <button onclick='invokeLambda()'>点我调用 Lambda</button>
   <p id='lambdaResult'></p>
   <script>
     async function invokeLambda() {
       try {
         const response = await fetch('<<API_URL>>');
         const data = await response.text();
         document.getElementById('lambdaResult').innerText = 'Lambda 返回: ' + data;
       } catch (error) {
         document.getElementById('lambdaResult').innerText = '调用失败: ' + error;
       }
     }
   </script>" > /usr/share/nginx/html/index.html
   ```
   这段脚本将：
   - 更新软件包索引并安装 Nginx。
   - 开启 Nginx 服务，并设置开机自启。
   - 覆盖默认网页内容为一个简单的 HTML，包含一个按钮和一段脚本。点击按钮将调用 `invokeLambda()`，该函数通过 JavaScript 的 `fetch` 调用一个 API URL，并将响应数据显示在页面上。**注意**：其中 `<<API_URL>>` 是占位符，我们稍后在Lambda和API Gateway配置完成后，会将实际的 API Gateway 终端URL替换进来并更新网页。
   
   点击“启动实例”。确认您理解密钥下载事项并执行启动。
4. 返回 EC2实例列表，可以看到新实例正在启动。当状态变为“正在运行 (running)”且通过状态检查后，表示已启动成功。

**步骤2：验证 EC2 Web 服务**

1. 在实例列表中，点击您的 “KongLing-WebServer” 实例，查看详情。在“网络”栏找到 **公有 IPv4 地址** 或 **公共 DNS**。复制该地址。
2. 在浏览器中访问 `http://<公有IP>`，应看到刚才设置的简单网页，显示标题“KongLing Demo Site”，下面有一段欢迎词和一个“点我调用 Lambda”的按钮。
3. 现在点击这个按钮，目前可能不会有实际返回，因为我们尚未配置 Lambda 和 API Gateway，`<<API_URL>>` 还是占位符。稍后我们将在配置好 API Gateway 后，将该 URL 更新到网页中。此时点击按钮可能会报请求错误，这是预期的，因为 URL 无效。

如果页面没有显示，可能是安全组或User Data设置问题：
- 确认 EC2 实例附加的安全组 (KongLing-EC2-SG) 入站规则有允许 0.0.0.0/0 的 TCP 80。如果之前限定来源为 ALB-SG，则目前直接访问可能被拒绝。可以临时放宽来源为 0.0.0.0/0 或用 ALB 测试。
- 确认 Nginx 服务已启动。如果需要，可通过SSH登录EC2检查 web 服务是否运行（使用密钥SSH登录，命令：`curl localhost` 查看是否返回 HTML）。

至此，我们的 EC2 Web服务器已经对外可访问，并提供了一个网页界面。在后续步骤配置好 API Gateway 和 Lambda 后，我们会回来让这个页面与后端联动。现在继续部署其它组件。

> 🔹 *免费额度*: EC2 的 t2.micro 实例对于新注册用户每月有750小时的免费额度（ ([Free Container Services - AWS](https://aws.amazon.com/free/containers/#:~:text=12%20MONTHS%20FREE))398】。请尽量使用免费操作系统（如 Amazon Linux、Ubuntu 等）以避免许可费用。EBS 硬盘有 30GB 的免费额度。**费用提示**：EC2 若长时间开机但不使用会产生费用，建议在不需要时关机或释放。另外，对于静态网站，也可以使用更便宜的方案如 **Amazon S3 静态网站托管**，完全免服务器，成本极低（免费容量5GB）。本例用EC2主要是演示计算服务。若仅托管静态网页触发Lambda，S3+CloudFront会更划算。

---

## 4. 部署应用负载均衡 ALB (Application Load Balancer)

现在我们部署 **Application Load Balancer (ALB)** 来作为流量入口。ALB 将接收用户的HTTP请求，并将其分发给后端的多个目标。例如，它可以将请求转发到我们之前启动的EC2实例，或转发到后续部署的ECS容器服务，实现负载均衡和冗余。在本架构中，我们计划主要将 ALB 用于分发给 ECS Fargate 容器任务，但也可以用它来代理EC2的流量。

我们之前已经创建了 ALB 所需的基本网络（VPC、公有子网）和一个 ALB 专用安全组 (ALB-SG) 允许80端口。下面通过控制台创建 ALB 并配置其监听和目标组。

**步骤1：创建目标组 (Target Group)**

在创建 ALB 之前，先创建一个目标组，用于注册 ALB 后端的目标实例或IP。由于我们将使用 ECS Fargate（IP模式）以及可能也包含EC2实例，所以目标类型应选择 “IP”。

1. 打开 EC2 服务控制台，在左侧导航栏找到“负载均衡 -> 目标组”并点击。
2. 点击“创建目标组”：
   - **目标组名称**：如 “KongLing-TG”.
   - **目标类型**：选择 “IP 地址”。
   - **协议**：HTTP，端口 80。
   - **VPC**：选择我们的 KongLing-VPC。
   - **健康检查**：保留默认 HTTP，路径为 “/”即可（Nginx默认index页面）。健康检查用于ALB判断后端是否正常。
3. 点击“创建目标组”。暂不用注册目标实例，稍后我们让 ECS 服务自动注册。

**步骤2：创建 ALB**

1. 在 EC2控制台左侧，点击“负载均衡器”，然后点击“创建负载均衡器”。选择 **Application Load Balancer** 类型。
2. **基本配置**：
   - **名称**：输入 “KongLing-ALB”。
   - **方案 (Scheme)**：选择 “Internet-facing”（互联网可见），因为我们需要接受公网请求。
   - **IP 地址类型**：选择 “IPv4”。
3. **网络映射**：
   - **VPC**：选择我们的 KongLing-VPC。
   - **可用区映射**：选择至少两个公有子网，将 ALB 部署在不同 AZ 实现高可用。勾选 ap-northeast-1a 和 1c 中的 **公有子网**（KongLing-PublicSubnet-A 和 KongLing-PublicSubnet-C）。确保每个选择框右侧都选中了对应子网的 ID。
4. **监听器和路由**：
   - 默认已经有一个监听器：协议 HTTP，端口 80。确认无误。
   - **目标组**：这里我们可以直接选择之前创建的 “KongLing-TG” 作为默认目标组。如果没看到，点“新建”也可以创建，但我们已有则直接选择之。
5. **安全组**：
   - 选择已有安全组，勾选我们创建的 “KongLing-ALB-SG”。它开放了80端口给公网。
6. 其余高级设置保持默认，点击底部“创建负载均衡器”。

ALB 创建需要半分钟左右。创建成功后，在“负载均衡器”列表会出现 “provisioning”，稍后变为“active”。点选 ALB 可以查看其 **DNS 名称**，类似 `KongLing-ALB-xxxxxxxx.ap-northeast-1.elb.amazonaws.com`。这个 DNS 一旦 Active 就可以在浏览器访问，但目前因为无注册后端，会返回503。然而我们可以尝试后续将EC2注册进去测试。

**（可选）步骤3：将 EC2 实例注册到 ALB 进行测试**

在我们部署 ECS 之前，可以先临时把已有的EC2实例注册到目标组，以验证 ALB 工作：

1. 点击左侧“目标组”，选中 “KongLing-TG”，进入“目标”选项卡，点击“注册目标”。
2. 下拉中 **目标** 选择我们的 EC2 实例ID（KongLing-WebServer），端口填80，点击添加至列表，然后保存注册。
3. 几十秒后，刷新页面，应看到EC2的状态变为“healthy”（通过健康检查）。
4. 现在复制 ALB 的 DNS名称，在浏览器访问 `http://KongLing-ALB-...elb.amazonaws.com`，应该看到和直接访问EC2相同的页面。如果正常，说明 ALB 已成功转发流量给EC2实例。

完成测试后，可选择将EC2从目标组移除（保持空）或者留着也可以。稍后当我们有ECS任务加入，该目标组会同时有多个目标。

**小结**：现在我们已经有一个Internet-facing的 ALB 监听80端口，并关联到KongLing-TG目标组。接下来，当我们部署 ECS服务时，会将其任务的IP自动注册到这个目标组中，从而通过 ALB 访问 ECS 容器。

> 🔸 *费用提示*: ALB 按运行时间和流量收费，没有免费套餐。大约每小时收费约 $0.022，在一个月持续运行约 $16，再加上流量费用（按L ([Pricing - AWS WAF - Amazon Web Services (AWS)](https://aws.amazon.com/waf/pricing/#:~:text=Web%20ACL%20charges%20%3D%20%245,00%2Fmonth))1-L4】。如果仅有很低的流量或测试用途，这可能相对昂贵。低成本替代方案：对于简单应用，可直接使用 EC2 的弹性IP或 CloudFront+S3 托管网站，不需要负载均衡。但无法提供多实例高可用。如果您的应用只有一个服务器，省去ALB可节约成本；当需要水平扩展时再引入ALB。此外，AWS 还提供轻量的 **Application Load Balancer - 基于请求定价** 模式，或使用 **AWS API Gateway** 代替ALB来处理HTTP请求（API Gateway对小流量非常便宜，见后文）。

---

## 5. 部署 ECS Fargate 容器服务 (Amazon ECS on Fargate)

现在进入云计算的现代领域：**容器**。我们将使用 AWS 的 Elastic Container Service (ECS) 搭配 **Fargate** 模式来运行容器化的应用。这让我们无需管理EC2服务器，直接运行容器，AWS 会在后台按需提供计算资源。我们的目标是在 ECS 中部署一个简单的Web服务容器，并通过 ALB 对外提供服务，实现与EC2类似的功能，但具备更好的扩展性。

具体来说，我们将在 ECS 内：
- 创建一个 ECS 集群 (Cluster)。
- 定义一个 Task Definition（任务定义），指定容器镜像和运行参数。
- 使用 Fargate Launch Type 创建一个 Service，根据任务定义启动若干任务（容器实例），并将它们注册到 ALB 目标组，实现负载均衡。

为简化步骤，我们将使用 **AWS 提供的示例容器镜像** 来部署（一个简单的NGINX网页），这样不需要自行构建镜像。AWS 有一个公共ECR镜像 `amazon/amazon-ecs-sample` 供演示使用，我们就用它。

**步骤1：创建 ECS 集群**

1. 打开 AWS 控制台的 **ECS (Elastic Container Service)**。在左侧菜单，点击“集群”。
2. 点击 “创建集群”。在选项中选择 “Networking only (Fargate)” 模板（即只创建一个空集群，专用于Fargate）。
3. **集群名称**：输入 “Oddsp ([Aurora RDS instance can not be stopped - Stack Overflow](https://stackoverflow.com/questions/43972237/aurora-rds-instance-can-not-be-stopped#:~:text=Aurora%20RDS%20instance%20can%20not,Unfortunately%2C%20some))。
4. 其他默认即可，点击 “创建”。几秒后，一个新的集群就创建好了。

（Fargate 集群实际上不需要预置容器实例，所以集群创建只是一个逻辑组织。）

**步骤2：创建任务定义 (Task Definition)**

1. 在ECS左侧，点击“任务定义”，然后点击“创建新的任务定义”。
2. 选择 “Fargate” 类型，点击下一步。
3. **任务定义名称**：输入 “KongLingTask”.
4. **任务执行角色**：选择 `ecsTaskExecutionRole`（如果不存在，可以按照提示创建一个。这个角色允许任务拉取ECR镜像和发送日志到CloudWatch）。
5. **Fargate 配置**：平台版本选择 “LATEST”，兼容操作系统 Linux。
6. **任务大小**：为任务分配CPU和内存。选择例如 “0.25 vCPU”和“0.5 GB Memory”（非常小的规格即可跑我们的示例NGINX）。
7. **添加容器**：在“容器定义”部分，点击“添加容器”按钮，设置：
   - **容器名称**：如 “webapp”.
   - **镜像**：输入 `amazon/amazon-ecs-sample` （这是一个公共Docker Hub镜像的名称，无需前缀）。
   - **端口映射**：容器端口填入 80，协议 TCP。（这个镜像在容器内用80端口提供服务）。
   - 点击“添加”。
8. 其它设置（环境变量、日志）可以保持默认。滚动到底部点击“创建”。

任务定义创建完成后，您可以点开它查看详情，包括刚刚配置的容器和资源参数。

**步骤3：创建服务 (Service) 并将容器附加到 ALB**

1. 在ECS左侧菜单，点击“集群”，进入 “KongLing-Cluster” 页面。点击上方的“创建” -> 选择 “创建服务”。
2. **配置服务**：
   - **启动类型**：选择 “FARGATE”。
   - **任务定义**：选择刚刚创建的 “KongLingTask:1” （版本1）。
   - **集群**：确保为 “KongLing-Cluster”。
   - **服务名称**：输入 “KongLing-Service”。
   - **任务数量**：输入 “2”。（示例启动2个容器任务，以演示负载均衡效果。您也可填1，仅启动单任务）。
   - **部署选项**：保留滚动部署默认。
3. **网络配置**：
   - **子网**：选择我们 VPC 中的 **私有子网**（KongLing-PrivateSubnet-A 和 C）。这里我们将容器任务运行在私有子网，以加强安全（它们不需要直接公网IP，由ALB代转发流量）。务必选择 **两个不同AZ** 的子网，以便2个任务各跨一AZ，实现高可用。
   - **安全组**：选择我们创建的 **ECS-SG (KongLing-ECS-SG)**。它允许 ALB 的流量进来。
   - **自动分配公有 IP**：选择 “DISABLED”（任务运行在私有子网，无需直接公有IP）。
4. **负载均衡**：
   - 选择 “Application Load Balancer”。
   - **负载均衡器名称**：选择我们创建的 “KongLing-ALB”。
   - **监听端口**：选中 Listener `HTTP:80`.
   - **目标组名称**：选择之前创建的 “KongLing-TG”。当出现容器的Port配置时，选择容器“webapp:80:80”。
   - 点击 “添加至负载均衡器”。下方应出现一行，表示服务将使用 ALB 监听HTTP端口，将容器80端口流量注册到目标组KongLing-TG。
   - **服务发现**部分留空（不使用 Cloud Map）。
5. **其他设置**：
   - **部署角色**保留 ECS默认。
   - **单击任务 IAM 角色**如果有其他需求可设置，这里不需要（为空即用任务定义里execution role）。
   - **策略**可以保持默认（重试次数等）。
6. 点击“创建服务”。

ECS 服务开始创建，会启动2个任务。您可以在集群的“Tasks”选项卡看到新任务的状态，从 PROVISIONING -> PENDING -> RUNNING。每个任务启动后，将自动把其 IP 地址注册到 ALB 的目标组中（这个过程由 ECS服务和ALB的集成自动完成）。几分钟内，服务应显示为稳定的运行2个任务。

**步骤4：测试 ECS 服务通过 ALB 提供内容**

ECS 任务使用了示例容器 `amazon/amazon-ecs-sample`，这个容器其实运行了一个简单的网站（会显示 “Congratulations!” 的测试页）。我们通过 ALB 的 DNS 来访问它：

1. 获取 ALB 的 DNS名称（如先前所示，可在 EC2负载均衡界面或在ECS服务的负载均衡详情中找到）。
2. 在浏览器访问 `http://<ALB_DNS_NAME>`。现在您应该看到一个 ECS 示例的网页，上面写着“Congratulations! You have successfully deployed the Amazon ECS sample.”之类的字样。每刷新一次，ALB可能会将请求发送到不同的容器实例，但因为内容相同所以看不出差异。可在 ALB Target 健康检查或CloudWatch里看到各目标请求数。
3. 这证明我们的 ALB + Fargate 服务已经成功运行！它与先前EC2的网页并不冲突，因为 ALB 默认路由所有请求到我们配置的 target（ECS任务），EC2即使仍在目标组中也能协同，但建议避免混用以清晰分流。我们可以把EC2从目标组移除，专留ECS的目标。

现在，我们拥有了一个通过 ALB 对外的高可用Web服务，由2个无服务器容器实例支撑。这模拟了真实生产环境利用容器部署应用的模式。接下来，我们将继续设置 Lambda 和 API Gateway，完成无服务器部分的功能，并演示与前端页面的交互。

> 🔹 *免费额度*: **ECS 服务本身免费**，但使用 Fargate 运行容器会根据 vCPU和内存小时收费。Fargate **没有 ([amazon web services - Is AWS Fargate included in their "first year for free" plan? - Server Fault](https://serverfault.com/questions/1085498/is-aws-fargate-included-in-their-first-year-for-free-plan#:~:text=4))-L142】。相对于用 EC2 自行承载容器，Fargate 按秒计费的灵活性更好，但小负载下可能成本较高。如果要省钱，您可以考虑使用 ECS 的 EC2 模式：即在一个 EC2 t2.micro 实例上运行容器，这样利用 EC2 免费额度，相当于免费运 ([amazon web services - Is AWS Fargate included in their "first year for free" plan? - Server Fault](https://serverfault.com/questions/1085498/is-aws-fargate-included-in-their-first-year-for-free-plan#:~:text=Fargate%20is%20not%20on%20the,work%20in%20a%20similar%20way))-L140】。对于长期运行的小型工作负载，**直接用Lambda替代**也是一种方法：如果业务可以拆成函数请求，Lambda 每月100万请求免费，更经济（无需一 ([amazon web services - Is AWS Fargate included in their "first year for free" plan? - Server Fault](https://serverfault.com/questions/1085498/is-aws-fargate-included-in-their-first-year-for-free-plan#:~:text=performance,in%20a%20fast%20car))-L144】。在本示例中，我们使用2个Fargate任务只是演示，当不需要时可以随时缩容或删除服务以节省费用。

---

## 6. 部署 AWS Lambda 函数 和 API Gateway (无服务器后端)

现在我们来部署**AWS Lambda**函数，并通过**Amazon API Gateway**来触发它。Lambda 是 AWS 的无服务器计算服务，支持按需运行代码而无需预置服务器。API Gateway 则可以将HTTP请求转换为对Lambda的触发，从而让前端通过标准HTTP接口调用后端函数。

在我们的架构中，这一步将实现：用户点击EC2网页上的按钮 -> 浏览器调用 API Gateway 提供的 HTTP 接口 -> API Gateway 转发请求给 Lambda 函数执行 -> 将结果返回给浏览器显示。我们会构建一个简单的 Lambda 函数，比如返回当前时间或一个问候语，以验证链路。

**步骤1：创建 Lambda 函数**

1. 打开 AWS 管理控制台的 **Lambda 服务**。点击“创建函数”。
2. **函数配置**：
   - **函数名称**：输入 “KongLingFunction”。
   - **运行时**：选择您熟悉的语言运行时。这里选择 Python 3.9 为例。
   - **权限**：展开“更改默认执行角色”。选择 “创建一个新角色，带有基本的 Lambda 权限”。（这样会自动创建一个角色授予 Lambda 写 CloudWatch Logs 的权限）。
   - 其余保持默认，点击 “创建函数”。
3. 函数创建成功后，会进入函数的配置页面。在页面中间可以看到一个代码编辑器区域，我们在此填写代码：
   - 找到 `lambda_function.py` 或类似默认文件（取决于语言）。将其内容改为以下简单代码：
     ```python
     def lambda_handler(event, context):
         # 这个函数简单返回一个问候语或当前时间
         import datetime
         now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
         return {
             "statusCode": 200,
             "body": f"Hello from Lambda! Current time is {now}"
         }
     ```
     这段 Python 代码每次被调用会返回状态码200和一段文本消息，包含当前时间戳。您也可以定制不同逻辑，比如读取数据库等（需配置权限和VPC，不在此展开）。
   - 点击右上方的“部署”按钮，将代码保存部署。

4. **测试函数**：在上方点击“测试”选项卡，创建一个测试事件（随便填入默认的HelloWorld事件数据即可），然后点击测试。下方应显示执行结果，状态Code 200，并在日志中看到返回的消息字符串。这表示函数能够成功运行。

> ℹ️ *关于 Lambda VPC 配置*: 我们的函数逻辑不需要访问Aurora数据库，所以未将Lambda放入VPC。如果您希望Lambda直接访问VPC内的资源（如Aurora），需要在函数配置中的“VPC”设置里选择 VPC和子网以及一个安全组（可使用之前DB-SG），这样Lambda执行环境就会连入您VPC。但请注意，这可能引入冷启动延迟，且需确保Lambda的SG和DB-SG互相允许通信。本例为简化，不演示Lambda直连数据库。如需实现，可稍后自行尝试。

**步骤2：创建 API Gateway 并集成 Lambda**

1. 打开 **API Gateway 服务** 控制台。点击“创建 API”，选择 “REST API” 类型（由于我们需要通过简单REST接口调用，REST API适合；也可用HTTP API简化，但控制台指导以REST API为例）。
2. **设置 API**：
   - **API名称**：如 “KongLingAPI”。
   - **端点类型**：选择 “区域” (Regional)。
   - 点击 “创建 API”。
3. 在左侧资源树上，会看到一个根资源 “/”。我们需要在其下创建一个子资源和方法代表我们的后端接口：
   - 选中 “/”，点击 “操作” 下拉，选择 “创建资源”。输入：
     - **资源名称**：如 “hello” （代表我们的接口路径）。
     - **资源路径**：自动填充为 “hello”。
     - 勾选 “启用 API Gateway CORS” 选项，以便简化跨域设置（前端网页不同源调用API需要CORS）。
     - 点击 “创建资源”。
   - 接下来确保新创建的 “/hello” 资源被选中，点击 “操作 ([amazon web services - Is AWS Fargate included in their "first year for free" plan? - Server Fault](https://serverfault.com/questions/1085498/is-aws-fargate-included-in-their-first-year-for-free-plan#:~:text=4)) ([Aurora Vs. RDS: Which AWS Database Solution Is Best?](https://www.cloudzero.com/blog/aurora-vs-rds/#:~:text=seven%20on%20RDS%20on%20Amazon,Aurora%20Serverless%20costs%20in%20advance))下拉中选择 “GET”，然后打勾确认。
   - 配置 GET 方法的集成：
     - **集成类型**：选择 “Lambda 函数”。
     - **使用Lambda代理集成**：勾选（这样可直接将整个请求传给Lambda，由Lambda自行处理输入输出）。
     - **区域**：选择 “ap-northeast-1”。
     - **Lambda 函数**：输入刚才创建的函数名称 “KongLingFunction”，在出现的下拉中选中它。
     - 点击“保存”。弹窗会提示授权API Gateway调用Lambda的权限，点击 “OK” 自动添加权限。
   - 现在 GET /hello 方法已集成 Lambda。您会看到 Method Execution 详细页。可以点击 “测试” 按钮直接在 API Gateway 中测试此方法是否返回正确。
     - 点击“测试”，然后点击 “测试”下的蓝色按钮发送请求。如果集成成功，右侧会显示测试结果，其中**响应正文**应是 Lambda 返回的字符串（带时间的 Hello 消息），HTTP状态码200。

4. **部署 API**：
   - 在左侧 “操作” 下拉中选择 “部署 API”。
   - **部署阶段**：选择 “[新阶段]”，输入阶段名称，比如 “prod”（表示生产环境）。
   - 点击 “部署”。 
   - 部署后，将跳转到一个阶段详情页面。在那里可以看到 “Invoke URL” （调用URL），类似于 `https://<random-id>.execute-api.ap-northeast-1.amazonaws.com/prod`。
   - 记下这个 Invoke URL。完整的调用地址包括资源路径，比如我们的GET方法是 `/hello`，因此完整URL为：
     ```
     https://<api_id>.execute-api.ap-northeast-1.amazonaws.com/prod/hello
     ```
   这个URL即为前端需要调用的地址。

5. **（可选）配置 CORS**：
   - 如果在创建资源时勾选了启用CORS，一部分CORS配置已经自动完成。但为了保险，我们可以手动配置CORS响应：
   - 在左侧资源树中，选中 “/hello”，点击 “操作” -> “启用 CORS”。保持默认选项，点击 “启用”。API Gateway 会自动在该资源下创建一个 OPTIONS 方法并添加适当的响应头（允许所有来源调用等）。
   - 再次部署API（选择已有阶段 prod）使CORS配置生效。

**步骤3：将 API URL 集成到前端页面**

现在我们有了 API Gateway 的URL，可以在EC2网页中调用它了。接下来需要把EC2上的网页JS中的 `<<API_URL>>` 替换为实际的 API 调用URL。这可以通过多种方式实现，例如重新打包AMI或通过SSH修改文件。不过作为演示，我们可以直接SSH登录EC2修改，或更简单地，使用 SSM Session Manager 无需密钥登录修改文件。

这里假设使用 SSH：
1. 使用之前的密钥，通过SSH连接 EC2 实例：
   ```bash
   ssh -i YourKey.pem ec2-user@<EC2-Public-IP>
   ```
2. 编辑网页文件：
   ```bash
   sudo vim /usr/share/nginx/html/index.html
   ```
   找到里面的 `fetch('<<API_URL>>')`，将 `<API_URL>` 替换为刚才记录的完整Invoke URL加`/hello`路径。例如：
   ```js
   fetch('https://abcd1234.execute-api.ap-northeast-1.amazonaws.com/prod/hello')
   ```
   保存退出。
3. 或者，也可以直接在User Data里写好API URL然后重启实例让User Data重新执行（Amazon Linux默认User Data只运行一次，手动重跑需要3. 保存后，在浏览器刷新 EC2 网站页面。点击“点我调用 Lambda”按钮，此时应成功触发 Lambda，并在页面下方显示类似“Lambda 返回: Hello from Lambda! Current time is 2025-04-08 17:17:00”的字样（内容和时间以您的 Lambda 实际返回为准）。这说明前端通过 API Gateway 调用了后端 Lambda，并拿到了响应。

**故障排查**：如果点击按钮没有反应或在浏览器控制台看到CORS错误：
- 确认 API 已部署，并使用正确的 URL（包括了阶段和资源路径）。
- 确认 API Gateway 已启用CORS。在浏览器网络请求中查看响应头，看是否包含 `Access-Control-Allow-Origin: *` 等。如果没有，重新在 API Gateway 控制台启用CORS并部署。
- 确认 Lambda 正常返回了内容而不是错误。在 API Gateway 控制台的 CloudWatch 日志或 Lambda CloudWatch 日志中检查调用记录，寻找报错信息并修正代码。

完成以上步骤，我们实现了**从前端网页 -> API Gateway -> Lambda 后端**的调用链路。用户点击按钮，就通过 API Gateway 间接触发 Lambda 完成计算并返回结果。这种模式在无服务器架构中非常常见，使前后端解耦且具有极高伸缩性。

> 🔹 *免费额度*: **Lambda** 每月前 100万 次调用免费，计算时间 40万GB-毫秒免费。一般小项目几乎用不完，非常优惠。**API Gateway** 每月也有前 100万 次调用免费（12个月内）。因此对于低流量应用，Lambda+API Gateway 几乎可以零成本运行。此外，它们按调用计费，不像EC2/ECS需要持续运行资源，闲时无费用，非常适合间歇工作负载。**费用提示**：若对实时性要求不高且访问量小，可以考虑用纯Lambda取代持续运行的EC2/ECS服务，将成本降到最低。但如果持续高并发请求，API Gateway 超过免费额度后按百万次/$3.5收费，此时使用ALB固定成本可能更经济。架构设计需根据流量水平选择合适方案。

---

## 7. Lambda + API Gateway + EC2 联动示例

在前面的步骤中，我们已经搭建了 EC2 前端、API Gateway 和 Lambda 后端，并使它们互相通信。本节我们综合梳理这一联动过程，帮助理解架构各部分如何协同工作：

- **前端 (EC2)**：运行着一个简单的网站（由 Nginx 提供），包含 HTML/JavaScript。当用户点击按钮时，浏览器通过 JavaScript `fetch` 发起 AJAX 请求，URL 指向 API Gateway 提供的endpoint。
- **API 网关**：充当了前端和后端之间的**门面**。它将HTTP请求转换为对后端 Lambda 的调用。由于配置了 Lambda Proxy 集成，API Gateway 会将整个请求事件传递给 Lambda，并期待 Lambda 按格式返回。我们在 API Gateway 中启用了 CORS，这样浏览器允许前端域名访问不同域下的资源（API域名）。
- **Lambda 函数**：作为后端处理程序，被触发后执行自定义代码。本例中只是返回问候语和时间戳。Lambda 将结果通过返回值输出，API Gateway 收到后封装成HTTP响应发送回调用方。
- **前端接收响应**：浏览器收到 API Gateway 的HTTP响应，在我们的代码中，`invokeLambda()` 的 Promise 得到结果文本，然后将其插入页面，从而用户看到按钮触发后返回的数据。

这一系列步骤体现了 Serverless 架构的优势：前端无缝调用后端函数，而无需关心后端服务器部署，Lambda 根据请求触发按需执行，API Gateway 提供标准化HTTP接口。在生产环境，可以利用这一模式构建高度弹性的微服务。例如，将繁重任务由Lambda完成（甚至异步通过SQS、SNS触发），前端只需调用API获取结果。

**实践验证**：您可以多次点击按钮，观察返回时间的变化，验证Lambda每次都被调用。也可以在Lambda代码中添加更多逻辑，如生成随机数、查询Aurora数据库（需要为Lambda配置VPC和权限）等来丰富功能。

**安全方面**：要注意保护API接口不被滥用。可考虑在API Gateway启用身份验证（如使用API Key、IAM授权或Cognito用户池），或者为Lambda添加WAF保护。由于我们的示例API是公开的且有免费额度，所以暂未实现鉴权，但生产环境应根据需要锁下来访权限。

**总结**：通过这个完整示例，我们已经验证架构中 EC2、API Gateway、Lambda 三者联动的可行性。在实际项目中，您可以把EC2换成托管前端（如S3+CloudFront静态网站）、API Gateway+Lambda作为后端处理业务逻辑，Aurora 提供数据库支撑，实现一个全栈应用，而且绝大部分时候都运行在按需计费模式下，成本随着使用量线性扩展，非常高效。

---

## 8. 部署 AWS WAF 防火墙 (Web Application Firewall)

网络应用难免面对各类攻击，比如SQL注入、XSS跨站脚本、恶意机器人等。AWS 的 **WAF (Web Application Firewall)** 可以在应用层提供防护。WAF 可以与 ALB、API Gateway 或 CloudFront 集成，在这些流量入口处根据自定义规则或托管规则过滤恶意请求。

在我们的架构中，WAF 可部署在 ALB 前面，保护进入 ALB 的HTTP请求。也可以分别给 API Gateway 设置一个 WAF Web ACL。本示例将演示创建一个 WAF Web ACL，并绑定到 ALB 实现基本防护。我们将使用 AWS 托管规则组来快速应用一组常见的防护规则。

**步骤1：创建 WAF Web ACL**

1. 进入 **WAF & AWS Shield** 控制台（或者在服务列表搜索 “WAF”）。
2. 点击 “Create web ACL” 创建新的 Web访问控制列表。
3. **基本信息**：
   - Web ACL 名称：如 “KongLing-WAF”.
   - 区域：选择 “区域性 (Regional)”。
   - 资源范围：选择 “Amazon CloudFront” 或 “区域性资源”。因为我们要保护 ALB (区域性资源)，选择 “Regional (ap-northeast-1)”。
   - 日志选项：此处可跳过或启用，演示不强求。
4. **关联资源**：
   - 下拉选择要保护的资源。选中我们的 ALB “KongLing-ALB”。（如果API Gateway也想保护，可一并选中，但前提API GW类型支持WAF。Regional REST API 可通过“Regional” WAF保护）。
5. **添加规则**：
   - 点击 “Add rules” -> 选择 “Add managed rules”。AWS 提供许多托管规则组，我们选择常用的：
     - 在 AWS Managed Rules 中，勾选 **AWSManagedRulesCommonRuleSet**（常见漏洞防护）和 **AWSManagedRulesAmazonIpReputationList**（AWS IP信誉名单，可挡掉恶意IP）。
     - 这些规则可以基本涵盖OWASP Top10攻击等。
   - 点击 Add rules 添加选中规则组到当前ACL。它们默认操作是 “Block” 可疑请求，可以保持默认。
   - 规则列表中可调整优先顺序，默认顺序即可。
6. **默认操作**：
   - 针对未被任何规则匹配的请求，选择 “Allow”（允许）。也就是只有触发规则的流量被阻断，其他正常流量通过。
7. **标签**：可选填，略过。
8. 点击 “创建 web ACL”。需要几秒完成，成功后页面会列出当前 ACL 及关联的 ALB 资源。

**步骤2：测试 WAF**

由于我们添加的是通用规则，如果我们的访问没有恶意特征，一般不会被拦截。可以尝试一些简单测试：
- 在浏览器通过 ALB 访问网站，应该仍然正常。
- 模拟恶意请求：例如在URL后加上类似 `?sql=SELECT * FROM users` 这样的查询参数，多次刷新。查看 WAF 控制台中 “监控”->“可视化”页面，或 ALB访问日志（如果启用）可以看是否有拦截记录。托管规则可能视此类模式为SQL注入而拦截（也可能认为无害放过，如果不精确）。
- 您也可以在 WAF ACL 中手动添加一条 IP 规则作为测试：如阻止自己电脑当前IP，然后再访问网站看是否被拦截返回403。

**WAF 生效验证**：
WAF 拦截请求时，ALB会返回403 Forbidden。可以在WAF的“规则”->“Web ACL 日志”中查看被阻止请求的计数。如果只是正常浏览，不会有block事件。

> 🔸 *费用提示*: WAF 按 WebACL 数量、规则条数、及请求数计费。每个 WebACL ~$5/月，每条自定义规则 ~$1/月，每处理100万请求 $0.6。托管规则组可能含多条规则，也会计费。对于小规模应用，这是额外成本。如果预算有限，可以暂不使用WAF，而通过安全组只允许可信IP、在应用层做好校验、或者使用 **CloudFront**（自带一些基础保护）等方式降低风险。另一个替代是使用第三方如 Cloudflare 免费计划提供的WAF基本规则。在需要更高级防护时再考虑启用AWS WAF。**免费额度**：WAF 没有免费套餐，试用需注意及时删除以免月末产生固定费用。

---

## 9. 部署 AWS CodePipeline 持续集成流水线

现代云架构少不了 CI/CD 来自动化代码部署。AWS 提供一系列开发者工具服务，如 CodeCommit（代码存储）、CodeBuild（代码构建）、CodeDeploy（部署）、以及 CodePipeline（CI/CD编排）。在架构图底部，我们看到了 CodePipeline、CodeCommit 等服务。

本节我们将创建一个简单的 **AWS CodePipeline** 流水线作为示例。鉴于我们当前的应用主要是 Lambda 和 ECS，我们示范将 Lambda 函数的代码通过 CodePipeline 自动部署：当开发者更新代码仓库时，让流水线检测到并更新 Lambda 函数。这个过程需要用到 CodeCommit 作为源，和 CodeBuild 来打包代码及调用 AWS CLI 部署 Lambda。

为简化，假设我们的 Lambda 函数代码非常简单，没有额外依赖，那么可以直接在 CodeBuild 中使用 AWS CLI 更新函数代码。如果函数依赖第三方库，则需要在 CodeBuild中安装依赖、打包成.zip并通过 Lambda API更新。

**前置准备**：创建 CodeCommit 仓库并上传 Lambda 代码。

1. 进入 **CodeCommit** 控制台，点击“创建仓库”。输入仓库名称 “KongLing-lambda-repo”，创建。
2. 使用 Git 将当前 Lambda 函数代码push到 CodeCommit：
   - 在本地创建一个文件夹，将 `lambda_function.py` 保存其中（内容和之前Lambda控制台里的一致）。初始化 git 并将 CodeCommit 仓库设置为远程，将文件推送。具体步骤可参考 CodeCommit 控制台提供的连接命令（可以使用 HTTPS，通过 AWS用户账户认证，或者使用 IAM用户HTTPS Git密码）。
   - 成功后，在 CodeCommit 控制台应能看到仓库里的文件（lambda_function.py 等）。

**步骤1：创建 CodeBuild 项目**

CodeBuild 将负责将代码构建并部署更新Lambda。我们可以直接在 Build 过程中调用 AWS CLI 更新 Lambda 函数代码。所以需要赋予 CodeBuild 权限去更新 Lambda。

1. 打开 **CodeBuild** 控制台，点击 “创建构建项目”。
2. **项目配置**：
   - 项目名称：如 “KongLing-lambda-build”.
   - 源提供程序：选择 “CodeCommit”，仓库选刚刚创建的 “KongLing-lambda-repo”，分支选 “main”。
   - 环境映像：选择 “托管映像”，操作系统“Ubuntu”，运行时“Standard”，映像版本“aws/codebuild/standard:5.0” 或最新。
   - 环境类型：Linux，构建规格：选择最小的 `arm1.small` 或 `linux.small` 以省资源。
   - **服务角色**：可以选择新建角色。CodeBuild 将创建/使用一个 IAM 角色，需附加权限允许它更新 Lambda。为了简便，这里创建后再手动给该角色添加 AWSLambdaFullAccess 权限（实验环境做法，生产应缩小权限范围）。
   - 构建规范：选择 “插入构建命令”。我们不另写buildspec文件，直接在界面提供简单命令。选择单一阶段，在“构建命令”窗口中填入：
     ```bash
     # 安装 AWS CLI v2 (CodeBuild映像一般带awscli v2，无需安装，如缺则安装)
     # pip install awscli -U
     # 打包代码 (对于纯python单文件无需zip，直接用本地文件更新)
     echo "Updating Lambda code..."
     aws lambda update-function-code --function-name KongLingFunction --zip-file fileb://function.zip
     ```
     *解释*:我们假定仓库里除了 `lambda_function.py` 可能还有其他依赖文件，要打包成zip。这里应先压缩：`zip -r function.zip .` 把当前目录代码打包，然后调用 AWS CLI 更新Lambda代码。`--function-name KongLingFunction` 要和实际Lambda名称匹配；`--zip-file fileb://function.zip` 指定zip文件。本例因为我们的函数无额外文件，也可以不zip直接使用 `--zip-file fileb://lambda_function.py` 但 Lambda 更新接口要求zip格式，所以我们还是打包空壳也行。
   - 将这些命令写入 CodeBuild的 Build 阶段。
3. 点击创建项目。然后去 IAM 控制台，找到新创建的 CodeBuild 角色(名称类似 codebuild-KongLing-lambda-build-service-role)，附加策略 **AWSLambda_FullAccess**（或自定义一个只允许更新特定函数的策略）。这确保 CodeBuild 有权限执行 `lambda update-function-code`。

**步骤2：创建 CodePipeline**

1. 打开 **CodePipeline** 控制台，点击 “创建流水线”。
2. **流水线设置**：
   - 名称：如 “KongLing-Pipeline”.
   - 新建服务角色或默认皆可。
   - 高级设置里，默认即可（也可以启用流水线触发器）。
3. **添加源阶段**：
   - 提供程序：选择 “CodeCommit”。
   - 仓库：选择刚创建的 “KongLing-lambda-repo”。
   - 分支：选择 “main”。
   - 检测更改：选择 “立即”（启动WebHook自动触发）。
4. **添加构建阶段**：
   - 构建提供商：选择 “AWS CodeBuild”。
   - 项目名称：选择 “KongLing-lambda-build”。
   - CodePipeline 会自动连接前面创建的 CodeBuild 项目。
5. **部署阶段**：
   - 这里我们其实在 CodeBuild 已经完成部署Lambda，无需额外部署动作。所以可以**跳过部署**阶段。直接点击 “跳过部署阶段”（或添加一个空部署）。
6. 点击 “创建流水线”。Pipeline 会自动启动执行一次（拿最新源码触发build）。

**验证 Pipeline**：
- 在 CodePipeline 控制台的 pipeline 页面，可以看到 Source 阶段成功，然后 Build 阶段执行。如果一切配置正常，Build 阶段会显示通过并绿色对勾。这说明 CodeBuild 脚本成功运行。可以点击 CodeBuild 日志查看详细输出，应该能看到 “Updating Lambda code...” 等日志以及 AWS CLI 调用结果。如果有报错（比如权限问题或命令问题），需要调整脚本或权限再重试。
- 如果Pipeline执行成功，那么Lambda 函数代码已被更新为 CodeCommit 仓库中的内容。可以在 Lambda 控制台打开函数，检查代码是否与仓库一致。
- 现在尝试修改一下仓库代码，比如更改返回消息文本，推送 commit。几秒钟后，Pipeline 会自动检测到变更，触发新一次流水线执行，完成后Lambda函数即应用了新代码。刷新前端调用，可以看到返回结果变化。这就实现了CI/CD。

**注意**：为了聚焦演示，我们跳过了 CodeDeploy 等组件。实际上，对于 ECS 部署或Lambda更复杂的发布（如流量控制），可以利用 CodeDeploy结合Pipeline实现蓝绿部署等高级功能。这里Pipeline简单直接地更新了Lambda代码。

> 🔹 *免费额度*: **CodePipeline** 对新老客户每月提供一个免费活动的Pipeline。额外Pipeline收费约 $1/月。**CodeBuild** 有每月 100分钟的免费构建时间（使用特定小型规格）供一年新用户使用，之后按分钟计费。我们的构建很短，费用几乎可忽略。**费用提示**：持续启用的Pipeline本身费用低廉，但关联的资源（如每次构建用的Compute时间）也会产生费用。建议在学习完后暂停或删除Pipeline和Build项目，以免持续占用。在小型项目上，如果不想付费使用Pipeline，也可以考虑使用GitHub Actions等外部CI服务结合AWS部署凭据实现自动部署。对于完全免费的选项，可以用 AWS CodeSuite 结合免费额度足够应对低频次部署需求。

---

## 10. CloudWatch 日志与监控

有了应用和基础设施后，持续的监控和日志收集至关重要。AWS **CloudWatch** 提供统一的日志、指标和告警功能。我们的架构中已经在悄然使用CloudWatch：例如，Lambda 自动将执行日志写入 CloudWatch Logs，ECS Fargate 任务如果设置了log driver（默认 awslogs）也会输出日志到 CloudWatch，EC2 实例的默认指标（CPU利用率、网络等）在 CloudWatch Metrics 中，ALB 的请求数和性能指标也在 CloudWatch汇总。这里我们介绍如何查看这些监控信息，并创建一个简单的告警。

**查看关键日志和指标**：

- **Lambda 日志**：进入 CloudWatch 控制台，左侧点击“日志 -> 日志组”。找到名字以 `/aws/lambda/KongLingFunction` 的日志组，点进去可以看到每次调用Lambda所产生的日志流和日志消息（包括我们调用时返回的内容log等）。这对于调试Lambda非常有用。
- **ECS 容器日志**：如果在任务定义中配置了 awslogs 日志驱动（默认示例任务好像已配置），在 CloudWatch Logs里也会有 `/aws/ecs/<任务定义名称>` 的日志组，包含容器输出日志。可以检查示例容器是否打印了访问日志。
- **EC2 系统日志**：默认未接管。如果需要，可安装 CloudWatch Agent 将系统日志发送到 CloudWatch。
- **指标 (Metrics)**：在 CloudWatch左侧点“指标”，可以查看各服务的指标。例如：
  - EC2 -> 按实例 -> 选中您的EC2实例，可查看CPUUtilization等曲线。
  - ECS -> ContainerInsights（若启用）或使用Service名查看任务数等。
  - ELB -> 选择ALB指标，可以看到RequestCount、TargetResponseTime等。
  - Lambda -> 按函数名查看调用次数、错误率、持续时间等指标。
  - RDS/Aurora -> 数据库的CPU、连接数等指标在 RDS 分类下可查看（需要启用增强监控才能有更细项）。
  
- **Dashboard**：CloudWatch支持创建仪表盘，将多个指标图表放一起。您可以尝试创建一个Dashboard，将EC2 CPU、ALB请求数、Lambda并发数等放在同一页面，直观监控系统健康。

**创建示例告警 (Alarm)**：

我们以 EC2 实例的 CPU 利用率为例设置一个警报，当CPU持续过高时提醒我们。
1. 在 CloudWatch 控制台，点击左侧“警报 -> 创建警报”。
2. 点击“选择指标”，浏览到 EC2 -> Per-Instance Metrics -> 找到您的实例ID -> 选择 **CPUUtilization** 指标。
3. 点击“选择指标”进入配置：
   - 统计方式：平均值
   - 周期：5 分钟（默认）
   - 条件：大于 > 80 （即CPU利用率 > 80%）
   - 持续时间：连续3个周期（15分钟）都违反即报警（可调整）。
4. 点击“下一步”，配置通知：
   - 选择一个现有的 SNS 主题或创建新主题，用于发送通知邮件或短信。如果没有SNS可跳过（但CloudWatch要求必须选一个动作，可以先创建SNS topic但不订阅，以示范）。
   - 动作：Alarm状态时，发送通知到SNS主题 “KongLingAlerts” (假设已创建并订阅邮箱)。
   - （您也可以设置当Alarm恢复正常时的通知）
5. 警报名称：如 “EC2-HighCPU”.
6. 点击完成创建。

这样，一个CloudWatch告警就创建好了。如果 EC2 CPU 连续15分钟超过80%，Alarm会变为“Alarm”状态并发通知。同时，该Alarm也会显示在 AWS控制台首页的警报列表中。您可以通过压测EC2或者在EC2中执行高CPU的任务来触发此告警测试。

**CloudWatch in Action**：
CloudWatch 还能配合 Auto Scaling 或 Lambda 实现自动化操作。例如，可以设置当指标超过阈值时自动执行某个 SNS通知或Lambda函数。这可以用于自动重启服务、扩容等。我们的示例未包括Auto Scaling，但您可自行探索EC2或ECS的Auto Scaling策略，这些策略通常基于CloudWatch指标触发。

> 🔹 *免费额度*: CloudWatch 提供**相当慷慨的免费指标和日志额度**。具体包括每月免费的指标数据、10个自定义指标、5GB日志存储和处理、3个Dashboard等。一般小规模项目不会产生明显CloudWatch费用。**费用提示**：注意长时间存储大体量日志会产生费用，建议为Log设置过期策略，如30天自动删除，避免日志无限增长收费。启用详细监控或自定义指标也会产生额外费用，必要时再用。总之，充分利用免费额度监控关键指标是很有必要的，但也应留意不必要的数据收集以防费用累积。

---

## 11. CloudFormation 模板示例 (基础网络 + EC2 + Lambda)

在以上步骤中，我们手动点击控制台完成了所有资源部署。在实际中，使用 **基础设施即代码 (Infrastructure as Code)** 工具可以更加高效、一致地创建和管理云资源。AWS 提供的 IaC 服务是 **CloudFormation**，通过模板（YAML/JSON）描述资源，一键部署整个栈。

本节我们提供一个简化的 CloudFormation 模板示例，用于创建**基础网络环境 + 一个EC2实例 + 一个Lambda函数**，相当于把之前的部分步骤用代码实现。您可以将此模板保存为 `KongLing-demo.yaml` 并通过 CloudFormation 控制台或 AWS CLI 部署，来验证自动化创建资源的过程。

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: KongLing 基础架构示例 - 包含一个VPC网络环境、一个EC2实例和一个Lambda函数
Parameters:
  KeyName:
    Description: 用于 EC2 SSH 的密钥对名称
    Type: String
    Default: my-keypair  # 替换为您自己的密钥名称
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      Tags:
        - Key: Name
          Value: KongLingVPC

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags: [{ Key: Name, Value: KongLingIGW }]
  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: ap-northeast-1a
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: KongLingPublicSubnet

  RouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags: [{ Key: Name, Value: KongLingPublicRouteTable }]
  PublicRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref RouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
  SubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet
      RouteTableId: !Ref RouteTable

  EC2SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: KongLing-EC2-SG
      GroupDescription: Allow SSH and HTTP
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0   # 实际应限制为管理IP
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
  LambdaSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: KongLing-Lambda-SG
      GroupDescription: Security group for Lambda (if needed for VPC access)
      VpcId: !Ref VPC
      SecurityGroupEgress:
        - IpProtocol: -1
          FromPort: 0
          ToPort: 0
          CidrIp: 0.0.0.0/0

  EC2Instance:
    Type: AWS::EC2::Instance
    Metadata:
      AWS::CloudFormation::Init:
        config:
          packages:
            yum:
              nginx: []   # 安装 Nginx
          services:
            sysvinit:
              nginx:
                enabled: true
                ensureRunning: true
    Properties:
      InstanceType: t2.micro
      ImageId: !Ref LatestAmiId   # 使用下方Mappings获取AMI
      KeyName: !Ref KeyName
      NetworkInterfaces:
        - DeviceIndex: 0
          SubnetId: !Ref PublicSubnet
          GroupSet: [ !Ref EC2SecurityGroup ]
          AssociatePublicIpAddress: true
      UserData: 
        Fn::Base64: !Sub |
          #!/bin/bash
          yum update -y
          yum install -y nginx
          service nginx start
          echo "<h2>EC2 Web Server Running</h2>" > /usr/share/nginx/html/index.html
  # 获取最新Amazon Linux2的AMI
  LatestAmiId:
    Type: "AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>"
    Default: "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"

  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: KongLing-lambda-exec-role
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal: { Service: lambda.amazonaws.com }
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

  HelloLambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: KongLingFunctionCFN
      Handler: index.handler
      Runtime: python3.9
      Role: !GetAtt LambdaExecutionRole.Arn
      Timeout: 10
      Code:
        ZipFile: |
          def handler(event, context):
              return {
                  "statusCode": 200,
                  "body": "Hello from Lambda (Deployed via CloudFormation)!"
              }
      VpcConfig:
        SecurityGroupIds: [ !Ref LambdaSecurityGroup ]
        SubnetIds: [ !Ref PublicSubnet ]
Outputs:
  EC2PublicIP:
    Description: EC2实例的公网IP地址
    Value: !GetAtt EC2Instance.PublicIp
  LambdaFunctionName:
    Description: 部署的Lambda函数名称
    Value: !Ref HelloLambdaFunction
```

**模板说明**：上述模板会创建一个VPC (10.0.0.0/16)、一个公有子网(10.0.1.0/24)、Internet网关和路由，让该子网具备互联网访问；两个安全组（EC2和Lambda用）；一个 t2.micro EC2实例（Amazon Linux2，附带UserData安装Nginx并输出一行网页）；以及一个示例Lambda函数（Python3.9版，代码内联返回Hello消息）。为了简单，Lambda被放在Public子网且没有实际访问内网资源，仅演示CFN如何创建Lambda。还包括Outputs输出EC2的IP和Lambda名称。

您可以通过 AWS CLI 部署该模板，例如：
```bash
aws cloudformation create-stack --stack-name KongLingDemoStack \
  --template-body file://KongLing-demo.yaml --parameters ParameterKey=KeyName,ParameterValue=<YourKey> \
  --capabilities CAPABILITY_IAM
```
等待几分钟，Stack 创建完毕后，CloudFormation 会显示成功。然后您可以在 EC2 控制台看到新建的实例、在Lambda控制台看到新函数。通过Outputs提供的 EC2PublicIP，访问其80端口，能看到 "EC2 Web Server Running" 字样；调用LambdaFunctionName指定的函数，可以获得返回消息。

这个模板仅搭建了基本资源。您可以在此基础上继续添加，例如增设RDS实例（可用 `AWS::RDS::DBInstance` 资源）、ECS集群和Task（较复杂，建议用CFN的 AWS::ECS::Service 等资源定义或借助更高级别的CDK/Serverless Framework）等。不管怎样，通过模板来管理基础设施可以避免手工错误，并且方便销毁/重建整个环境。**要删除Stack**，只需在控制台或CLI执行删除操作，CloudFormation会自动依赖次序删干净所有资源，非常方便（注意EC2上的手动修改不会反映在栈中，删除栈时可能保留或失败）。

---

## 12. 资源清理与一键关闭脚本

完成实践后，为避免持续费用，建议及时清理或关闭不再使用的资源。以下给出各资源的关闭或删除要点，以及一个简单的Shell脚本示例，可通过 AWS CLI 停止主要计费项。

**逐项资源清理指引**：

- **EC2 实例**：可在 EC2控制台对实例执行“停止”（停止后不计CPU费用，但EBS存储仍计费几毛钱/月）或“终止”（永久删除实例和磁盘）。如果只是短暂停用，选择停止实例即可。
- **ECS 服务**：在 ECS 控制台，集群下找到服务 “KongLing-Service”，选择更新 Desired Count 为 0（缩容服务，停止所有任务）。或者直接删除服务（Delete），会停止并注销任务。这样 Fargate 不会再收费。
- **ALB 负载均衡**：在 EC2 -> 负载均衡器 列表，选中 “KongLing-ALB”，执行删除。相应的目标组也可删除。ALB 删除后不再计费。
- **Aurora 数据库**：在 RDS 控制台，选择 Aurora 数据库集群。可以对集群执行 “停止”（Aurora支持最多7天停机，超时会自动重启）。停止期间不计实例费，仅存储费仍算。如果不再需要，最好 **快照备份后删除**：先删除 DB 集群（可选保留快照），然后删除快照避免存储费。注意删除Aurora集群时勾选删除自动备份和取消删除保护。
- **Lambda 函数**：Lambda 本身不产生费用（除非被频繁调用超出免费额），保留空闲无碍。但如不需要，可在 Lambda 控制台删除函数。
- **API Gateway**：在 API Gateway 控制台，找到 API并选择删除，可删除已部署的API（记得先删除自定义域名映射如有）。API删除则不再计费。
- **WAF WebACL**：在 WAF 控制台 dissociate (解除关联) 资源然后删除 WebACL。每个ACL每月$5，所以不用时应删掉。
- **CodePipeline/CodeBuild/CodeCommit**：这些服务不使用时费用极微（Pipeline $1/月，CodeCommit 5用户免费，CodeBuild按用量）。但如不需要，也可删除Pipeline，删除CodeBuild项目，和删除CodeCommit仓库，确保不再触发构建。
- **CloudWatch Logs**：大量日志可能积攒存储费用，可在 CloudWatch Logs 控制台删除不需要的日志组（如 /aws/lambda/… 等）。也可为日志组设置保留周期（例如30天）。
- **其他**：安全组、VPC 等空闲资源虽然不收费，但为了环境整洁，也可以在EC2和VPC控制台删除（先删子网、网关、路由，再删VPC）。如果用CFN创建的，一并删除Stack就好了。

**一键停止脚本示例**：

如果您已安装并配置 AWS CLI，以下是一段Linux Shell脚本，用于停止/删除主要资源。运行前请替换其中的占位资源ID/名称为您自己的。

```bash
#!/bin/bash
REGION="ap-northeast-1"

# 停止 EC2 实例
EC2_ID="i-0123456789abcdef0"
aws ec2 stop-instances --region $REGION --instance-ids $EC2_ID

# 停止 Aurora 数据库集群
DB_CLUSTER_ID="KongLing-aurora-cluster-id"
aws rds stop-db-cluster --region $REGION --db-cluster-identifier $DB_CLUSTER_ID

# 缩容 ECS 服务至0任务
CLUSTER_NAME="KongLing-Cluster"
SERVICE_NAME="KongLing-Service"
aws ecs update-service --region $REGION --cluster $CLUSTER_NAME --service $SERVICE_NAME --desired-count 0

# 删除 ALB (必须先删除Listeners和TargetGroups if any)
ALB_ARN=$(aws elbv2 describe-load-balancers --region $REGION --names "KongLing-ALB" --query "LoadBalancers[0].LoadBalancerArn" --output text)
aws elbv2 delete-load-balancer --region $REGION --load-balancer-arn $ALB_ARN
# 删除 Target Group
TG_ARN=$(aws elbv2 describe-target-groups --region $REGION --names "KongLing-TG" --query "TargetGroups[0].TargetGroupArn" --output text)
aws elbv2 delete-target-group --region $REGION --target-group-arn $TG_ARN

# 禁用 CodePipeline (通过删除来停止计费)
PIPELINE_NAME="KongLing-Pipeline"
aws codepipeline delete-pipeline --region $REGION --name $PIPELINE_NAME

# 提示其他资源人工检查
echo "请检查Lambda函数、API Gateway、WAF等并手动删除不需要的资源。"
```

> ⚠️ **注意**：脚本执行需非常小心。删除负载均衡等操作是立即的且不可逆。最好逐条执行确认无误。尤其删除RDS/集群操作，这里我们用的是stop暂停Aurora，未自动删除以免数据丢失。如果确定不再需要，应改为 `aws rds delete-db-cluster` 并加 `--skip-final-snapshot`（或先创建快照再删）。

运行上述脚本后，大部分计费资源都会停止。验证AWS控制台，确保实例变停止、Aurora变停止、ECS任务为0、ALB已消失、Pipeline已删。此时保留的费用主要是存储（EBS卷、Aurora存储、Logs等极少部分）。

**恢复使用**：如果只是暂停，为了下次实验，可以保留资源不删除，启动前再重新启动EC2实例（RDS可启动集群：`aws rds start-db-cluster`）、ECS服务改回原task数、重建ALB等。不过ALB删了恢复麻烦点，需要重建并更新前端配置。所以实际演练中可以考虑不断开ALB，只是停止后端，这样ALB几小时费用也有限但方便下次直接启动。

最后，不用的资源彻底删除是最佳节约手段。删除顺序建议先删依赖下游（如先删Pipeline、WAF，后删ALB，最后删VPC）。通过本节指南，您应该能安全地关停或移除所有资源。

---

## 13. AWS 学习路径推荐

通过本次实战，您已经初步掌握了在 AWS 上部署一个涵盖多种服务的架构。其中涉及到网络、计算、存储、安全、运维、DevOps等各个方面。AWS 服务众多，建议您按照以下功能模块分阶段深入学习，每一模块掌握核心服务后再拓展新的领域：

- **基础与网络模块**：先理解 **IAM (身份与访问管理)** 用于权限控制，这是安全的基础，然后深入 **VPC (虚拟私有云)** 网络，包括子网划分、路由表、Internet/NAT 网关、**安全组** 与 **网络ACL** 等。掌握这些后，您可以构建隔离的网络环境，为其它服务打下基础。相关服务：VPC、Subnet、Route Table、Internet Gateway、NAT Gateway、Security Group、IAM、AWS Organizations 等。

- **计算模块**：学习 **EC2** 的各种使用，包括选择合适的实例类型、EBS存储、AMI 镜像制作、Auto Scaling伸缩组、弹性负载均衡 (ALB/ELB/NLB) 等。同时了解容器相关的 **ECS** 和 **ECR**（容器镜像仓库），以及容器编排的替代方案 **EKS**（托管Kubernetes）。对比无服务器计算 **Lambda** 的使用场景和限制。掌握这些之后，您可以根据工作负载选择合适的计算形式。相关服务：EC2、Auto Scaling、ELB(ALB/NLB)、ECS/Fargate、ECR、EKS、Lambda、Elastic Beanstalk（打包PaaS部署）、AWS Batch 等。

- **数据库与存储模块**：学习 **RDS** 托管数据库（MySQL/PostgreSQL/Oracle/SQL Server等）和 **Aurora** 的高级特性，以及缓存数据库 **ElastiCache** (Redis/Memcached)。了解 NoSQL 数据库 **DynamoDB** 的用法和优缺点。掌握 **S3** 对象存储的用法、生命周期管理和权限控制，以及网络文件系统 **EFS** 和数据传输服务 **Storage Gateway** 等。根据应用需求选择合适的数据存储方案。相关服务：RDS、Aurora、DynamoDB、ElastiCache、S3、EFS、FSx、Storage Gateway、Amazon Redshift (数据仓库)、Amazon Athena 等。

- **安全模块**：深入学习 **IAM** 策略写法，掌握**多账号环境**下的权限管理。了解 **KMS (密钥管理服务)** 用于加密数据，**AWS Config** 和 **CloudTrail** 用于审计跟踪。学习 **WAF** 和 **Shield** 防护DDoS，**GuardDuty** 威胁检测，以及 **Inspector** 漏洞扫描等安全服务。安全是贯穿始终的任务，需要不断实践和审计。

- **监控与可观测性模块**：强化 **CloudWatch** 的使用，包括自定义指标、Dashboards、Alarms，以及 **CloudWatch Logs Insights** 日志分析。学习 **CloudTrail** 记录API调用用于安全分析。了解 X-Ray 分布式跟踪系统用于分析微服务性能瓶颈。掌握这些工具，才能对系统运行状况了如指掌。相关服务：CloudWatch、CloudTrail、X-Ray、AWS CloudShell（运维shell）、AWS Managed Services（系统管理）等。

- **DevOps 与自动化模块**：熟练使用 **CloudFormation** 模板编写和管理，更大型的可以学习 **AWS CDK**（代码定义基础设施）。掌握 **CodeCommit**/**CodeBuild**/**CodeDeploy**/**CodePipeline** 的CI/CD流程，或使用第三方CI如 GitHub Actions 集成 AWS 部署。了解 **AWS SAM** 或 **Serverless Framework** 快速部署无服务器应用。实现基础设施和应用部署的自动化，提高效率和一致性。相关服务：CloudFormation、CDK、CodeCommit、CodeBuild, CodeDeploy, CodePipeline、Elastic Beanstalk（CI/CD一体化服务）、OpsWorks等。

- **高级主题**：根据兴趣和业务需求，可以进一步学习 **大数据与分析**（EMR, Glue, Kinesis, Athena, Redshift 等）、**人工智能与机器学习**（SageMaker，Rekognition等）、**物联网IoT**、**边缘计算**（CloudFront, AWS Wavelength）等领域的服务。同时别忘了关注 **成本管理**（AWS Budgets, Cost Explorer）来优化你的架构成本。

学习AWS是一个持续过程。建议利用官方文档、AWS Training课程和实践项目相结合的方式。每学习一个服务，尽量亲手在**免费额度**内做小型实验，加深理解。通过本次KongLing架构实战，你已经体验了将多个服务拼合成解决方案的过程。接下来，可以尝试从零开始自己设计一个架构并实现，比如一个无服务器投票应用（S3托管前端+API Gateway+Lambda+DynamoDB），或一个LAMP博客系统（EC2+RDS+ElastiCache+CloudFront）等。在实践中融会贯通各模块的知识。

最后，记得时刻关注AWS的新服务和新特性（AWS官方博客、发布会 re:Invent 等），云计算技术发展迅速，不断学习才能保持您的架构在安全、性能和成本上的最佳实践状态。祝您在 AWS 云上探索愉快，收获满满！

