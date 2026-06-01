# Olympus Architecture

Olympus is a modular AI autonomous trading platform. The public architecture is intentionally high level: it documents names, boundaries, and safety invariants without exposing private strategy code, credentials, host data, or operational endpoints.

## Name Map

| Layer | Name | Responsibility |
|---|---|---|
| Platform | Olympus | Umbrella system, documentation, deployment discipline |
| Gateway | Hermes | Natural-language entry, skill routing, Telegram/dashboard/CLI integration |
| Core | Aegis | Risk, orchestration, execution, bridges, backtests, model gateway |
| Brain | TradingAgents | Multi-agent analysis upstream |

## System Diagram

```text
Human operator
  |
  | natural language
  v
Hermes
  | routes command to skills
  v
Aegis orchestrator
  |
  +--> analysis-adapter
  |      |
  |      +--> model-gateway
  |      |      |
  |      |      +--> provider chain with fallback
  |      |
  |      +--> TradingAgents bridge
  |
  +--> risk-engine
  |      |
  |      +--> limits, EV gate, confirmation checks
  |
  +--> execution-service
  |      |
  |      +--> paper execution by default
  |      +--> live execution only when explicitly enabled
  |
  +--> broker bridge
  |
  +--> exchange bridge
  |      |
  |      +--> readiness and read-only balance views
  |
  +--> backtest-service
         |
         +--> healthcheck gate
         +--> strategy competition scoring
         +--> factor attribution

Private dashboard reads local Aegis APIs.
Private strategy repo runs strategy competitions and stores candidate outputs.
Ops repo maintains host, disk, backup, and remote-admin procedures.
```

## Data Flow

1. A human sends a natural-language request through Telegram, dashboard, or CLI.
2. Hermes authenticates and routes the request to the correct skill.
3. Aegis orchestrator coordinates analysis, risk, and execution.
4. The analysis path calls TradingAgents through the model gateway where configured.
5. The risk path applies limits, EV checks, and confirmation requirements.
6. The execution path stays in paper mode unless live mode is explicitly enabled.
7. Dashboard and reporting surfaces read state locally and do not place orders.

## Strategy Lifecycle

1. Candidate strategy code is developed privately.
2. Aegis generic backtest and healthcheck tools evaluate the candidate.
3. Competition runs compare candidates and parameter variants on common samples.
4. The leaderboard returns advisory statuses: `promote_candidate`, `hold`, or `retire_candidate`.
5. Human review decides whether a candidate graduates.
6. Graduated strategies still pass risk, EV, and confirmation gates at runtime.

## Model Gateway Boundary

The model gateway exists to make analysis calls more reliable and observable. It provides provider routing, fallback, latency tracking, token accounting, estimated cost, and task hints.

It does not:

- place orders
- withdraw funds
- bypass risk checks
- store real credentials in code
- change strategy graduation rules

## Operations Boundary

Operations may include private network access, backups, disk layout, container runtime maintenance, and self-hosted proxy maintenance such as Reality or Hysteria2. Public documentation may mention those capabilities at a category level only. Real endpoints, credential material, certificate paths, and host-specific identifiers stay private.

## Public Documentation Boundary

Allowed in this repository:

- architecture diagrams
- public repository links
- private repository names and purposes
- placeholder-based deployment and migration steps
- safety invariants

Not allowed in this repository:

- real host identifiers
- Telegram actor IDs
- API credentials
- certificate paths
- private strategy implementation
- private proxy endpoint details
- account balances or positions
- order history
- private operational logs
