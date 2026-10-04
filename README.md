# OneCompany-AI-Agent

一人AI自我进化公司：DeepSeek Harness 多Agent自动化商业系统（One-Person AI Company Solution）

> **Version**: v2.1
> **Date**: 2026-10-04
> **Token来源**: DeepSeek API
> **公司架构**: 参考 VoxYZ 6角色闭环 + 独立合规Agent

## 项目简介

基于 DeepSeek Harness（DSH）构建的多Agent一人公司系统。7个AI角色组成虚拟团队，完成市场挖掘、行业调研、内容产出、项目交付、质检审计；搭配代理池实现全球互联网信息采集，目标实现海外技术服务的商业闭环。

> ⚠️ **重要声明**：虚拟货币相关业务在中国境内属于非法金融活动。文档中稳定币收款方案仅为**境外注册实体在海外司法辖区独立运营**的技术推演，境内主体禁止使用该收款方案。所有法律、税务风险由运营实体自行承担。

## 仓库目录

```
OneCompany-AI-Agent/
├── README.md                     # 仓库首页（本文件）
├── docs/
│   ├── OneCompany_Solution.md    # 完整方案文档 v2.1
│   ├── DEPLOYMENT.md             # 部署方案（环境/插件/代理/预算/常驻/FAQ）
│   └── CLIENT.md                 # 智能公司客户端说明（桌面窗口风格）
└── LICENSE                       # MIT License
```

## 核心能力

1. **多Agent虚拟团队**：Minion / Sage / Scout / Quill / Xalt / Observer / 合规Agent
2. **智能代理路由**：多代理池，自动故障切换
3. **全球市场信源采集**：网页搜索、GitHub开源监控、众包平台MCP
4. **预算熔断机制**：Token成本管控，业务止损
5. **任务审批流**：小额自动审批，>100美元项目强制人工审批
6. **灾备备份、性能监控、外部告警推送**
7. **智能公司客户端**：桌面窗口风格的管理控制台（总览/组织/任务/审批/资料库/动态 6 大视图，详见 docs/CLIENT.md）

## 快速开始

1. 安装 Node.js 24 + pnpm
2. 安装 DSH：`npm install -g @deepseek-ai/dsh@latest`
3. 创建独立 Profile 并安装所有插件（详见 docs/OneCompany_Solution.md；部署操作见 docs/DEPLOYMENT.md）
4. 配置环境变量 DeepSeek API Key
5. 配置代理池与预算 YAML
6. PM2 常驻启动 DSH Web UI

## 实施阶段

1. 代理验证期（第1周）
2. 信源验证期（第2周）
3. 最小闭环期（第3-4周）
4. 财务接入期（第2月）
5. 稳定运转期（第2-3月）
6. 自我进化期（第4月+）

## 风险提示

- DSH 属于开发者预览版，存在插件崩溃、会话丢失风险
- 代理仅解决网络访问，不等于法律合规
- 大额商业项目必须人工复核，AI 不可独立签约
- Upwork 平台自动化操作存在账号风控风险

## License

MIT
