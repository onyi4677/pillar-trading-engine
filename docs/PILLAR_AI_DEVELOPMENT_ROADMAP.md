# Pillar AI Development Roadmap

**Project:** Pillar AI Trading Engine  
**Repository:** `onyi4677/pillar-trading-engine`  
**Status:** Agreed development direction; implementation has not yet begun  
**Functional reference:** `crypto-bot-august-2026.tar` backup supplied for the refactoring project  
**Initial deployment target:** AWS Singapore (`ap-southeast-1`)  
**Initial exchange:** Binance (subject to account, API and jurisdictional eligibility)

---

## 1. Vision

Develop Pillar AI from its existing automated cryptocurrency trading engine into a reliable, observable, testable and eventually multi-tenant trading platform.

The immediate goal is **not to rewrite the bot or replace its trading intelligence**. The goal is to progressively refactor the large `src/bot.py` file, preserve the working architecture and capabilities in the August 2026 backup, and make the system easier to:

- understand and maintain;
- backtest and evaluate;
- monitor at module and system levels;
- control operationally and through risk limits;
- deploy and update safely;
- extend to independently managed tenant accounts;
- connect to a future client-facing web application.

Pillar AI's existing strategies, indicators, machine-learning components, on-chain intelligence, exchange integrations, risk controls, DCA and scale-in functionality are to be preserved unless a change is explicitly reviewed and approved.

## 2. Guiding engineering principles

1. **Preserve behaviour before optimising it.** Establish a baseline and regression tests before moving or changing trading logic.
2. **Refactor incrementally.** Extract cohesive responsibilities from `bot.py` in small, reviewable steps.
3. **Reuse existing modules.** Inspect the current codebase and its dependency graph before creating new modules or duplicating functionality.
4. **Prove code is unused before removing it.** Search call sites, dynamic references, configuration, scheduled jobs and integrations before deleting legacy methods.
5. **Separate refactoring from strategy changes.** A structural change should not silently alter entries, exits, sizing, leverage or risk settings.
6. **Measure before claiming improvement.** Use repeatable backtests, paper trading and controlled forward tests.
7. **Make every tenant independent.** Credentials, settings, positions, orders, logs, limits and performance data must be isolated by tenant/account.
8. **Secure by default.** Never commit exchange credentials, tokens, private keys, account data or production secrets.

## 3. Starting point and baseline

The supplied August 2026 archive is the functional reference for this project. It was reported to be working at the time it was backed up. The purpose of the audit is to understand and improve its structure, not to presume that it is broken.

The initial archive inspection identified:

- a very large `src/bot.py` containing the main `CryptoTradingBot` class;
- existing service and intelligence modules, including risk management, Binance Futures, Bitget, signals, Telegram, ML models, Harmony, DCA, scale-in and market-regime components;
- duplicated method definitions and potentially obsolete or experimental code that require call-graph verification;
- credential/configuration files in the archive that must not be committed to GitHub.

The archive is the reference snapshot, not an instruction to overwrite the current repository blindly. Before implementation, compare the archive with the repository's current contents and preserve any valid work already present in GitHub.

### Milestone 1 — Baseline audit

Deliverables:

- inventory all classes and methods in `bot.py`;
- classify methods as active, duplicated, unused, obsolete, experimental or uncertain;
- map method call sites, dependencies, class state and configuration references;
- trace the full market-data → analysis → signal → strategy → risk → order → position-management lifecycle;
- identify existing modules that already own each responsibility;
- document current strategies, indicators, thresholds and risk settings;
- identify the current entry points, background tasks, scheduled jobs and exchange connections;
- record existing tests and establish a reproducible baseline;
- create a list of known limitations and unresolved questions without changing live trading behaviour.

**Exit condition:** a reviewed dependency map and a baseline test plan exist before extraction begins.

## 4. Target architecture

The target is a modular architecture built around the existing code, not a wholesale replacement. Exact module paths will be decided after the baseline audit.

| Responsibility | Expected ownership |
|---|---|
| Application orchestration | Starts services and coordinates the trading lifecycle |
| Market data and intelligence | Market feeds, technical data, on-chain intelligence, sentiment and market regime |
| Signal engine | Indicator/model outputs, signal generation and signal validation |
| Strategy engine | Strategy rules, selection, configuration and strategy-specific evaluation |
| Risk management | Exposure, sizing, limits, stop controls, drawdown and pre-trade checks |
| Execution | Exchange adapters, order placement, fills, retries and reconciliation |
| Position management | Open-position lifecycle, DCA, scale-in and exit management |
| Backtesting | Historical data, simulation, fees, slippage and reproducible strategy evaluation |
| Monitoring and reporting | Health checks, metrics, structured logs, alerts and performance summaries |
| Notifications | Telegram and other operational notifications |
| Configuration and secrets | Validated settings, environment-specific configuration and secret references |
| Platform/API services | Future tenant, account, permission and frontend integration services |

The existing `bot.py` may remain temporarily as a compatibility layer or orchestrator while methods are extracted. It should become smaller through tested, incremental changes rather than a single high-risk rewrite.

## 5. Refactoring milestones

### Milestone 2 — Safe modular extraction

- group methods by responsibility;
- extract one cohesive group at a time into the appropriate existing or newly justified module;
- replace direct internal dependencies with explicit interfaces where appropriate;
- avoid circular imports and uncontrolled shared mutable state;
- retain compatibility wrappers where needed during transition;
- run regression tests after every extraction;
- document removed methods and the evidence that they were unused.

**Exit condition:** the refactored engine preserves the baseline behaviour for equivalent inputs and scenarios.

### Milestone 3 — Backtesting and strategy evaluation

Build or strengthen a repeatable backtesting framework that can evaluate individual strategies and the combined engine.

It should account for:

- historical candle and market data quality;
- exchange trading rules and order constraints;
- fees, spread, slippage and realistic fills;
- funding payments where relevant to futures;
- position sizing, leverage and margin constraints;
- stop-loss, take-profit, trailing-stop, DCA and scale-in behaviour;
- warm-up periods and indicator/model data availability;
- prevention of look-ahead bias and data leakage;
- reproducible parameters, datasets and test runs.

Evaluate results separately across trending, sideways/ranging and high-volatility conditions. Compare strategies and long/short directions independently.

### Milestone 4 — Operational metrics and controls

Each major module should expose a health state and relevant metrics.

| Component | Example metrics |
|---|---|
| Market data/intelligence | Feed freshness, missing data, API latency and errors |
| Regime detector | Regime distribution, transitions, confidence and later classification outcomes |
| Signal engine | Signals generated, accepted/rejected, confidence and subsequent outcomes |
| Strategy engine | Trades, net P&L, expectancy and profit factor by strategy |
| Risk manager | Checks passed/blocked, exposure, limit breaches and drawdown |
| Execution engine | Order success/failure, latency, retries, fills and slippage |
| ML models | Prediction metrics, calibration, inference latency and failures |
| Exchange services | Connectivity, rate-limit events, reconciliation differences |
| Notifications | Delivery success, failures and latency |

System-wide reporting should include:

- realised and unrealised P&L, gross and net of fees;
- win rate, average win/loss and expectancy;
- profit factor;
- maximum drawdown and current drawdown;
- exposure, margin use and open positions;
- performance by strategy, symbol, direction and market regime;
- order rejection, execution failure and slippage rates;
- service health, data freshness and API error rates.

Metrics must be derived from recorded events and exchange/account state, not inferred from log messages alone. Monitoring must not expose API secrets or one tenant's information to another tenant.

### Milestone 5 — GitHub release and deployment preparation

- keep the repository private if it contains proprietary implementation details;
- establish a clean dependency manifest and reproducible environment;
- add a reviewed `.gitignore`;
- exclude `.env`, `secret.env`, tokens, credentials, local databases, private account data, caches and generated artifacts;
- rotate any credentials that may have been exposed in backups or previous environments;
- add automated linting, type checks where practical, unit tests and regression tests;
- use reviewed commits and tagged releases;
- prepare a deployment configuration separate from development configuration.

GitHub Actions may later deploy through AWS IAM role federation (OIDC), rather than storing long-lived AWS access keys in GitHub secrets.

### Milestone 6 — AWS Singapore deployment

Initial target region: `ap-southeast-1` (Singapore).

The deployment should include:

- a suitably sized compute runtime for the measured workload;
- a stable outbound network identity where required by exchange API allow-listing;
- secure secret storage, preferably AWS Secrets Manager or an equivalent managed secret store;
- durable storage for configuration, state and required trading records;
- centralised logs, monitoring and alerting;
- service supervision, automatic restart policies and controlled shutdown;
- backups and documented recovery procedures;
- least-privilege IAM permissions;
- environment separation for test, paper-trading and production.

Infrastructure choices and instance sizing must be based on measured CPU, memory, network and storage requirements rather than assumptions.

### Milestone 7 — Binance integration and controlled trading

Start with the owner's account only.

- verify Binance account and API eligibility in the intended deployment region;
- use testnet or paper trading before production;
- use API keys without withdrawal permission;
- enable IP restrictions where supported;
- validate balances, positions, open orders, fills and reconciliation;
- test network interruptions, exchange errors, rate limits and process restarts;
- enforce account-level loss, exposure and position limits;
- provide a tested emergency stop and safe recovery procedure;
- require explicit approval before enabling live trading.

No strategy or risk-parameter changes should be introduced as a side effect of deployment.

## 6. Multi-tenant product direction

The architecture should be multi-tenant-ready, but the first production validation should remain a single-account deployment.

Future tenant capabilities:

- tenant registration and account ownership verification;
- secure exchange account linking;
- separately encrypted API credentials;
- independent trading configurations and strategy permissions;
- account-level risk limits and emergency stops;
- isolated positions, orders, trade history and performance reports;
- tenant-scoped audit logs and access control;
- clear consent, risk disclosures and operational support processes.

A tenant must not be able to read or modify another tenant's credentials, settings, trading activity or records. Shared application code must not mean shared account state.

Before onboarding clients, assess applicable legal, regulatory, data-protection and exchange requirements in each operating jurisdiction. The platform must not assume that a client can connect an exchange account merely because an API is technically available.

## 7. Future client-facing frontend

A future frontend, potentially following the development approach used for `app.onyitechub.com`, can provide:

- registration, authentication and tenant workspaces;
- exchange account connection and status;
- strategy configuration and permissions;
- account-level risk settings;
- live operational status and alerts;
- positions, orders and trade history;
- performance dashboards and downloadable reports;
- audit and account-management controls.

The frontend should communicate with authenticated platform APIs. It should never receive or store raw exchange secrets in browser code. The trading engine should remain independently deployable from the frontend.

## 8. Trading performance research question

A key motivation for this project is the observed deterioration in trading performance when the market stopped trending.

This must be investigated with evidence rather than assumed to have one cause.

Research questions include:

- How does net expectancy differ between trending and sideways regimes?
- Which strategies continue to perform in each regime?
- Are entries being accepted when trend confirmation is weak or conflicting?
- Are regime classifications timely and based on live, correctly scaled data?
- Are position sizing and risk controls appropriate for the regime?
- How much of the loss is attributable to direction, exits, fees, slippage, DCA or scale-in behaviour?
- Do backtests remain robust on out-of-sample periods and different market conditions?

Potential changes such as stricter sideways-market filters, no-trade conditions or regime-specific sizing must be evaluated independently and approved before they are introduced.

## 9. Security and operational rules

- Never commit API keys, secrets, tokens, private keys or production account data.
- Rotate credentials that have appeared in archives, logs, screenshots or repositories.
- Disable withdrawal permissions on trading API keys.
- Apply least privilege and tenant isolation.
- Encrypt secrets at rest and in transit.
- Maintain audit trails for configuration changes and trading actions.
- Use paper trading and testnet before enabling live trading.
- Set hard loss/exposure limits and verify emergency-stop behaviour.
- Back up necessary state and test recovery.
- Confirm exchange terms, geographic access and applicable legal obligations before live or client trading.

## 10. Definition of success

Pillar AI's refactoring and deployment project will be considered successful when:

1. The original working capabilities have been inventoried and preserved.
2. The large `bot.py` has been progressively decomposed into clear, testable responsibilities.
3. Regression tests demonstrate that structural refactoring has not silently changed trading behaviour.
4. Strategies can be evaluated independently and together using realistic backtests.
5. Module health and trading performance can be monitored from recorded data.
6. The engine can be deployed and operated reliably on AWS Singapore.
7. The owner's Binance account can be tested safely and, after explicit approval, used for controlled live trading.
8. The architecture supports future tenant isolation and a separate client-facing frontend.

**Guiding principle:** Preserve what works, measure what happens, test what changes, and deploy only what has been validated.

---

## 11. Immediate next action

Begin Milestone 1 using the August 2026 archive as the functional reference:

1. Compare the archive with the current GitHub repository.
2. Map the complete `bot.py` method inventory and dependency graph.
3. Identify existing modules and their responsibilities.
4. Establish the baseline execution path and regression tests.
5. Present the audit findings for review before modifying trading logic.

**This roadmap records the agreed direction. It does not claim that refactoring, backtesting, AWS deployment, Binance production trading or multi-tenant onboarding have already been completed.**
