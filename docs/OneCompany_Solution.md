# 一人AI自我进化公司：完整部署方案（v2.1）

**版本**：v2.1
**日期**：2026-10-04
**更新要点**：基于v2.0补全全部12项缺漏项；新增独立合规Agent；移除高风险RentAHuman.ai组件；增加产品矩阵、定价、获客、备份灾备、性能基准、止损退出机制；修正代理单点故障、Agent对外签约风控；微调分阶段实施路径
**Token来源**：DeepSeek API
**公司架构**：参考 VoxYZ 6角色闭环 + 独立合规Agent（只读审计角色）
**财务架构**：境外主体 + USDT/USDC非托管结算
**市场范围**：全球（通过代理层访问）

> ⚠️**重要前置提示**：虚拟货币相关业务活动在我国境内属于非法金融活动。本文USDT/USDC收款仅为**境外主体在海外司法辖区开展业务**的技术方案推演；若本人位于境内，不可以直接参与、运营、收款，必须由境外注册实体独立承担全部法律与税务责任。

## 一、方案总览

本方案构建一套 **7×24小时自主运转、全球市场探嗅、自我进化、稳定币结算** 的AI一人公司。相比v1.0，核心变化是增加了**代理层（全球出站通道）** 和**全球市场信息探嗅矩阵（三类信源）**；v2.1补齐产品、定价、灾备、税务、止损等缺失模块，强化风控与审计能力。

| 层级 | 核心组件 | 职责 |
|:---|:---|:---|
| **底座层** | DeepSeek Harness + DeepSeek API | Agent运行时 + LLM推理 |
| **代理层** | dsh-proxy / dsh-websearch-direct / dsh-egress-router | 全球出站通道 + 按需路由 + 多代理池故障切换 |
| **信息探嗅层** | 网站/众包/开源对标三类信源 | 全球市场信号采集 |
| **组织层** | 6角色Agent团队（参考VoxYZ）+ 独立合规Agent | 决策、研究、内容、发布、质检、审计合规 |
| **执行层** | 开发/浏览器/多媒体工具链 | 交付执行 |
| **财务层** | 境外主体 + USDT/USDC非托管结算 | 自动结算到钱包，交易日志留存 |
| **控制层** | Token预算 + 熔断机制 | 成本管控 + 生存边界 |
| **交互层** | DSH Web UI + 外部消息推送 | 查看信息 + 接收进度 + 人工干预 |

**核心生存逻辑**：Token预算 = 工作量标尺。系统消耗DeepSeek Token执行任务，通过交付产品赚取USDT/USDC，收入回流覆盖Token成本后进入盈利状态。

## 二、Token来源：DeepSeek API配置

### 2.1 定价与预算规划

DeepSeek API采用**峰谷双倍差价**机制：

| 计费维度 | deepseek-flash（空闲/高峰） | deepseek-v4-pro（空闲/高峰） |
|:---|:---|:---|
| 输入（缓存命中） | ¥0.02 / ¥0.04 每百万token | ¥0.15 / ¥0.30 每百万token |
| 输入（缓存未命中） | ¥1 / ¥2 每百万token | ¥4.5 / ¥9 每百万token |
| 输出 | ¥4 / ¥8 每百万token | ¥13.5 / ¥27 每百万token |

**空闲时段**：北京时间周一至周五 9:00-12:00、14:00-18:00 以外所有时间（含周末及法定节假日全天），价格为高峰时段的一半。

**每日10元预算的可行性**：假设日均消耗100万输入token（90%缓存命中）+ 20万输出token：
- 空闲时段成本 ≈ **¥0.92/天**
- 高峰时段成本 ≈ **¥1.84/天**

**结论**：10元/天预算对于日常运维绰绰有余，但全球信息采集的额外成本（代理流量、API调用）需要单独计入预算。

### 2.2 充值方式

DeepSeek采用**预充值模式**，支持支付宝/微信在线充值，充值余额**永久有效不过期**。企业用户支持对公汇款。

### 2.3 API Key配置

```
DEEPSEEK_API_KEY=sk-your-key-here
DEEPSEEK_BASE_URL=https://api.deepseek.com
DEEPSEEK_MODEL=deepseek-flash
```

## 三、代理层：全球出站通道配置

代理层是整个方案中"全球市场探嗅"的技术前提。DSH默认走本机网络出站，国内宽带段访问X/Twitter、部分海外API时会被拦截。代理层解决两个问题：**模型请求走代理**（DeepSeek API在国内可直连，但部分海外模型需要代理）和**web搜索/抓取走代理**（访问全球信息源）。

> v2.1更新：引入多代理池、节点健康检测，自动剔除失效节点，解决单代理IP单点故障。

### 3.1 三个代理插件选型

| 插件 | 包名 | 核心能力 | 适用场景 |
|:---|:---|:---|:---|
| **dsh-proxy** | `copylee711/dsh-proxy` | 配置全局代理，模型请求、web搜索、web fetch、HTTP MCP全部走代理；支持Per-provider代理 | 基础代理需求 |
| **dsh-websearch-direct** | `MYCF711/dsh-websearch-direct` | 引擎=逻辑源、入口=直连/镜像/加速路由自动回退，**不消耗任何模型token**；支持全局代理和按入口开关代理 | 精细路由控制 |
| **dsh-egress-router** | `gswenxue/dsh-egress-router` | 三种模式（关闭/智能/全局），**选择性走节点**，直连失败自动回退，失效只标记不删除；支持多代理池轮换 | 智能路由，成本最优 |

**推荐组合**：以**dsh-egress-router**为主（智能模式，多代理池），配合**dsh-websearch-direct**的按入口开关代理能力，实现"DeepSeek API直连 + X/Twitter走代理 + 国内信息源直连"的精细路由。

### 3.2 路由策略

| 流量类型 | 路由方式 | 原因 |
|:---|:---|:---|
| DeepSeek API | 直连 | 国内可直接访问，走代理增加延迟 |
| X/Twitter、Reddit、Google | 走代理池 | 国内直接访问受限，多节点轮换防封禁 |
| 国内信息源（小红书、百度） | 直连 | 国内可达，代理反而降低速度 |
| 海外API（Exa、Tavily、Firecrawl） | 按需走代理 | 部分API国内可达，部分需要代理 |

安装命令：

```bash
# 按需出海路由（推荐主用）
npx -y @deepseek-ai/dsh plugin --profile web add gswenxue/dsh-egress-router
# 免API Key搜索+按入口开关代理
npx -y @deepseek-ai/dsh plugin --profile web add MYCF711/dsh-websearch-direct
```

## 四、全球市场信息探嗅矩阵

这是v2.0方案的核心新增模块。AI公司通过三类信源进行全球市场探嗅：**网站信息源、众包平台、开源项目对标**。

> v2.1更新：移除RentAHuman.ai组件，物理世界任务不纳入系统能力；真人外包需求统一走Upwork原生MCP。

### 4.1 第一类：网站信息源

| 工具 | 覆盖范围 | 核心能力 | 接入方式 |
|:---|:---|:---|:---|
| **dsh-web-tools** | 8+ Provider | 将Exa、Tavily、Firecrawl、Parallel、Brave、You.com、Jina、SearXNG、小红书、Twitter/X接入DSH标准`web_search`/`web_fetch`接口，支持多API Key轮询、Provider自动回退 | 插件安装 |
| **Trend Pulse** | 37个来源 | 实时趋势聚合，支持按国家/地区/城市查看兴趣度，相关查询和主题探索，29工具MCP Server | MCP接入 |
| **SAC_search** | 约200个搜索引擎 | 免API Key元搜索，并发执行、结果去重聚合、按引擎权重与时效性评分 | 插件安装 |
| **modsearch** | Firecrawl默认 | 多引擎（Firecrawl/Tavily/Exa/Grok X），自动故障转移，Firecrawl默认免Key | 插件安装 |

**dsh-web-tools**是核心采集工具，它将Exa、Tavily、Firecrawl等8个Provider接入DSH标准接口，支持**Provider自动回退**——一个API Key额度耗尽时自动切换下一个。

**Trend Pulse**覆盖37个来源，支持按地理市场查看趋势，是Scout Agent进行全球热点监控的首选工具。

### 4.2 第二类：众包平台

| 平台 | 核心能力 | AI接入方式 |
|:---|:---|:---|
| **Upwork MCP** | 搜索人才、发布工作、管理合同、提案管理、收益监控 | 官方MCP Server，支持Claude/ChatGPT/Cursor等AI工具 |

> v2.1变更：删除RentAHuman.ai、Gigbuz。物理世界线下任务，如需真人交付，统一通过Upwork MCP雇佣自由职业者，依托Upwork原生资金托管，规避平台风险。

### 4.3 第三类：开源项目对标

| 工具 | 覆盖范围 | 核心能力 |
|:---|:---|:---|
| **OSSInsight** | 100亿+ GitHub事件 | 分析任意仓库和开发者，提供星标、提交、PR、Issue、贡献者等指标，支持自然语言查询生成SQL |
| **Repo Scout** | GitHub/GitLab | 监控星标增长速度，在项目成为主流前发现"突破性"开源项目 |
| **OpenGithubs** | 中文社区 | 开源项目发现平台，提供日报、周报、月报 |

**OSSInsight**是Sage Agent进行**技术趋势分析**的核心工具。它分析超过100亿GitHub事件，可以通过自然语言查询生成SQL并可视化结果。

### 4.4 信源矩阵与Agent角色的映射

| Agent角色 | 主要信源 | 产出 |
|:---|:---|:---|
| **Scout（增长主管）** | Trend Pulse + dsh-web-tools | 全球热点信号、市场情绪 |
| **Sage（研究主管）** | OSSInsight + dsh-industry-research | 技术趋势报告、竞争分析 |
| **Minion（CEO幕僚长）** | Upwork MCP | 任务分派、接单决策 |
| **Quill（创意总监）** | dsh-web-tools（内容抓取） | 内容素材、文案 |
| **Xalt（社媒总监）** | Trend Pulse + X/Twitter搜索 | 发布内容、互动 |
| **Observer（观察员）** | 全信源汇总 | 质量检查、历程记录 |
| **合规Agent（新增，只读审计）** | 全系统日志、订单、提案、平台规则 | 风险审计、单据归集、预警提醒 |

## 五、公司架构：VoxYZ 6角色模式复刻 + 独立合规Agent

### 5.1 角色分工

| 角色 | 职责 | 对应DSH实现 |
|:---|:---|:---|
| **Minion（CEO幕僚长）** | 决策协调、任务分派、圆桌讨论；**>100美元项目提案强制提交人工审批** | 主Agent + dsh-onecompany调度 |
| **Sage（研究主管）** | 战略分析、深度研究、制定策略 | 调研Agent + OSSInsight + dsh-industry-research |
| **Scout（增长主管）** | 市场情报、客户发掘、信号追踪 | 探嗅Agent + Trend Pulse + modsearch |
| **Quill（创意总监）** | 文案撰写、内容设计、叙事构思 | 内容Agent + dsh-browser |
| **Xalt（社交媒体总监）** | 发布内容、用户互动、社媒管理 | 发布Agent + dsh-orchestrate |
| **Observer（公司观察员）** | 质量检查、全局观察、历程记录 | 质检Agent + dsh-agent-teams |
| **合规Agent（新增）** | 审计日志、单据归集、条款风险扫描、平台风控预警、行政事项到期提醒；**禁止对外发起交易、签约** | 独立只读Agent，对接dsh-cost-ledger |

### 5.2 VoxYZ的核心闭环机制

```
Agent提出想法 → 自动审批（小额）/人工审批（大额） → 创建任务+步骤 → Worker认领执行
→ 发出事件 → 触发新反应 → 回到第一步
```

### 5.3 三个关键修复

**坑一：任务打架（竞态条件）** → **砍掉一个执行者**，VPS做唯一执行者，控制平面只做轻量评估。
**坑二：触发器绕过审批流** → **提取共享函数**，所有路径必须调用统一入口。
**坑三：配额满了但队列还在涨** → **Cap Gates——在入口处拒绝**，配额满了提案直接被拒。

## 六、产品矩阵、定价策略、获客渠道（v2.1新增）

### 6.1 产品矩阵（依托机器视觉+工业自动化背景）

> 上线顺序：入门款优先，快速跑通闭环；进阶款次之；高端咨询强制人工复核。

1. **入门款（全自动交付，无需人工）**
   - 产品：开源项目竞品调研报告、技术趋势周报、GitHub仓库指标诊断、市场关键词挖掘
   - 交付物：结构化Markdown报告 + 可视化图表（Sage+Scout自动产出）
   - 目标客户：海外独立开发者、初创技术团队

2. **进阶款（半自动化，Observer质检）**
   - 产品：小型Agent脚本开发、MCP工具封装、网页抓取自动化、轻量数据Pipeline
   - 交付物：可运行代码 + 部署文档
   - 目标客户：海外小团队、独立创业者

3. **高端款（强制人工审批复核，AI仅做前期调研）**
   - 产品：工业视觉小项目需求拆解、C#/OpenCvSharp原型开发、自动化流程方案设计
   - 交付物：需求文档 + 代码原型 + 方案PPT
   - 目标客户：小型自动化公司（高溢价长尾）

### 6.2 定价策略（美元计价，Upwork/独立站）

| 产品 | 定价模式 | 价格 | 备注 |
|:---|:---|:---|:---|
| 开源竞品调研周报 | 订阅制 | $29/月 | 自动每周交付，可自助购买 |
| 一次性技术趋势报告 | 一次性 | $49/份 | AI全自动交付 |
| MCP工具/爬虫脚本开发 | 项目报价 | $99–249/项目 | AI初稿，Observer质检，复杂项人工复核 |
| 工业自动化方案咨询 | 项目报价 | $399–999/项目 | **强制人工审批，AI仅调研整理，不允许AI独立承诺交付** |

> 优惠策略：首单8折；包年订阅9折。
> 成本锚点：单份报告Token成本控制在$0.3以内。

### 6.3 获客渠道细化（Upwork之外）

1. GitHub：发布轻量开源MCP工具，README内嵌产品站点引流（面向开发者客户）
2. X/Twitter：Xalt Agent每日发布行业技术洞察、开源项目分析，引流到独立产品站点
3. Indie Hackers / Hacker News：定期发布自动化AI案例，内容引流
4. 独立站SEO：静态站点，自动发布由Scout+Quill生成的技术博客
5. 冷启动策略：前30单低价交付，积累Upwork评价，之后恢复标准定价

## 七、财务架构：全球运营主体 + USDT/USDC非托管结算

### 7.1 前置条件：境外运营主体

**这是整个财务架构的前提条件。** 2026年2月，中国人民银行等八部门发布的银发〔2026〕42号通知明确：**境外单位和个人不得以任何形式非法向境内主体提供虚拟货币相关服务**。如果运营主体、服务器、银行账户、税务身份都在境内，即使用代理访问全球信息，用USDT收款仍然是境内监管明确禁止的行为。

**合规路径**：注册境外运营主体（推荐美国怀俄明州LLC，远程注册友好），以该主体身份接入非托管支付平台、开设境外银行账户、履行税务申报义务。

**实操流程**：海外电话号码 → 美国LLC注册 → EIN税号 → 美国商业银行账户 → Stripe商业账户 → Wise多币种账户 → USDT出入金 → 公司年审和税务维护。大部分步骤可以远程完成。

> 一次性注册成本：$150–300；年度维护成本：$100–200（年审+注册代理地址费）。备选：新加坡/香港实体，成本更高，适合后期营收增长后切换。

### 7.2 非托管支付平台选型

| 平台 | 支持网络 | 费用 | 结算方式 | 接入复杂度 |
|:---|:---|:---|:---|:---|
| **Tavarov Pay** | BNB Chain、Ethereum、Base、Solana | 1% | 同交易直达钱包（99%到账） | 低（WooCommerce插件） |
| **AllScale Checkout** | 多链 | 0.6%（最低$0.10） | 即时到账USDT | 低 |
| **Morph Payments** | 多链 | 非托管 | 资金直接链上转入自托管钱包 | 中 |
| **PengoPay** | 多链 | 非托管 | USDT/USDC多链结算 | 中 |

**推荐方案：Tavarov Pay**。每笔支付是一笔交易，99%直达你的钱包，平台不持有资金，无提现等待期。接入流程：用收款钱包签名登录 → 创建API Key（从测试key开始）→ 配置webhook → 填入收款钱包地址。

### 7.3 与DSH的集成方式

**方案A：自建轻量收款API** — 用Node.js/Python搭建HTTP服务，调用Tavarov Pay的API创建支付链接，DSH商务Agent通过HTTP请求触发。
**方案B：WooCommerce桥接** — 部署轻量WooCommerce站点作为产品交付页面，安装Tavarov Pay插件，DSH Agent通过API创建订单。

**关键约束**：Morph Payments和PengoPay这类非托管平台解决的是"技术通道"问题，不解决"运营主体合法性"问题。合规前提是境外运营实体。

### 7.4 USDT/USDC收入的法币申报路径（v2.1新增）

1. 所有链上收款记录由系统自动保存：订单号、客户信息、交易哈希、金额、时间戳（dsh-cost-ledger扩展存储链上交易日志）
2. 境外LLC会计将稳定币收入按**收款当日法币汇率**换算为美元记账
3. 按美国州税法+联邦税法，每年申报企业所得税；稳定币属于企业资产，按资产规则计税
4. 提现：稳定币按需兑换为法币，进入Wise商业账户，留存全部兑换凭证

> 关键：所有记账、凭证留存，**交给境外持证会计师处理**，不依靠AI独立完成税务申报。合规Agent仅负责收集、整理单据，提交会计师。

## 八、UI界面与交互方式

### 8.1 查看公司信息：六个核心视图

安装`dsh-onecompany`插件后，侧边栏底部出现"一人公司"入口：

| 页面 | 内容 |
|:---|:---|
| **总览页** | 公司整体运行状态、待办事项、今日动态摘要 |
| **组织页** | 所有AI员工名册（姓名、职位、汇报对象、模型配置、Token预算） |
| **任务页** | 完整任务看板（状态、认领人、进度评论） |
| **审批页** | 等待决策的事项集中展示，顶部有未读角标 |
| **资料库页** | 员工读写的工作文档，按项目ACL权限管理 |
| **动态页** | 工作日志与运行时事件，按员工和日期聚合Token消耗 |

### 8.2 查看具体进度：子代理实时监控

安装`@leetoners/dsh-ui-subagent-monitor`插件，屏幕右上角常驻卡片式面板，实时展示每个子代理运行状态（🔵运行中 / 🟢完成 / 🔴失败 / 🟠已打断），面板顶部环形图展示上下文窗口占用率和缓存命中率。

### 8.3 与AI公司交互：四种方式

**方式一：Web UI直接对话** — 打开任意员工会话，直接插话或下达新指令。
**方式二：命令行指令** — `/company status`、`/company tasks`、`/company assign`、`/company approve/reject`。
**方式三：外部消息推送** — 安装`dsh-notify-plugin`或`multi-channel-notify`，支持个人微信ClawBot、企业微信、Telegram、Server酱、自定义Webhook。可配置事件类型：对话完成、对话失败、等待审批、需要授权、TODO进度推进。
**方式四：审批中心直接操作** — 员工提交越权申请时，审批页出现待办项，点击通过或驳回，结果以信箱消息自动回传。

## 九、备份灾备、版本升级策略（v2.1新增）

### 9.1 三层备份与灾难恢复

1. **应用层**：PM2容器状态自动保存；DSH配置文件、Agent角色定义、YAML预算配置，每小时自动提交私有Git仓库。
2. **数据层**：任务记录、客户资料、交付产物、Token消耗日志、链上交易记录，每日凌晨自动备份到S3兼容对象存储，保留30天快照。
3. **灾备层**：VPS故障时，一键脚本重建DSH环境，拉取备份配置、插件、资料库。

> 熔断保护：备份连续失败，触发系统暂停新任务，推送告警，防止数据丢失。

### 9.2 DSH & 插件版本升级策略

- 主版本升级（DSH v0.1.x → v0.2.x）：不自动升级；人工在测试环境完整验证所有插件兼容性，验证通过后再升级生产环境。
- 小版本补丁升级：每周自动在测试环境更新，无报错后，低峰时段更新生产。
- 插件：锁定插件版本号，禁止使用`latest`标签；每次升级前快照备份。

> 禁止生产环境自动升级，DSH当前是开发者预览版，破坏性变更风险高。

## 十、性能基准指标（v2.1新增）

Observer Agent持续采集，写入仪表盘：

| 指标 | 目标基线 | 告警阈值 |
|:---|:---|:---|
| 单份市场报告Token消耗 | <120k token | >200k token触发告警，终止任务 |
| 任务完成成功率 | >90% | <70%自动冻结同类任务 |
| 每美元Token成本产出收入 | > $5收入/$1 Token | <$2收入/$1 Token触发业务策略复盘 |
| 代理采集失败率 | <5% | >15%自动切换代理池节点 |
| 缓存命中率 | >85% | <60%优化prompt与缓存策略 |

## 十一、退出与止损机制（v2.1新增）

两类止损：**业务止损** + **系统紧急关停**

1. **业务止损**：连续30天，收入无法覆盖Token+代理+境外实体维护成本 → 系统自动暂停新订单，仅处理已下单交付任务，停止对外获客。
2. **紧急关停触发条件（任一满足）**
   - 单日Token消耗超过预算上限；
   - 连续5次支付回调异常；
   - 代理IP大规模封禁，信源采集失败率>30%；
   - Upwork账号风控告警。
3. **关停动作**：停止新增任务、暂停社媒发布、暂停接单，保留完整数据备份，人工介入评估重启/终止。

## 十二、初始种子预算与代理流量成本估算（v2.1新增）

### 12.1 初始种子总预算：$300

- $100：DeepSeek API预充值
- $120：代理流量季费
- $80：境外LLC注册前置费用

> 资金分批充值至API账户，配合预算熔断机制，控制最大损失。

### 12.2 代理流量成本估算

采用住宅代理做网页抓取/搜索；模型API流量不走代理。

- 单价：$8–12 / GB
- 初期预估月用量：3–6GB
- 月度代理成本：**$24 ~ $72**

> 优化：dsh-egress-router智能路由，国内站点直连，仅海外站点流量走代理池，降低流量消耗。

## 十三、完整部署命令速查

### 13.1 环境准备

```bash
# ═══ Node.js环境 ═══
nvm install 24 && nvm use 24
npm install -g pnpm@11.7.0
# ═══ DeepSeek Harness安装 ═══
npm install -g @deepseek-ai/dsh@latest
# ═══ 创建独立Profile ═══
dsh profile create ai-company
dsh profile use ai-company
```

### 13.2 核心插件安装

```bash
# ═══ 基础开发插件 ═══
npx -y @deepseek-ai/dsh plugin --profile web add @dsh-market/plugin@0.2.1
npx -y @deepseek-ai/dsh plugin --profile web add dsh-browser@0.1.0
npx -y @deepseek-ai/dsh plugin --profile web add dsh-memory@0.1.0
npx -y @deepseek-ai/dsh plugin --profile web add dsh-at-file@0.6.3
# ═══ Token预算控制 ═══
npx -y @deepseek-ai/dsh plugin --profile web add dsh-agent-budget
npx -y @deepseek-ai/dsh plugin --profile web add dsh-fuse
npx -y @deepseek-ai/dsh plugin --profile web add dsh-cost-meter
npx -y @deepseek-ai/dsh plugin --profile web add dsh-cost-ledger
# ═══ 团队编排 ═══
npx -y @deepseek-ai/dsh plugin --profile web add @nanmicoder/dsh-agent-teams@latest
# ═══ 公司管理UI ═══
dsh plugin --profile web add "github:xiazhi88/dsh-onecompany"
npx -y @deepseek-ai/dsh plugin --profile web add @leetoners/dsh-ui-subagent-monitor
npx -y @deepseek-ai/dsh plugin --profile web add dsh-notify-plugin
```

### 13.3 代理层与全球信息源

```bash
# ═══ 代理层 ═══
npx -y @deepseek-ai/dsh plugin --profile web add gswenxue/dsh-egress-router
npx -y @deepseek-ai/dsh plugin --profile web add MYCF711/dsh-websearch-direct
# ═══ 全球网站信息源 ═══
npx -y @deepseek-ai/dsh plugin --profile web add A3Boy/dsh-web-tools
npx -y @deepseek-ai/dsh plugin --profile web add modsearch
# ═══ MCP工具接入（Trend Pulse / Upwork MCP / OSSInsight） ═══
# Trend Pulse: pip install mcp-trendpulse
# Upwork MCP: pip install upwork-mcp
# OSSInsight MCP: @iflow-mcp/mcp-srv-ossinsight
```

### 13.4 7×24小时常驻配置

```bash
npm install -g pm2
pm2 start "npx @deepseek-ai/dsh web" --name ai-company
pm2 save
pm2 startup
crontab -e
# 添加: */5 * * * * cd /path/to/dsh && pm2 restart ai-company --update-env
```

### 13.5 Token预算配置

```yaml
budgets:
  - limitUsd: 10        # 每日预算
    window: day
  - limitUsd: 300       # 月度预算
    window: month
policies:
  maxReasoningEffort: medium
  allowedModels:
    - deepseek/deepseek-flash
  schedule:
    preferOffPeak: true
```

## 十四、实施路径（分阶段，v2.1微调）

| 阶段 | 时间 | 目标 | 关键动作 |
|:---|:---|:---|:---|
| **代理验证期** | 第1周 | 打通全球出站通道 | 安装代理插件，配置多代理池；验证X/Twitter、Google、Upwork可达 |
| **信源验证期** | 第2周 | 验证三类信源采集链路 | 安装dsh-web-tools、Trend Pulse、OSSInsight，采集一批真实信号，测试生成市场报告 |
| **最小闭环期** | 第3-4周 | 跑通"探嗅→验证→交付" | 2-3个Agent完成一个小型任务；接入Upwork MCP测试接单链路；上线入门款产品 |
| **财务接入期** | 第2月 | 接入非托管支付 | 注册境外LLC；配置Tavarov Pay测试模式；完成首笔小额USDT收款测试 |
| **稳定运转期** | 第2-3月 | 自动化运转 | 启用6角色Agent + 合规Agent；配置crontab心跳；持续监控Token消耗与收入比率；>100美元项目强制人工审批 |
| **进化期** | 第4月+ | 自我优化 | 分析历史数据，淘汰低效模式，固化成功工作流；拓展高端工业咨询项目 |

## 十五、关键风险与边界

**1. DSH仍为开发者预览版本**：会话损坏和插件崩溃是已知问题。建议采用"有人值守的半自动模式"作为过渡，每次升级前备份。

**2. 代理层是通道，不是合规解决方案**：代理让你能访问全球信息，但不改变运营主体的法律身份。USDT/USDC收款的前提是境外运营主体。

**3. 境外运营主体的持续维护成本**：美国LLC的年审、EIN申报、Wise账户税务信息、USDT出入金记录，都是每月或每季度需要处理的行政事务。由合规Agent跟踪提醒，交由境外会计师处理。

**4. 全球信息采集的成本需要纳入Token预算**：Apollo MCP的x402按次计费（web_scrape $0.02、x_search $0.75）、住宅代理流量成本、API Key调用成本，都应被纳入统一的工作量标尺。建议将"总运营预算"定义为Token + 采集成本 + 代理流量。

**5. AI Agent的行为边界在全球框架下需要更严格定义**：当Agent通过代理访问全球市场、以境外主体身份接单和收款时，它对外发送的邮件、提交的投标、签署的条款都代表真实商业实体。>100美元项目提案强制人工审批。

**6. Upwork MCP的账号风险**：使用浏览器自动化方式接入Upwork（lead-upwork方案）可能触发平台风控，建议优先使用官方MCP Server。

## 十六、待确认项（v2.1更新）

| 序号 | 确认项 | 选项 |
|:---|:---|:---|
| 1 | 是否已设立境外运营主体 | 是 / 否 / 计划中 |
| 2 | Token日预算 | 10元 / 其他 |
| 3 | 代理路由模式 | 智能 / 全局 / 按入口 |
| 4 | 信源优先级 | 网站 / 众包 / 开源 三类如何排序 |
| 5 | 接单平台 | Upwork / 自有站点 / 两者 |
| 6 | 财务通道选型 | Tavarov Pay / AllScale / Morph |
| 7 | 是否接入外部消息推送 | 微信 / Telegram / 否 |
| 8 | 初始角色数量 | 6个（完整）/ 2-3个（精简） |
| 9 | 是否启用独立【合规Agent】 | 启用 / 暂不启用 |
| 10 | 首发产品选择 | 市场报告（入门款） / MCP脚本开发 |

## 十七、已补齐缺漏项清单（v2.0原始12项全部完成）

| 序号 | v2.0缺漏项 | v2.1补齐内容 |
|:---|:---|:---|
| 1 | 具体产品方向 | 三层产品矩阵，匹配工业视觉背景 |
| 2 | 定价策略 | 分产品美元定价、优惠策略 |
| 3 | 获客渠道细化 | Upwork以外5条获客路径+冷启动方案 |
| 4 | 法律实体注册流程 | 美国怀俄明LLC注册步骤、一次性/年度成本 |
| 5 | 税务申报流程 | 稳定币收入记账、汇率换算、凭证留存、会计师外包 |
| 6 | 备份与灾难恢复 | 三层备份+对象存储快照+一键重建脚本 |
| 7 | 版本升级策略 | DSH主/次版本、插件锁版本规则，禁止生产自动升级 |
| 8 | 性能基准 | Token消耗、成功率、ROI、代理失败率等仪表盘指标+告警阈值 |
| 9 | 退出机制 | 业务止损、紧急关停触发条件与动作 |
| 10 | 初始种子预算 | $300种子资金分配方案 |
| 11 | 代理流量成本估算 | 住宅代理单价、月度用量区间、成本区间 |
| 12 | 合规Agent的职责定义 | 新增独立只读审计角色、权限边界、工作内容 |

---

**文档结束**

*本方案基于DeepSeek Harness v0.1.0-rc.7生态整理，部分插件为社区开发，版本兼容性需实际验证。建议按"代理验证期→信源验证期→最小闭环期→财务接入期→稳定运转期→进化期"分阶段推进。*
