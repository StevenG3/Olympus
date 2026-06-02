# AGENTS.md — Olympus Deploy & Migration Runbook

> Audience: autonomous coding agents such as Codex or Claude. Read this top to bottom before deploying or migrating Olympus. Stop on any STOP condition. This public runbook uses placeholders only.

## 0. Operating Rules

- Treat **Olympus** as the whole platform name.
- Treat **Hermes** as the natural-language gateway.
- Treat **Aegis** as the trading core.
- Treat **TradingAgents** as the upstream analysis brain.
- Default to paper mode. Do not enable live trading unless the human explicitly asks in the current task.
- Never commit real credentials, host identifiers, Telegram actor IDs, certificate paths, account data, strategy implementation, or proxy endpoint details.
- Bind services to `127.0.0.1`. Do not bind trading services to public interfaces.
- Verify each phase before continuing. Retry transient failures up to three times, then STOP with logs and the failing command.

## 1. Repository Topology

| Repo | Clone URL | Suggested path | Visibility | Purpose |
|---|---|---|---|---|
| Olympus | `https://github.com/StevenG3/Olympus.git` | `~/apps/olympus` | public | System overview and this runbook |
| Aegis | `https://github.com/StevenG3/aegis.git` | `~/apps/aegis` | public | Trading core, risk, execution, bridges, backtest, model gateway |
| Hermes Agent | `https://github.com/StevenG3/hermes-agent.git` | `~/apps/hermes-agent` | public | Natural-language gateway and skills |
| TradingAgents | `https://github.com/TauricResearch/TradingAgents.git` | `~/apps/tradingagents-official` | public upstream | Multi-agent market analysis |
| Aegis Dashboard | `https://github.com/StevenG3/aegis-dashboard.git` | `~/apps/aegis-dashboard` | private | Local read-only dashboard |
| Aegis Strategies | `https://github.com/StevenG3/aegis-strategies.git` | `~/apps/aegis-strategies` | private | Private strategies, competition runs, graduation workflow |
| Ops | `https://github.com/StevenG3/ops.git` | `~/ops` | private | Host operations, disk layout, backups, remote admin |

Do not copy private repository content into this public repository.

## 2. Service Map

All ports below are expected to bind locally.

| Service | Port | Owner | Role |
|---|---:|---|---|
| Aegis orchestrator | 18081 | Aegis | Main API and workflow coordination |
| Analysis adapter | 18085 | Aegis | Analysis API facade |
| Broker bridge | 18086 | Aegis | Broker connectivity |
| Exchange bridge | 18087 | Aegis | Exchange readiness and read-only balance surfaces |
| Backtest service | 18088 | Aegis | Strategy backtests, healthchecks, competition scoring |
| Strategy competition | 18088 `/strategy-competition` | Aegis backtest-service | Candidate ranking and promote/hold/retire recommendations |
| Unified LLM model gateway | 18085 via analysis-adapter | Aegis analysis-adapter | Provider chain fallback, token accounting, and cost logging for analysis calls |
| TradingAgents bridge | 18181 | TradingAgents + Aegis integration | Analysis brain API |
| Hermes | 8644 | Hermes | Gateway webhook/API surface |
| Dashboard | 8910 | private dashboard | Local read-only UI |
| Broker gateway | 4001, 5900 | local operator | Broker gateway process |

Dependency order:

1. TradingAgents bridge
2. Aegis stack
3. Hermes gateway
4. Private dashboard
5. Private strategy competition jobs, if requested

## 3. Prerequisites

```bash
docker --version
docker compose version
git --version
gh --version
```

Expected host invariants:

- Docker and container runtime data live on the SSD system disk.
- Cold data, backups, and new project archives may live on secondary storage.
- GitHub authentication can read private repos only when the operator has granted access.
- Remote dashboard access uses tunnels or private mesh networking.

STOP if Docker is unavailable, GitHub authentication is missing for required private repos, or a host-specific path is unknown.

## 4. Central Config

Create one local ignored config file. Keep it outside git and distribute values into per-project ignored env files.

```bash
mkdir -p ~/aegis-stack
install -m 600 /dev/null ~/aegis-stack/secrets.env
```

Required for paper analysis:

- one LLM provider credential
- provider routing preference, for example a local chain such as `<PROVIDER_A>,<PROVIDER_B>,<PROVIDER_C>`

Optional:

- Telegram bot token
- allowed Telegram users, using placeholders such as `<YOUR_TG_ID>` in docs
- broker or exchange credentials for explicitly requested live operation
- model gateway routing, for example `LLM_PROVIDER_CHAIN=<PROVIDER_A>,<PROVIDER_B>,<PROVIDER_C>`
- model cost hints such as `<PROVIDER>_<MODEL>_INPUT_COST_PER_1K_USD` and `<PROVIDER>_<MODEL>_OUTPUT_COST_PER_1K_USD`
- strategy competition settings such as promote count, score weights, minimum sample size, and bottom-streak retire threshold

Safe defaults:

- `MODE=paper`
- `LIVE_TRADING_ENABLED=false`
- broker live mode disabled
- exchange trading disabled
- confirmation threshold configured conservatively

STOP if no LLM credential is configured for analysis. Do not print credential values in logs.

Model gateway config reference:

```bash
LLM_PROVIDER_CHAIN=<PROVIDER_A>,<PROVIDER_B>,<PROVIDER_C>
LLM_TIMEOUT_SECONDS=30
LLM_MAX_RETRIES=1
OPENAI_INPUT_COST_PER_1K_USD=<PLACEHOLDER>
OPENAI_OUTPUT_COST_PER_1K_USD=<PLACEHOLDER>
ANTHROPIC_INPUT_COST_PER_1K_USD=<PLACEHOLDER>
ANTHROPIC_OUTPUT_COST_PER_1K_USD=<PLACEHOLDER>
```

Competition config reference:

```bash
COMPETITION_PROMOTE_TOP_N=1
COMPETITION_MIN_SAMPLE_SIZE=<PLACEHOLDER>
COMPETITION_BOTTOM_STREAK_RETIRE_THRESHOLD=3
COMPETITION_SCORE_WEIGHTS=<PLACEHOLDER_JSON>
```

Use repository examples or service defaults for exact variable names when they differ. Keep real values in ignored env files only.

## 5. Clone

```bash
mkdir -p ~/apps
cd ~/apps

git clone https://github.com/StevenG3/aegis.git
git clone https://github.com/StevenG3/hermes-agent.git
git clone https://github.com/TauricResearch/TradingAgents.git tradingagents-official
```

Private repos require access:

```bash
git clone https://github.com/StevenG3/aegis-dashboard.git
git clone https://github.com/StevenG3/aegis-strategies.git
git clone https://github.com/StevenG3/ops.git ~/ops
```

If private clone access fails, continue only with public paper-mode services and report the missing private repo.

## 6. Config Distribution

Use each repository's example config where available, then append or import local ignored values from `~/aegis-stack/secrets.env`.

```bash
cp -n ~/apps/aegis/deploy/.env.example ~/apps/aegis/deploy/.env
cat ~/aegis-stack/secrets.env >> ~/apps/aegis/deploy/.env

cp -n ~/apps/tradingagents-official/.env.example ~/apps/tradingagents-official/.env 2>/dev/null || true
cat ~/aegis-stack/secrets.env >> ~/apps/tradingagents-official/.env

mkdir -p ~/apps/data/hermes-official
cp ~/aegis-stack/secrets.env ~/apps/data/hermes-official/.env
chmod 600 ~/apps/data/hermes-official/.env
```

Do not echo env file contents back to the user.

## 7. Deploy

### Phase 1: TradingAgents Bridge

```bash
cd ~/apps/tradingagents-official
docker compose up -d --build
curl -s --max-time 5 http://127.0.0.1:18181/healthz
```

### Phase 2: Aegis

```bash
cd ~/apps/aegis/deploy
docker compose up -d --build

for p in 18081 18085 18086 18087 18088; do
  printf "%s " "$p"
  curl -s --max-time 5 "http://127.0.0.1:$p/healthz"
  echo
done
```

### Phase 3: Hermes

```bash
cd ~/apps/hermes-agent/deploy
docker compose up -d
docker exec -u 10000 hermes rm -f /opt/data/.skills_prompt_snapshot.json 2>/dev/null || true
docker compose restart hermes
```

### Phase 4: Dashboard

```bash
cd ~/apps/aegis-dashboard
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
DASHBOARD_PORT=8910 nohup python server.py >/tmp/olympus-dashboard.log 2>&1 &
curl -s --max-time 5 http://127.0.0.1:8910/ >/dev/null
```

## 8. Verification

```bash
docker ps --format '{{.Names}}\t{{.Status}}' | rg 'aegis|hermes|tradingagents'
curl -s http://127.0.0.1:18081/healthz
curl -s http://127.0.0.1:18088/strategies
curl -s http://127.0.0.1:18087/readyz
curl -s http://127.0.0.1:8910/ >/dev/null
```

Strategy competition endpoint smoke:

```bash
curl -s http://127.0.0.1:18088/strategy-competition \
  -H 'content-type: application/json' \
  -d '{
    "entries": [
      {
        "strategy": "candidate_a",
        "params": {"fast": 20, "slow": 50},
        "healthcheck": {"verdict": "PASS"},
        "key_metrics": {
          "median_return": 6,
          "beat_bh_share": 0.7,
          "sharpe": 1,
          "max_dd": 10,
          "exit_breakdown": {"signal": 3}
        },
        "history": {"bottom_streak": 0}
      },
      {
        "strategy": "blocked_candidate",
        "params": {"fast": 10, "slow": 30},
        "healthcheck": {"verdict": "BLOCK"},
        "key_metrics": {
          "median_return": 30,
          "beat_bh_share": 1,
          "sharpe": 4,
          "max_dd": 5,
          "exit_breakdown": {"signal": 3}
        },
        "history": {"bottom_streak": 0}
      }
    ],
    "promote_top_n": 1
  }'
```

PASS means the response ranks the passing candidate first, marks only advisory statuses such as `promote_candidate`, `hold`, or `retire_candidate`, and never triggers execution.

Model gateway fallback smoke:

```bash
curl -s http://127.0.0.1:18085/healthz
# Then run an analysis request with the first provider intentionally disabled in local ignored config.
# PASS means the response succeeds through the next provider and logs provider, model, latency, token estimate, and estimated cost without printing secrets.
```

Hermes smoke tests, when Telegram or CLI mode is configured:

```bash
docker exec -u 10000 hermes hermes chat -q "show my balances"
docker exec -u 10000 hermes hermes chat -q "backtest BTCUSDT moving averages"
```

PASS means health checks are OK, the dashboard is reachable locally, and natural-language requests route into the expected Aegis skill.

## 9. Strategy Health, Competition, and Graduation

Strategy implementation and competition artifacts live in the private Aegis Strategies repo.

Rules:

- Public Aegis may contain generic scoring, healthcheck, backtest, EV gate, and factor-attribution machinery.
- Private Aegis Strategies contains candidate strategies, parameter grids, competition outputs, and graduation notes.
- The competition arena ranks strategies on common samples and produces `promote_candidate`, `hold`, or `retire_candidate`.
- A blocked healthcheck always ranks below passing candidates.
- `promote_candidate` is advisory only. It does not graduate code, enable live mode, or place orders.
- Graduation requires human review and the existing healthcheck gate.

Expected private outputs:

- `leaderboard.json`
- human-readable competition summary
- healthcheck verdicts
- EV and factor-attribution notes when available

Do not publish these outputs unless the human explicitly sanitizes and approves them.

## 10. Unified LLM Model Gateway

Aegis owns the model gateway for analysis calls. It is not in the order path.

Required behavior:

- Provider chain is configured locally, not hard-coded.
- Fallback moves to the next provider on timeout, rate limit, or provider failure.
- All provider failures return a friendly analysis error.
- Calls record provider, model, latency, token counts, estimated cost, task hint, and success/failure.
- Logs redact credential-like values and provider raw errors when needed.
- The gateway never calls broker, exchange order, withdrawal, or risk bypass APIs.
- `LLM_PROVIDER_CHAIN` controls provider order; unset values should fall back to safe service defaults.
- `*_COST_PER_1K_USD` values are telemetry hints. Missing cost hints must not block analysis.

Compatibility:

- Existing provider defaults may be used as the first chain entry.
- TradingAgents and analysis-adapter should keep analysis semantics unchanged while using the gateway for model calls.

## 11. Migration

### Backup on old host

```bash
mkdir -p ~/olympus-backup

docker run --rm -v aegis_data:/data -v ~/olympus-backup:/backup alpine \
  tar czf /backup/aegis_data.tgz -C /data .

tar czf ~/olympus-backup/hermes_data.tgz -C ~/apps/data/hermes-official .
cp ~/aegis-stack/secrets.env ~/olympus-backup/secrets.env.local
```

Do not commit backup files.

### Restore on new host

```bash
mkdir -p ~/apps ~/aegis-stack
cp ~/olympus-backup/secrets.env.local ~/aegis-stack/secrets.env
chmod 600 ~/aegis-stack/secrets.env

docker volume create aegis_data
docker run --rm -v aegis_data:/data -v ~/olympus-backup:/backup alpine \
  tar xzf /backup/aegis_data.tgz -C /data

mkdir -p ~/apps/data/hermes-official
tar xzf ~/olympus-backup/hermes_data.tgz -C ~/apps/data/hermes-official
```

Then repeat clone, config distribution, deploy, and verification.

STOP if `aegis_data` restore fails. Do not silently start with a fresh ledger or scorecard store.

## 12. Remote Access

Use placeholders in docs:

```bash
ssh -L 8910:127.0.0.1:8910 <user>@<SERVER_IP>
```

Open `http://127.0.0.1:8910` locally after the tunnel is established.

## 13. Safety Invariants

1. Paper mode is default.
2. Live mode requires explicit human instruction, live flags, configured broker or exchange credentials, risk pass, and order confirmation.
3. Backtest, competition, dashboard, and exchange read surfaces are not order execution surfaces.
4. All services bind locally unless a human explicitly requests and reviews a different private-network design.
5. No real host identifiers, Telegram actor IDs, credentials, certificate paths, private strategy code, account data, balances, positions, or private proxy endpoints appear in public repos.
6. Private repos are referenced only by name, URL, and purpose in public docs.

## 14. STOP Conditions

STOP and report if:

- a command would print real credentials or host identifiers
- private strategy code would be copied into a public repo
- live trading is requested without explicit current-task confirmation
- Docker or GitHub authentication is missing and required
- service health fails after three retries
- the safety scan finds non-placeholder sensitive material
- a migration restore fails

## 15. Known Gotchas

- Inside containers, use Docker service DNS for peer services. `127.0.0.1` points to the current container.
- Hermes skill routing may need the skills prompt snapshot removed and the container restarted.
- Dashboard is a local read-only UI; do not treat it as an auth boundary for public exposure.
- Docker build cache can make disk usage look larger than real unique usage.
- Private competition results can include strategy insight; keep them private unless explicitly sanitized.
