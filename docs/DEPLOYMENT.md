# OneCompany-AI-Agent 部署方案（Deployment Guide）

**版本**：v1.0
**日期**：2026-10-04
**对应方案**：[OneCompany_Solution.md](./OneCompany_Solution.md)（v2.1）
**目标**：将"一人AI自我进化公司"从方案文档落地为可运行的 7×24 小时系统

> ⚠️ 本部署文档为技术操作手册。涉及稳定币收款的部分仅适用于**境外注册实体在海外司法辖区独立运营**的场景，境内主体不得使用。部署前请完整阅读方案文档的合规声明与风险边界章节。

---

## 1. 部署总览

系统由以下层次构成，部署顺序按层次自下而上：

```
┌─────────────────────────────────────────────┐
│  交互层：DSH Web UI（浏览器访问 / 消息推送）    │
├─────────────────────────────────────────────┤
│  组织层：7个Agent（Minion/Sage/Scout/Quill/  │
│          Xalt/Observer/合规Agent）+ 审批流     │
├─────────────────────────────────────────────┤
│  执行层：dsh-browser / 开发工具 / 多媒体工具链  │
├─────────────────────────────────────────────┤
│  信息探嗅层：dsh-web-tools / TrendPulse /     │
│          OSSInsight / Upwork MCP             │
├─────────────────────────────────────────────┤
│  代理层：dsh-egress-router（智能路由）+        │
│          dsh-websearch-direct（按入口开关）    │
├─────────────────────────────────────────────┤
│  底座层：DeepSeek Harness（DSH）+ DeepSeek API│
└─────────────────────────────────────────────┘
       控制层：Token预算 + 熔断（贯穿全局）
```

**推荐运行环境**：Linux VPS（Ubuntu 22.04 LTS，2核4G 起，20GB 磁盘），或开发期使用本机（Windows/macOS 均可，命令等价）。

## 2. 部署前置条件

| 项目 | 要求 | 说明 |
|:---|:---|:---|
| Node.js | 24.x | DSH 运行时要求 |
| 包管理器 | pnpm 11.x | 插件管理 |
| DeepSeek API Key | 有效 Key | 模型推理的唯一来源 |
| 预算 | ¥10/天 起（种子 $300） | 见方案 §12 |
| 代理服务 | （可选）住宅代理/机场 | 海外信息源采集需要，见 §6 |
| 境外主体 | （财务接入期前） | USDT/USDC 收款前置条件，见方案 §7 |

## 3. 分步部署流程

### 3.1 环境准备（Node.js 24 + pnpm）

```bash
# ─── Linux (Ubuntu/Debian) ───
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
export NVM_DIR="$HOME/.nvm" && \. "$NVM_DIR/nvm.sh"
nvm install 24 && nvm use 24

# ─── 安装 pnpm ───
npm install -g pnpm@11.7.0

# ─── 验证 ───
node -v    # 期望 v24.x
pnpm -v    # 期望 11.7.x
```

> Windows 用户：到 nodejs.org 下载 Node.js 24 LTS 安装包，然后同样执行 `npm install -g pnpm@11.7.0`。

### 3.2 安装 DeepSeek Harness（DSH）

```bash
npm install -g @deepseek-ai/dsh@latest
dsh --version   # 验证安装
```

> DSH 当前为开发者预览版（v0.1.x），安装后先 `dsh doctor` 自检（如支持）。

### 3.3 创建独立 Profile

为一人公司创建专属配置空间，避免污染其他 DSH 配置：

```bash
dsh profile create ai-company
dsh profile use ai-company
```

### 3.4 配置环境变量

创建 `.env` 文件（生产环境建议放 `~/.dsh/profiles/ai-company/.env`）：

```env
# ─── DeepSeek API（必填）───
DEEPSEEK_API_KEY=sk-your-key-here
DEEPSEEK_BASE_URL=https://api.deepseek.com
DEEPSEEK_MODEL=deepseek-flash

# ─── 代理（可选，按需填写）───
# PROXY_URL=http://127.0.0.1:7890            # 本地代理出口
# RESIDENTIAL_PROXY=user:pass@host:port      # 住宅代理

# ─── 消息推送（可选）───
# TELEGRAM_BOT_TOKEN=...
# TELEGRAM_CHAT_ID=...
```

### 3.5 安装核心插件（分四组）

```bash
# ═══ ① 基础开发插件 ═══
npx -y @deepseek-ai/dsh plugin --profile web add @dsh-market/plugin@0.2.1
npx -y @deepseek-ai/dsh plugin --profile web add dsh-browser@0.1.0
npx -y @deepseek-ai/dsh plugin --profile web add dsh-memory@0.1.0
npx -y @deepseek-ai/dsh plugin --profile web add dsh-at-file@0.6.3

# ═══ ② Token预算与成本控制 ═══
npx -y @deepseek-ai/dsh plugin --profile web add dsh-agent-budget
npx -y @deepseek-ai/dsh plugin --profile web add dsh-fuse
npx -y @deepseek-ai/dsh plugin --profile web add dsh-cost-meter
npx -y @deepseek-ai/dsh plugin --profile web add dsh-cost-ledger

# ═══ ③ 团队编排 ═══
npx -y @deepseek-ai/dsh plugin --profile web add @nanmicoder/dsh-agent-teams@latest

# ═══ ④ 公司管理 UI 与监控 ═══
dsh plugin --profile web add "github:xiazhi88/dsh-onecompany"
npx -y @deepseek-ai/dsh plugin --profile web add @leetoners/dsh-ui-subagent-monitor
npx -y @deepseek-ai/dsh plugin --profile web add dsh-notify-plugin
```

### 3.6 配置代理层（全球出站通道）

代理层解决"国内宽带无法直连海外信息源"的问题。推荐组合：**dsh-egress-router（智能模式）为主 + dsh-websearch-direct（按入口开关）为辅**。

```bash
# ─── 安装代理插件 ───
npx -y @deepseek-ai/dsh plugin --profile web add gswenxue/dsh-egress-router
npx -y @deepseek-ai/dsh plugin --profile web add MYCF711/dsh-websearch-direct
```

路由策略（写入 egress-router 配置）：

| 流量类型 | 路由 | 说明 |
|:---|:---|:---|
| DeepSeek API | 直连 | 国内可直连，走代理增加延迟 |
| X/Twitter、Reddit、Google | 代理池 | 国内受限；多节点轮换防封禁 |
| 国内信息源（小红书、百度） | 直连 | 节省代理流量 |
| 海外 API（Exa/Tavily/Firecrawl） | 按需 | 根据连通性自动选择 |

### 3.7 安装信息探嗅层

```bash
# ═══ 网站信息源 ═══
npx -y @deepseek-ai/dsh plugin --profile web add A3Boy/dsh-web-tools
npx -y @deepseek-ai/dsh plugin --profile web add modsearch

# ═══ MCP 工具（按需） ═══
# Trend Pulse（37来源趋势聚合，Scout 核心工具）
pip install mcp-trendpulse
# Upwork MCP（接单/交付，官方 MCP Server）
pip install upwork-mcp
# OSSInsight MCP（GitHub 100亿+事件分析，Sage 核心工具）
# 安装方式：@iflow-mcp/mcp-srv-ossinsight
```

> 注意：方案 v2.1 已移除 RentAHuman.ai；真人外包一律走 Upwork MCP，避免平台争议与资金托管风险。

### 3.8 配置 Token 预算与熔断

创建预算配置文件（DSH 加载路径以实际版本为准，一般位于 profile 目录）：

```yaml
# budget.yaml
budgets:
  - limitUsd: 10        # 每日预算（约 ¥10/天）
    window: day
  - limitUsd: 300       # 月度预算
    window: month
policies:
  maxReasoningEffort: medium
  allowedModels:
    - deepseek/deepseek-flash
  schedule:
    preferOffPeak: true     # 优先空闲时段（价格减半）
```

**熔断规则（Cap Gates）**：预算耗尽时，入口处直接拒绝新提案，不进入队列。

### 3.9 首次启动与功能验证

```bash
# ─── 启动 DSH Web UI ───
npx @deepseek-ai/dsh web

# 浏览器访问 http://localhost:3000 （端口以实际输出为准）
```

**启动自检清单**（对应方案 §14 的"代理验证期 + 信源验证期"）：

1. DSH Web UI 可访问，左侧边栏出现"一人公司"入口（dsh-onecompany 插件生效）；
2. 在任一员工会话提问，确认 DeepSeek API 正常响应；
3. 让 Scout 执行一次海外搜索（X/Twitter、Google 主题），确认代理路由生效；
4. 让 Sage 调用 OSSInsight 查询一个 GitHub 仓库指标，确认 MCP 连通；
5. 打开"动态页"确认 Token 消耗开始记账，预算仪表盘正常。

## 4. 7×24 常驻运行与监控

### 4.1 PM2 常驻

```bash
npm install -g pm2

# ─── 启动（生产推荐） ───
pm2 start "npx @deepseek-ai/dsh web" --name ai-company
pm2 save
pm2 startup   # 开机自启（按提示执行输出命令）

# ─── 心跳保活（每5分钟检查重启） ───
crontab -e
# 添加：
*/5 * * * * cd /path/to/dsh && pm2 restart ai-company --update-env
```

### 4.2 告警推送（可选）

安装 `dsh-notify-plugin` 后，在设置中配置通知渠道（Telegram / Webhook / Server酱），订阅事件：

- 对话完成 / 对话失败
- 等待审批（>100美元项目必须人工审批）
- 需要授权
- TODO 进度推进

### 4.3 性能监控基准（Observer 采集）

| 指标 | 基线 | 告警阈值 | 处置 |
|:---|:---|:---|:---|
| 单份报告 Token 消耗 | <120k | >200k | 终止任务，复盘 prompt |
| 任务成功率 | >90% | <70% | 冻结同类任务 |
| 收入/Token ROI | >$5/$1 | <$2/$1 | 业务策略复盘 |
| 代理采集失败率 | <5% | >15% | 切换代理池节点 |
| 缓存命中率 | >85% | <60% | 优化 prompt 与缓存 |

## 5. 部署自检清单（Checklist）

部署完成后逐项勾选：

- [ ] Node.js 24 + pnpm 11.7 安装并验证
- [ ] DSH 安装成功，`dsh --version` 可输出
- [ ] Profile `ai-company` 已创建并激活
- [ ] `.env` 已配置 DeepSeek API Key（模型可正常应答）
- [ ] 4 组核心插件安装无报错，`dsh-onecompany` 入口出现
- [ ] 代理插件已安装，海外搜索验证通过
- [ ] 预算配置生效，动态页有 Token 记账
- [ ] 至少 2 个 Agent 完成过一次真实任务
- [ ] PM2 常驻 + crontab 心跳已配置
- [ ] 首次全量备份已执行（见 §7）
- [ ] （财务接入期）境外主体 + Tavarov Pay 测试模式就绪

## 6. 常见问题排查（FAQ）

| 问题 | 可能原因 | 处置 |
|:---|:---|:---|
| API 报错/超时 | Key 无效 / 余额不足 | 检查 `DEEPSEEK_API_KEY`、控制台余额 |
| 海外搜索全部失败 | 代理未生效/节点失效 | 检查 egress-router 状态，切换节点，确认智能模式 |
| 插件崩溃/会话损坏 | DSH 预览版已知问题 | 重启会话；升级前备份（见 §7、§8） |
| 任务被拒绝 | 预算熔断触发 | 动态页查看消耗；提高预算或等待窗口重置 |
| 队列堆积 | Cap Gates 未生效 | 确认入口拒绝配置，检查预算 yaml 加载 |
| Upwork 接入被风控 | 非官方方式接入 | 仅用官方 upwork-mcp，禁用浏览器自动化方案 |
| 收款异常 | 主体未合规 | 稳定币收款前必须完成境外主体注册（方案 §7） |

## 7. 备份与灾难恢复

三层备份策略（对应方案 §9.1）：

1. **应用层**：PM2 状态保存；DSH 配置、Agent 角色定义、预算 YAML 每小时提交私有 Git 仓库；
2. **数据层**：任务记录、交付产物、Token 日志、链上交易记录每日凌晨备份到 S3 兼容对象存储，保留 30 天快照；
3. **灾备层**：VPS 故障时一键重建 DSH 环境，拉取备份恢复。

> 熔断保护：备份连续失败 → 系统暂停新任务并推送告警，防止数据丢失。

## 8. 版本升级策略

- **DSH 主版本**（0.1.x → 0.2.x）：禁止生产自动升级；先在测试环境验证全部插件兼容性，通过后再升级生产；
- **小版本补丁**：测试环境每周自动更新，无报错后低峰时段更新生产；
- **插件**：锁定版本号，禁用 `latest` 标签；每次升级前快照备份。

---

**部署完成后的下一步**：按方案 §14 实施路径推进——代理验证期（第1周）→ 信源验证期（第2周）→ 最小闭环期（第3-4周）→ 财务接入期（第2月）→ 稳定运转期（第2-3月）→ 进化期（第4月+）。

*本部署文档基于 DeepSeek Harness v0.1.0-rc.7 生态整理，部分插件为社区开发，实际命令与配置以安装时的插件文档为准。*
