# Pillar AI Baseline Audit — 4 November 2026

**Audit type:** Initial static inventory and repository comparison  
**Functional reference:** `crypto-bot-august-2026.tar`  
**Target repository:** `onyi4677/pillar-trading-engine`  
**Target branch:** `main`  
**Status:** Initial findings recorded; no trading source code changed

## 1. Executive summary

The August 2026 archive contains the substantially developed Pillar AI codebase that should be used as the functional reference for this refactoring project.

The target repository `pillar-trading-engine` currently contains a README and project documentation, but no application source tree. The archive has therefore not yet been imported into this repository.

The public repository `cryptobotalert` contains a much smaller, older-looking prototype with a very different nested directory structure. Its `src/src/bot.py` is approximately 1 KB and should not be mistaken for the 444 KB `bot.py` in the August archive.

**Decision:** Do not refactor against the small legacy prototype. First preserve and baseline the August archive, compare it with any relevant later code or changes, and then bring the reviewed baseline into `pillar-trading-engine` without overwriting valid work.

## 2. Archive inventory

The supplied TAR archive was readable and its main bot file passed Python AST parsing.

| Item | Initial finding |
|---|---:|
| Archive entries | 192 |
| Files | 176 |
| Python source files | 95 |
| Main `src/bot.py` size | 444,008 bytes |
| Main `src/bot.py` length | 8,979 lines |
| Classes in `bot.py` | 8 |
| Methods in `CryptoTradingBot` | 154 |

The main class contains the trading orchestration, signal analysis, execution, risk, notifications, scheduling and position-management pathways. This is the primary refactoring target.

### Existing modules already present

The archive is not a completely monolithic application. It already contains useful modules that should be reused and inspected before new abstractions are introduced, including:

- `src/services/risk_manager.py`
- `src/services/binance_futures_service.py`
- `src/services/bitget_service.py`
- `src/services/signal_service.py`
- `src/services/telegram_service.py`
- `src/ml_models.py`
- `src/harmony_signal_engine.py`
- `src/scale_in_manager.py`
- `src/dca_manager.py`
- `src/market_regime.py`
- `src/config/settings.py`
- existing modules under `src/intelligence/`

These are candidates for dependency mapping and operational instrumentation, not automatic candidates for rewriting.

## 3. Initial findings inside `bot.py`

### 3.1 Large responsibilities and methods

The following methods are particularly large and should be examined carefully before any extraction:

| Method | Approximate lines |
|---|---:|
| `execute_trade()` | 770 |
| `generate_advanced_signal()` | 515 |
| `run_sync_trading()` | 397 |
| `__init__()` | 358 |
| `generate_base_signal()` | 230 |
| `run_enhanced_analysis()` | 192 |
| `run_single_analysis()` | 183 |
| `get_blockchain_validation()` | 173 |
| `_format_enhanced_signal_message()` | 173 |

These sizes indicate where to investigate complexity. They do not, by themselves, prove that the methods are redundant or defective.

### 3.2 Duplicate method definition

The class defines `setup_logging()` twice, at approximately lines 555 and 4565. In Python, the later class definition replaces the earlier method under the same name. We must compare both implementations and all call expectations before deciding which code to retain.

### 3.3 Duplicate module-level entry point

The module defines a top-level `main()` twice, at approximately lines 8803 and 8886. The second definition replaces the first name at module load time. The two startup flows need to be compared, including async service initialisation, thread startup, CLI options and shutdown behaviour.

This is a high-priority audit item because entry-point changes can affect the way the trading engine starts and stops.

### 3.4 Import and dependency structure

The main file imports from several package roots, including `config`, `services`, `src` and `intelligence`, and modifies `sys.path` at runtime. There are repeated imports and conditional imports for optional services.

Before extraction, we must establish the intended working directory, package layout, startup command and import contract. Moving methods without resolving these assumptions could create import errors or change which implementation is loaded.

## 4. Syntax and packaging observations

- The main `src/bot.py` passed AST parsing.
- Three auxiliary Python scripts in the archive failed AST parsing during the initial scan:
  - `update_risk_safe.py`
  - `src/fix_telegram_error_handling.py`
  - `src/intelligence/fixed_method.py`
- These findings apply to those auxiliary scripts only; they do not establish that the core bot is broken.
- No root-level `requirements.txt` was found in the archive during the initial inventory. Dependency manifests and deployment instructions need to be reconstructed from imports and known working environments, then tested.
- The archive contains generated bytecode, backup copies, logs and maintenance scripts. These should be classified rather than copied indiscriminately into the production repository.

## 5. Credential and repository security finding

The archive includes credential/configuration locations such as `.env`, `secret.env` and `src/config.json`, along with other key-related files. Their values were not reproduced in this audit.

Separately, the GitHub repository `onyi4677/cryptobotalert` is public and its tracked tree includes a root-level `secrets.json`. Its `.gitignore` contains an ignore rule for `config/secrets.json`, which does not match the root-level `secrets.json` path.

The contents of that tracked file were not inspected. Treat any credentials it may contain as potentially exposed until verified otherwise.

Required security actions before using credentials in any deployment:

1. Rotate active exchange/API keys and any other credentials that may have appeared in archives or repositories.
2. Remove sensitive files from the repository's current tree and, where necessary, clean the Git history.
3. Add correct ignore rules and safe example configuration files.
4. Use managed secret storage for deployment; do not commit secrets to Git.
5. Ensure Binance API keys have no withdrawal permission and use IP restrictions where supported.

Deleting a file from the latest commit alone does not erase previous Git history.

## 6. GitHub repository comparison

### Target: `onyi4677/pillar-trading-engine`

At the time of this audit, the `main` branch contains:

- `README.md`
- `docs/PILLAR_AI_DEVELOPMENT_ROADMAP.md`
- `docs/BASELINE_AUDIT_2026-11-04.md` (this report)

There is currently no `src/` directory in this repository.

### Other repositories inspected

- `onyi4677/pillar-ai-engine-july-2026`: README-only at the inspected default branch.
- `onyi4677/cryptobotalert`: contains an older/smaller nested prototype; it is not equivalent to the August archive's main bot implementation.

The current `pillar-trading-engine` repository is therefore the intended documentation and future source-of-truth repository, while the August archive remains the code baseline pending a careful import and comparison.

## 7. What has not been done

This is an initial static audit, not a complete behavioural audit.

- No source code has been changed or refactored.
- No method has been deleted or moved.
- No trading strategy or risk parameter has been changed.
- No live exchange connection or order has been made.
- No backtest or profitability claim has been produced.
- No archive contents have been uploaded to GitHub yet.
- No credential values have been published in this report.

## 8. Next audit tasks

1. Create a complete method inventory for all 154 `CryptoTradingBot` methods, with line ranges and preliminary responsibility categories.
2. Trace direct calls, attribute access, dynamic dispatch, callbacks, scheduled jobs and thread/async interactions.
3. Map each method to existing modules and identify duplicate responsibilities.
4. Inspect both `setup_logging()` implementations and both module-level `main()` implementations.
5. Trace the complete signal-to-order and open-position management lifecycle.
6. Establish a clean baseline test strategy before any method is moved.
7. Compare the archive against any later local or repository versions the owner identifies.
8. Prepare a reviewed import plan for `pillar-trading-engine`, excluding credentials, logs, bytecode, local state and unneeded backups.

**No refactoring should begin until the method/dependency map and baseline test plan have been reviewed.**

---

**Guiding principle:** The August backup is the working reference. Preserve its behaviour first; modularise second; optimise trading decisions only after measurable testing.
