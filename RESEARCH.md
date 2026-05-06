# Claude for Financial Services — 深度调研分析

> 本调研基于对 `anthropics/financial-services` 仓库的全面分析，涵盖项目结构、Claude 耦合度、数据源依赖以及 Tushare 等中国数据源的替代可行性。

---

## 一、项目结构总览

本项目是 Anthropic 为金融服务行业打造的参考 Agent/Plugin 仓库，核心理念是 **"同一来源，两种交付"**：

- **Claude Cowork Plugin**：分析师在 Claude 桌面端/网页端安装插件使用
- **Claude Managed Agent (CMA)**：通过 `POST /v1/agents` API 部署为无头服务

### 核心目录

```
plugins/
  agent-plugins/           # 10个端到端工作流Agent插件（pitch-agent, model-builder等）
  vertical-plugins/        # 7个垂直行业技能源（financial-analysis为核心）
  partner-built/           # 2个合作伙伴插件（LSEG, S&P Global）
managed-agent-cookbooks/   # 10个Agent的API部署模板（agent.yaml + subagents）
claude-for-msft-365-install/  # Microsoft 365插件安装工具（独立）
scripts/                   # check.py, sync-agent-skills.py, deploy-managed-agent.sh等
```

### 关键数量

- **10** 个命名 Agent
- **7** 个垂直行业插件
- **2** 个合作伙伴插件
- **~90+** 个 SKILL.md 技能文件
- **11** 个外部 MCP 数据连接器
- **~30+** 个子 Agent YAML 文件

---

## 二、与 Claude 的耦合度分析

### 🔴 强耦合（迁移必须重写）

| 耦合点 | 影响范围 | 说明 |
|---|---|---|
| **模型名硬编码** | ~30+处 | `model: claude-opus-4-7` |
| **Managed Agents API** | `deploy-managed-agent.sh`, `orchestrate.py` | `POST /v1/agents`, `beta.agents.sessions.stream()` 等 Anthropic 独占预览API |
| **工具集类型名** | 所有 Agent YAML | `type: agent_toolset_20260401`, `type: mcp_toolset` 是 Anthropic CMA schema 约定 |
| **认证方式** | `deploy-managed-agent.sh` | `x-api-key: $ANTHROPIC_API_KEY`, `anthropic-version`, `anthropic-beta` header |
| **Cowork Plugin 体系** | `.claude-plugin/` 全部 | 插件发现、安装、`/comps`, `/dcf` 命令解析基于 Claude Code / Cowork 运行时 |
| **Skills 上传 API** | `deploy-managed-agent.sh` | `/v1/skills` multipart upload 是 Anthropic 独占 |

### 🟡 中等耦合（需要适配）

| 耦合点 | 说明 |
|---|---|
| **`handoff_request` 协议** | 跨Agent通信通过输出文本嵌入 `{"type": "handoff_request", ...}`，需改为函数调用或消息队列 |
| **Office JS 分支逻辑** | 部分 skill（如 `dcf-model`）内含 Office JS 环境假设 |
| **Claude for Microsoft 365** | `claude-for-msft-365-install/` 目录对其他LLM平台无用 |

### 🟢 弱耦合/可复用（基本无需修改）

| 组件 | 说明 |
|---|---|
| **MCP 协议本身** | 开放标准，`.mcp.json` 配置通用 |
| **Skills 内容** | 大部分金融建模方法论、分析流程是纯领域知识 |
| **Agent 系统提示词** | 角色定义、工作流、guardrails 是通用 prompt engineering |
| **JSON Schema 输出定义** | 标准 JSON Schema，任何支持 structured output 的平台可用 |
| **校验脚本** | `check.py`, `validate.py`, `sync-agent-skills.py` 是本地文件操作 |

### 耦合度量化

```
~50% 工作量可复用：skills、prompts、MCP配置、校验脚本
~30% 工作量需重写：部署脚本、API适配、orchestrate.py
~20% 工作量需适配：YAML manifest格式转换、handoff协议
```

---

## 三、现有数据源及认证情况

### 3.1 11个商业 MCP 连接器

定义在 `plugins/vertical-plugins/financial-analysis/.mcp.json`：

| 数据源 | MCP 端点 | 认证要求 | 说明 |
|---|---|---|---|
| **Daloopa** | `https://mcp.daloopa.com/server/mcp` | ✅ 企业订阅 | 结构化历史财务数据 |
| **Morningstar** | `https://mcp.morningstar.com/mcp` | ✅ 订阅 | 基金分析 |
| **S&P Global (Kensho)** | `https://kfinance.kensho.com/integrations/mcp` | ✅ **明确需订阅** | README写明需 S&P Global LLM-ready API subscription |
| **FactSet** | `https://mcp.factset.com/mcp` | ✅ 机构订阅 | 共识预期、供应链 |
| **Moody's** | `https://api.moodys.com/genai-ready-data/m1/mcp` | ✅ API key | 信用评级 |
| **MT Newswires** | `https://vast-mcp.blueskyapi.com/mtnewswires` | ✅ 订阅 | 实时财经新闻 |
| **Aiera** | `https://mcp-pub.aiera.com` | ✅ 订阅 | Earnings call AI |
| **LSEG** | `https://api.analytics.lseg.com/lfa/mcp` | ✅ **需有效凭证** | LSEG README: "valid credentials" |
| **PitchBook** | `https://premium.mcp.pitchbook.com/mcp` | ✅ Premium订阅 | PE/VC交易数据 |
| **Chronograph** | `https://ai.chronograph.pe/mcp` | ✅ 订阅 | PE估值基准 |
| **Egnyte** | `https://mcp-server.egnyte.com/mcp` | ✅ 租户认证 | 企业文件管理 |

> **重要**：`.mcp.json` 中只写了公开URL，但**每个都是付费/机构级服务**。MCP协议把认证封装在HTTP层，所以配置文件里看不到具体的key。README明确注明：*"MCP access may require a subscription or API key from the provider."*

### 3.2 Agent 实际启用的自定义 MCP 端点

各 `managed-agent-cookbooks/<slug>/agent.yaml` 通过环境变量注入：

| 环境变量 | 用途 | 对应Agent |
|---|---|---|
| `${CAPIQ_MCP_URL}` | Capital IQ 数据 | pitch-agent, model-builder, market-researcher, meeting-prep-agent |
| `${DALOOPA_MCP_URL}` | Daloopa 财务数据 | pitch-agent, model-builder, earnings-reviewer, market-researcher |
| `${FACTSET_MCP_URL}` | FactSet 数据 | earnings-reviewer, market-researcher |
| `${GL_MCP_URL}` | 内部总账系统 | gl-reconciler, month-end-closer |
| `${SUBLEDGER_MCP_URL}` | 内部子账系统 | gl-reconciler |
| `${PORTFOLIO_MCP_URL}` | 内部持仓系统 | valuation-reviewer |
| `${NAV_MCP_URL}` | 内部NAV系统 | statement-auditor |
| `${CRM_MCP_URL}` | 内部CRM系统 | meeting-prep-agent |
| `${SCREENING_MCP_URL}` | 制裁/PEP筛查 | kyc-screener |

> 所有敏感凭证都**不commit到仓库**，通过运行时环境变量注入。`AGENTS.md` 明确写道：*"credentials are injected via environment variables not committed to the repo."*

---

## 四、Tushare 数据覆盖能力分析

### 4.1 Tushare 简介

[Tushare](https://tushare.pro) 是国内最成熟的金融数据接口平台之一，采用**积分制**权限管理。基础数据免费或低积分，深度/高频数据需2000–5000+积分。

### 4.2 Tushare vs 项目数据需求 — 完整映射表

#### 公司财务数据

| 项目中的具体需求 | 当前数据源 | Tushare覆盖 | 免费平替 | 说明 |
|---|---|---|---|---|
| 利润表/资产负债表/现金流量表 | Daloopa, FactSet, CapIQ | ✅ **完全覆盖** | AKShare, Baostock | `income`/`balancesheet`/`cashflow` |
| 财务指标（ROE、毛利率等） | Daloopa, CapIQ | ✅ **完全覆盖** | AKShare | `fina_indicator` |
| 分行业/产品收入构成 | CapIQ, 10-K | ✅ **覆盖** | — | `fina_mainbz` |
| 业绩预告/快报 | FactSet | ✅ **覆盖** | — | `forecast`/`express` |
| 分红送股历史 | CapIQ | ✅ **覆盖** | — | `dividend` |
| **分部门/segment数据** | CapIQ, 10-K | ❌ **无** | Wind/Choice | — |
| **私有公司财务** | CapIQ, CIMs | ❌ **无** | — | — |

#### 股票行情与市场数据

| 项目中的具体需求 | 当前数据源 | Tushare覆盖 | 免费平替 | 说明 |
|---|---|---|---|---|
| 日线/周线/月线行情 | CapIQ, FactSet | ✅ **完全覆盖** | AKShare, Baostock | `daily`/`weekly`/`monthly` |
| 复权行情 | CapIQ, FactSet | ✅ **完全覆盖** | AKShare | `pro_bar` (adj='qfq/hfq') |
| PE、PB、PS、换手率、市值 | CapIQ, FactSet | ✅ **完全覆盖** | AKShare | `daily_basic` |
| 分钟级行情 | FactSet | ✅ **需高积分** | AKShare | Tushare 2000+积分；AKShare免费 |
| Beta系数、10Y国债收益率 | Web搜索, CapIQ | ❌ **无** | 中证指数, 中国货币网 | 需自行计算或买Wind |
| 个股资金流向 | Bloomberg | ✅ **覆盖** | AKShare | `moneyflow` |
| 股东人数/增减持 | CapIQ | ✅ **覆盖** | — | `stk_holdernumber`/`stk_holdertrade` |
| **EV multiples（EV/EBITDA等）** | FactSet, Bloomberg | ❌ **无预计算** | 需自行计算 | — |

#### 宏观数据

| 项目中的具体需求 | 当前数据源 | Tushare覆盖 | 免费平替 | 说明 |
|---|---|---|---|---|
| 中国GDP、CPI、PPI | LSEG | ✅ **完全覆盖** | 国家统计局API | `cn_gdp`/`cn_cpi`/`cn_ppi` |
| 中国PMI | LSEG | ✅ **完全覆盖** | 国家统计局 | `cn_pmi` |
| 中国M0/M1/M2 | LSEG | ✅ **完全覆盖** | 央行官网 | `cn_m` |
| LPR/SHIBOR | LSEG | ✅ **覆盖** | 中国货币网 | `shibor_lpr`/`shibor` |
| **美国10Y国债（DCF risk-free rate）** | Web搜索 | ❌ **无** | **FRED API** | 完全免费，REST接口，项目中刚需！ |
| **美国GDP/CPI/PMI** | LSEG | ❌ **无** | FRED API, 世界银行 | 项目中90%宏观skill假设美国数据 |
| **利率互换曲线** | LSEG | ❌ **无** | 中国货币网 | 需CFETS专业数据 |

#### 分析师一致预期 — **最大缺口**

| 项目中的具体需求 | 当前数据源 | Tushare覆盖 | 免费平替 | 说明 |
|---|---|---|---|---|
| 共识EPS、营收（季度/年度） | FactSet, S&P Kensho | ❌ **几乎无** | Wind/Choice/iFinD (付费) | 国内无免费IBES级共识数据 |
| 分析师数量、high/low估计 | FactSet, LSEG IBES | ❌ **无** | Wind/Choice (付费) | — |
| 估计修正历史 | FactSet, Bloomberg | ❌ **无** | — | — |
| 评级变化 | FactSet, MT Newswires | ❌ **无** | — | — |

> **这是项目中最大的数据缺口**。`earnings-reviewer`、`earnings-preview-beta`、`tear-sheet`、`initiating-coverage` 等核心 skill 都重度依赖共识预期。

#### 新闻/公告

| 项目中的具体需求 | 当前数据源 | Tushare覆盖 | 免费平替 | 说明 |
|---|---|---|---|---|
| 财经新闻/快讯 | MT Newswires | ⚠️ **部分覆盖** | AKShare, 新浪财经 | Tushare聚合多源新闻 |
| 上市公司公告 | SEC EDGAR | ✅ **覆盖** | 巨潮资讯网 | `disclosure_date` + 巨潮免费PDF |
| Earnings call transcript | Aiera | ❌ **无** | — | 中国业绩说明会文字记录无成熟免费源 |
| 业绩日历 | FactSet | ✅ **覆盖** | 东方财富 | Tushare有A股财报披露计划 |

#### 基金数据

| 项目中的具体需求 | 当前数据源 | Tushare覆盖 | 免费平替 | 说明 |
|---|---|---|---|---|
| 公募基金列表/净值 | Morningstar | ✅ **完全覆盖** | AKShare | `fund_basic`/`fund_nav` |
| ETF成分股/权重 | Morningstar | ✅ **覆盖** | AKShare | `fund_portfolio` |
| 基金分红 | Morningstar | ✅ **覆盖** | — | `fund_div` |
| 基金评级/业绩归因 | Morningstar | ❌ **无** | — | 国内无免费基金评级API |

#### 债券/外汇/期权

| 项目中的具体需求 | 当前数据源 | Tushare覆盖 | 免费平替 | 说明 |
|---|---|---|---|---|
| 中国国债/可转债行情 | LSEG | ✅ **部分覆盖** | 中国货币网, AKShare | `cb_basic`/`cb_daily` |
| 债券定价/Z-spread/信用利差 | LSEG YieldBook, Moody's | ❌ **无** | — | 无免费替代 |
| 外汇现货行情 | LSEG | ✅ **基础覆盖** | 中国货币网 | `fx_daily` |
| 远期汇率/波动率曲面 | LSEG | ❌ **无** | — | 无免费替代 |
| 上证50ETF期权 | LSEG | ✅ **覆盖** | — | `opt_basic`/`opt_daily` |
| 个股期权Greeks | LSEG | ❌ **无** | — | 无免费替代 |

#### PE/VC数据 — **完全无覆盖**

| 项目中的具体需求 | 当前数据源 | Tushare覆盖 | 付费平替 | 说明 |
|---|---|---|---|---|
| 融资轮次/估值 | PitchBook, S&P CapIQ | ❌ **无** | 清科/投中 (付费) | — |
| M&A交易记录 | PitchBook, CapIQ | ❌ **无** | Wind/Choice | — |
| Pre/post-money估值 | PitchBook | ❌ **无** | — | — |
| LP/GP数据 | PitchBook, Chronograph | ❌ **无** | — | — |

#### 内部系统数据

| 项目中的具体需求 | 当前数据源 | Tushare覆盖 | 替代方案 |
|---|---|---|---|
| GL总账/子账 | internal-gl MCP | ❌ **无** | 自建MCP server连接ERP |
| Portfolio持仓/NAV | portfolio MCP | ❌ **无** | 自建MCP server |
| CRM客户数据 | crm MCP | ❌ **无** | 自建MCP server |
| KYC筛查 | screening MCP | ❌ **无** | 自建MCP server |

---

## 五、免费平替全景图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    中国金融数据免费/低成本生态                         │
├─────────────────────────────────────────────────────────────────────┤
│  完全免费                                                           │
│  ├── AKShare ──── 股票/基金/期货/外汇/债券/实时行情/技术指标         │
│  ├── Baostock ─── A股历史K线/财务数据（免注册）                     │
│  ├── 国家统计局 ── GDP/CPI/PPI/工业增加值等全部宏观指标              │
│  ├── 中国人民银行 ─ M0/M1/M2/LPR/存准率/社融                       │
│  ├── 巨潮资讯网 ── A股全部财报/公告/招股书（PDF）                   │
│  ├── 中国货币网 ── 银行间利率/债券/外汇/互换                        │
│  ├── 中证指数公司 ─ 指数成分股/权重                                  │
│  └── FRED API ─── 美国宏观（GDP/利率/CPI/就业）── 项目中刚需！      │
├─────────────────────────────────────────────────────────────────────┤
│  低成本（积分制/年费<¥1000）                                        │
│  ├── Tushare Pro ─ 股票/财务/宏观/基金/特色数据（积分制）            │
│  ├── 理杏仁 ────── 基本面深度数据                                    │
│  └── Efinance ──── 东方财富实时行情                                  │
├─────────────────────────────────────────────────────────────────────┤
│  机构级付费（Wind的平替）                                          │
│  ├── 东方财富Choice ── <¥5,000/年，覆盖90% Wind功能                │
│  ├── 同花顺iFinD ──── Wind的50-70%价格                            │
│  └── 聚宽JoinQuant ── 量化+数据                                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、实操建议

### 6.1 Tushare 能立即做什么

如果 `TUSHARE_TOKEN` 已在环境变量中，建议走 **"中国市场专用Agent"** 路线，而不是试图替换现有欧美 skill：

1. **新建 `china-market-researcher` Agent**
   - 覆盖：A股行业研究、个股基本面分析、宏观数据跟踪
   - 数据：Tushare `daily_basic`（PE/PB/市值）、`income`/`balancesheet`/`cashflow`（三大报表）、`cn_gdp`/`cn_cpi`（宏观）

2. **新建 `china-fund-screener` Agent**
   - 覆盖：公募基金筛选、ETF成分股分析
   - 数据：Tushare `fund_basic`/`fund_nav`/`fund_portfolio`

3. **改造 `model-builder` 的 subagent（A股版本）**
   - 针对A股标的做DCF/可比分析
   - 用Tushare财务数据 + 自行计算WACC

### 6.2 需要补免费源的

| 项目skill需求 | 补什么 | 为什么 |
|---|---|---|
| 美国宏观（10Y国债、GDP、CPI） | **FRED API** | 完全免费，REST接口，DCF模型刚需risk-free rate |
| 实时行情/A股当日数据 | **AKShare** | 完全免费，不受Tushare积分限制 |
| 技术指标（KDJ、MACD等） | **AKShare** | Tushare没有技术指标计算 |

### 6.3 必须接受缺口的（或付费）

| 项目skill需求 | 缺口 | 方案 |
|---|---|---|
| 分析师一致预期 | 国内无免费源 | 用Tushare `forecast`（业绩预告）代替；或采购Choice |
| M&A/PE数据 | 无免费源 | 完全无法替代，需清科/投中/Wind |
| Earnings call transcript | 无免费源 | 暂时用公告+新闻代替 |
| 债券定价/外汇远期/期权Greeks | 无免费源 | 这些LSEG skill只能保留给有订阅的用户 |

### 6.4 Tushare 集成路径

Tushare 官方没有官方 MCP Server，但社区已有成熟实现：

| 项目 | 工具数 | 模式 | 推荐度 |
|---|---|---|---|
| `zhewenzhang/tushare_mcp` | 52个 | stdio + HTTP SSE | ⭐ 最推荐 |
| `erwanjun/tushare-mcp-server` | 多类别 | stdio | 可用 |
| `buuzzy/tushare_MCP` | 30+ | HTTP (Render托管) | 仅POC |

**推荐部署方式**：
1. Fork `zhewenzhang/tushare_mcp`
2. 部署为内部 HTTP SSE 服务
3. 在 `.mcp.json` 中新增 `tushare` 连接器
4. 在相关 skills 中添加中国市场分支逻辑

---

## 七、总结

- **~50%** 的项目内容（skills、prompts、校验脚本）是通用金融领域知识，与Claude弱耦合
- **~50%** 的内容（部署脚本、API调用、Plugin体系、handoff协议）与Anthropic生态强耦合
- **Tushare 技术上完全可以接入**，最佳路径是自建/托管社区 MCP Server，与现有 `.mcp.json` 连接器架构完全兼容
- Tushare 不会"替代"现有数据源，而是作为**中国市场的补充数据源**存在
- 项目中的 skills **硬编码了欧美市场假设**（US GAAP、SEC EDGAR、IBES共识），若要服务中国市场，**需要重写skills**而非仅仅替换数据源

---

*调研日期：2026-05-06*
