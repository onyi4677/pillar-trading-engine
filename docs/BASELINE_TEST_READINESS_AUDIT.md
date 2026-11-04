# Pillar AI — Baseline Test and Dependency Readiness Audit

**Baseline:** `crypto-bot-august-2026.tar` · **Audit date:** 2026-11-04

## Test inventory

The archive contains 13 files with test-like names. They are mostly standalone scripts and integration probes rather than a structured regression suite:

- Root scripts include `test_real_algo.py`, `focused_phase2_test.py`, `test_conditional.py`, `phase2_complete_test.py`, `simple_telegram_test.py`, `test_algo.py`, `test_algo_raw.py`, and `production_ready_test.py`.
- Telegram-focused scripts include `src/test_telegram_call.py`, `src/test_signal_telegram.py`, `src/test_telegram_live.py`, `src/test_telegram_init.py`, and `src/test_telegram_simple.py`.
- Only a small subset expose conventional test-named functions; many are executable scripts with top-level test logic.
- Several scripts import the main `bot` module or exchange/network libraries directly. Do not execute them as a suite until their side effects, credential use, and network behaviour are reviewed.

No standard test configuration or dependency manifest was found in the archive root or searched paths: no `requirements*.txt`, `pyproject.toml`, `Pipfile`, `poetry.lock`, `setup.py`, `pytest.ini`, or `tox.ini` was identified.

## Consequence for safe refactoring

At present there is no demonstrated automated baseline that proves the August engine's decisions and execution requests remain equivalent after code movement. Refactoring before building this baseline would make regressions difficult to detect.

## Recommended baseline test layers

### Layer A — Import and construction tests
- Verify the intended working directory and package imports.
- Verify configuration loading with fake credentials and network clients.
- Construct the bot without starting background threads or making exchange calls.
- Verify startup/shutdown lifecycle and initialization idempotency.

### Layer B — Pure decision tests
- Regime classification and thresholds.
- Indicator and feature calculations.
- Base and advanced signal generation.
- Signal conflict resolution and Harmony validation.
- Risk assessment and position multiplier decisions.
- Stop-loss/take-profit price calculation and rounding.

### Layer C — Execution contract tests
Use a fake Binance adapter and deterministic fixtures to assert:
- Exact side, symbol, quantity, position side, order type, and reduce-only behaviour.
- DCA first-leg and subsequent-leg requests.
- Protective stop/take-profit order expectations.
- Exchange error, timeout, rate-limit, retry, and duplicate-order behaviour.
- Local position registration and exchange reconciliation.
- No real credentials and no live exchange calls.

### Layer D — Concurrency and lifecycle tests
- Event-loop responsiveness while execution is in progress.
- No blocking `future.result()` call from the event-loop thread.
- No synchronous HTTP or sleep blocking async tasks.
- Thread-safe shutdown and task cancellation.
- No duplicate initialization or orphaned background threads.

### Layer E — Golden-master decision comparison
Before extracting code, capture deterministic representative inputs and outputs from the August baseline. Compare the original and refactored versions for:
- Signal type and confidence.
- Regime and indicator values.
- Risk approval/rejection and position multiplier.
- Calculated SL/TP and final quantity.
- Exchange request payload (against a fake adapter).

The first refactor must preserve outputs for the same inputs. Strategy improvements should be a separate later experiment.

## Dependency manifest work

Before deployment, derive dependencies from imports and known runtime requirements, then create a pinned, reviewed dependency manifest. Do not generate a requirements file by blindly installing every package found in the development environment. Separate runtime, development/test, and optional integrations. Remove unused dependencies only after import and runtime verification.

## Safety

- Do not run the archived ad-hoc scripts against real exchange credentials.
- Use fake exchange and Telegram clients for the first regression suite.
- Do not include `.env`, `secret.env`, config files containing tokens, or API keys in Git.
- No tests were executed and no source code was changed during this audit.
