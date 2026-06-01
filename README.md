# Olympus — AI Autonomous Trading Platform / AI 自主交易平台

> Natural language in -> multi-agent market analysis -> risk-gated paper or live execution, with private dashboards, strategy evaluation, and an autonomous Claude x Codex development loop.
>
> 自然语言输入 -> 多智能体市场分析 -> 经风控闸门的纸面或实盘执行,配私有看板、策略评估体系,以及 Claude x Codex 自主开发协作环。

Olympus is the umbrella name for the system. This public repository is the human-readable overview and machine-readable deployment runbook. It intentionally contains no live host identifiers, API credentials, private strategy code, certificate paths, positions, balances, or private operational data.

Olympus 是系统总称。本 public 仓库只放总览和部署 runbook,不包含真实主机标识、API 凭证、私有策略实现、证书路径、持仓、余额或私有运维数据。

## Naming / 命名体系

| Name | Meaning | Role / 作用 |
|---|---|---|
| **Olympus** | Mount Olympus / 奥林匹斯 | Umbrella AI autonomous trading platform / AI 自主交易平台总称 |
| **Hermes** | Messenger god / 信使神 | Natural-language gateway: Telegram and dashboard entry / 自然语言网关 |
| **Aegis** | Divine shield / 神盾 | Trading core: risk, execution, orchestrator, bridges, backtests / 交易内核 |
| **TradingAgents** | Analysis brain / 分析大脑 | Upstream multi-agent market analysis framework / 上游多智能体分析框架 |

## Repositories / 代码仓库

| Project | GitHub | Visibility | Role / 作用 |
|---|---|---|---|
| **Olympus** | https://github.com/StevenG3/Olympus | public | Overview, architecture, public deploy runbook / 总览、架构、公开部署 runbook |
| **Aegis** | https://github.com/StevenG3/aegis | public | Trading core and generic services / 交易内核与通用服务 |
| **Hermes Agent** | https://github.com/StevenG3/hermes-agent | public | Natural-language gateway and skill routing / 自然语言网关与技能路由 |
| **TradingAgents** | https://github.com/TauricResearch/TradingAgents | public upstream | Multi-agent analysis brain / 多智能体分析大脑 |
| **Aegis Dashboard** | https://github.com/StevenG3/aegis-dashboard | private | Local positions, PnL, backtest, and monitoring UI / 本地持仓、盈亏、回测与监控看板 |
| **Aegis Strategies** | https://github.com/StevenG3/aegis-strategies | private | Private strategy incubation, competition runs, and graduation workflow / 私有策略孵化、竞赛跑批与毕业流程 |
| **Ops** | https://github.com/StevenG3/ops | private | Host operations, disk layout, backups, proxy maintenance / 主机运维、磁盘布局、备份、代理维护 |

Private repositories are listed only by name and purpose. Their internal implementation, strategy logic, operational data, and credentials are not mirrored here.

private 仓只列名称和用途。其内部实现、策略逻辑、运维数据和凭证不会复制到这里。

## Architecture / 整体架构

```text
             Telegram / Browser / CLI
             自然语言入口
                     |
                     v
        +-----------------------------+
        | Hermes                      |
        | NL routing, skills, auth    |
        | 自然语言路由、技能、鉴权       |
        +--------------+--------------+
                       |
                       v
        +-----------------------------+
        | Aegis Orchestrator          |
        | Risk-gated workflow control |
        | 风控闸门与流程编排            |
        +---+----------+----------+---+
            |          |          |
            |          |          |
            v          v          v
   +-------------+ +---------+ +----------------+
   | Model       | | Risk    | | Execution      |
   | Gateway     | | Engine  | | Service        |
   | LLM routing | | Limits  | | paper/live gate|
   +------+------+ +----+----+ +-------+--------+
          |             |              |
          v             |              v
 +----------------+     |     +-------------------+
 | TradingAgents  |     |     | Broker/Exchange   |
 | bridge         |     |     | bridges           |
 | 分析大脑         |     |     | 券商/交易所桥       |
 +----------------+     |     +-------------------+
                        |
                        v
             +-----------------------+
             | Backtest + Competition|
             | Healthcheck + EV gate |
             | 回测、竞赛、体检、EV闸门 |
             +-----------+-----------+
                         |
                         v
             +-----------------------+
             | Private Dashboard     |
             | Read-only local UI    |
             | 私有只读本地看板        |
             +-----------------------+
```

All HTTP surfaces are intended to bind to `127.0.0.1`. Remote access should use SSH tunnels or a private mesh network. Do not expose trading services directly to the public internet.

所有 HTTP 面默认只绑定 `127.0.0.1`。远程访问应使用 SSH 隧道或私有组网,不要把交易服务直接暴露到公网。

See [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md) for the expanded service map.

## Capabilities / 当前能力

- **Natural-language operations / 自然语言操作**: Hermes routes Telegram, dashboard, or CLI requests into Aegis skills.
- **Risk-gated execution / 风控闸门执行**: paper mode is default; live mode requires explicit enablement and order confirmation.
- **Broker and exchange bridges / 券商与交易所桥**: Aegis separates market data, broker/exchange connectivity, execution, and read-only balance surfaces.
- **Strategy healthcheck graduation gate / 策略体检毕业门禁**: candidate strategies must pass health, robustness, cost, drawdown, and operational checks before graduation.
- **Strategy competition arena / 策略竞赛擂台**: generic leaderboard scoring compares candidate strategies and parameter variants on common backtest or paper-trading samples. Blocked healthchecks are forced to the bottom; winners become `promote_candidate` only, never automatic live strategies.
- **Expected-value gate / EV 门槛**: strategy promotion and order decisions can require positive expected value after costs, not just raw return.
- **Factor attribution / 因子归因**: scorecards can explain whether performance came from trend, volatility, market beta, regime effects, or idiosyncratic strategy edge.
- **Unified LLM model gateway / 统一 LLM 模型网关**: analysis calls can route across configured model providers with fallback, latency tracking, token accounting, estimated cost, and task hints.
- **Private strategy repository / 策略私有仓**: strategy implementations, competition outputs, and incubation notes remain in private storage.
- **Self-hosted ops proxy / 自托管运维代理**: Reality and Hysteria2 are maintained on the operations side for resilient remote administration. This overview only documents the existence of that operations lane, not endpoints or credentials.

## Safety Model / 安全模型

- **Paper by default / 默认纸面**: no real orders unless live mode is explicitly enabled.
- **Fail-closed confirmation / 失败关闭确认闸门**: live order paths require confirmation and risk checks.
- **Local-only services / 仅本地服务**: public exposure is not part of the design.
- **No credentials in git / 凭证不入库**: config files with real values stay local and ignored.
- **Private implementation boundary / 私有实现边界**: private strategy code, proxy details, account data, and host-specific operations stay outside this public overview.
- **Read-only analytics surfaces / 只读分析面**: dashboards, backtests, exchange-balance reads, and model telemetry do not place orders.

## Quick Start: Paper Mode / 快速开始:纸面模式

Paper mode needs Docker, Git, the public repositories, and one LLM credential configured locally. Live trading remains off unless explicitly enabled by the operator.

纸面模式需要 Docker、Git、public 仓库和一个本地配置的大模型凭证。除非操作者显式开启,实盘保持关闭。

```bash
mkdir -p ~/apps
cd ~/apps

git clone https://github.com/StevenG3/aegis.git
git clone https://github.com/StevenG3/hermes-agent.git
git clone https://github.com/TauricResearch/TradingAgents.git tradingagents-official

cp ~/apps/aegis/deploy/.env.example ~/apps/aegis/deploy/.env
# Edit local ignored env files with one LLM credential.

cd ~/apps/aegis/deploy
docker compose up -d --build

curl -s http://127.0.0.1:18081/healthz
```

For the full machine runbook, including TradingAgents, Hermes, dashboard, migration, verification, and STOP conditions, read [AGENTS.md](./AGENTS.md).

完整机器部署 runbook,包括 TradingAgents、Hermes、看板、迁移、验证和 STOP 条件,见 [AGENTS.md](./AGENTS.md)。

## Remote Access / 远程访问

```bash
ssh -L 8910:127.0.0.1:8910 <user>@<SERVER_IP>
```

Then open `http://127.0.0.1:8910` locally. Replace placeholders with your own values. Never commit the real values.

然后在本地打开 `http://127.0.0.1:8910`。请用你自己的值替换占位符,不要把真实值提交进仓库。

## Public Repo Boundary / public 仓边界

This repository may contain architecture, public repository links, placeholder-based runbooks, and safety invariants.

本仓库可以包含架构、public 仓链接、基于占位符的 runbook 和安全不变量。

This repository must not contain real host identifiers, Telegram actor IDs, API credentials, private strategy code, private proxy endpoints, certificate paths, account balances, positions, order history, or private operational logs.

本仓库不得包含真实主机标识、Telegram actor ID、API 凭证、私有策略代码、私有代理端点、证书路径、账户余额、持仓、订单历史或私有运维日志。
