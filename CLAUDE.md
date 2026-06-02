# CLAUDE.md — How Claude/LLMs work on Olympus

## 1. What is Olympus

Olympus is an AI autonomous trading platform that routes natural-language requests through Hermes into the Aegis trading core, with TradingAgents as the upstream multi-agent analysis brain. README.md explains the human-facing overview; AGENTS.md is the machine deploy and migration runbook.

Naming map:

- Olympus: umbrella platform.
- Hermes: natural-language gateway.
- Aegis: trading core for risk, execution, bridges, backtests, and model routing.
- TradingAgents: upstream multi-agent market analysis framework.

## 2. Working model

- Claude reviews code, writes focused Codex prompts in `~/docs/CODEX_*.md`, scores outcomes, and may merge approved pull requests.
- Codex executes: edits files, runs scripts, deploys services, verifies results, and reports evidence.
- The human sets direction, confirms risky changes, and gates any live trading.
- Claude should not silently implement or deploy by itself. Its default role is review, prompt design, and synthesis.

## 3. Ironclad rules

- Default to paper mode. Live trading requires explicit current-task human instruction plus fail-closed per-order confirmation.
- Broker, exchange, backtest, screener, and competition bridges are read-only or simulation surfaces unless the human explicitly enables a live execution task.
- Strategy code lives in private strategy storage. Immature strategies stay in incubating ignored storage and do not enter public Aegis.
- Screens, factor scores, backtests, and competition outputs are candidates only. They do not place orders, graduate strategies, or enable live mode.
- Never commit real API keys, host identifiers, Telegram actor IDs, certificate private keys, account data, balances, positions, private proxy endpoints, or private strategy implementation.
- Services bind to `127.0.0.1` by default. Dashboard and local APIs are not public auth boundaries.
- Do not weaken paper defaults, confirmation tokens, healthcheck hard BLOCK gates, candidates-only semantics, or private-repo boundaries.

## 4. Workflow

Each review cycle should follow this shape:

1. Inspect with read-only commands such as `git status`, `git diff`, `rg`, `curl` against local health endpoints, and test logs.
2. Score across completion, code/test/CI quality, safety, process compliance, and usability.
3. Produce one focused `CODEX_*.md` prompt with concrete acceptance criteria.
4. Let Codex execute and verify.
5. Review again before merge or deployment.

## 5. Scoring rubric

Use a 10-point score with a short breakdown:

- Completion: requested behavior and docs are present.
- Quality: code is small, tested, typed or linted where expected, and CI-friendly.
- Safety: paper default, confirmation, local binding, secret redaction, and read-only surfaces are preserved.
- Compliance: pull requests are used for material public-repo changes; private strategy content stays private; outputs remain candidates-only.
- Usability: humans and agents can operate the system from the docs without guessing.

Always list material deductions and the concrete next prompt needed to recover them.

## 6. Repo map and naming

| Repo | Visibility | Role |
|---|---|---|
| Olympus | public | Overview, architecture, public deploy runbook, and LLM working rules |
| Aegis | public | Trading core, risk, execution, bridges, backtest, competition, and model gateway |
| Hermes Agent | public | Natural-language gateway and skill routing |
| TradingAgents | public upstream | Multi-agent analysis brain |
| Aegis Dashboard | private | Local read-only portfolio, PnL, backtest, and monitoring UI |
| Aegis Strategies | private | Private strategies, competition outputs, incubation notes, and graduation workflow |
| Ops | private | Host operations, disks, backups, and remote administration |

Do not copy private repository contents into public repositories.

## 7. Key architecture decisions

- Docker and containerd runtime data should stay on the SSD. Secondary storage such as `/mnt/blockstorage` is for cold data, backups, archives, and large non-latency-sensitive assets.
- Strategy graduation is gated by the seven-part healthcheck. Edge, net-of-cost viability, and non-ruin constraints are hard BLOCK gates.
- EV admission starts in shadow mode until data quality is confirmed; enforce mode requires explicit human approval.
- Python remains the core implementation language because the system is AI-analysis-driven and minute-level. The primary bottlenecks are edge quality, provider latency, broker latency, and data reliability, not language runtime speed.
- Operations-side proxy details stay private. Reality over TCP is the primary remote-administration lane; Hysteria2 over UDP may be treated as an optional fallback only when explicitly requested.

## 8. Known gotchas

- Some GitHub CLI versions do not expose visibility in the same command shape; use `gh api` and inspect `.private` when needed.
- Hermes skill deployment may require copying files into the container, deleting the skills prompt snapshot, and restarting Hermes as UID 10000.
- `df` and `du` answer different questions. Use `sudo du -x` for filesystem-contained usage audits when permissions hide directories.
- Docker service DNS is different from host localhost. Inside a container, `127.0.0.1` points to that container, not the host.
- Competition depends on healthcheck semantics. Blocked candidates must rank below passing candidates even when raw returns look better.
- Backtest outputs and competition leaderboards can reveal strategy edge. Keep them private unless explicitly sanitized and approved.
