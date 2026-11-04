# Pillar AI — Startup and Trading-Loop Trace

**Baseline:** `crypto-bot-august-2026.tar`  
**File:** `crypto-bot/src/bot.py`  
**Audit date:** 2026-11-04

## Confirmed executable startup path

The active executable path is controlled by the final guard at lines 8915–8979:

1. Parse CLI arguments and instantiate `CryptoTradingBot(test_mode=args.test)`.
2. Set `bot.running = True` and register SIGINT/SIGTERM handlers that set it to `False`.
3. Handle selected one-off CLI commands directly.
4. For normal operation, call `asyncio.run(main(bot, args))`.
5. The active module-level `main(bot_instance, args)` is the **second** definition (lines 8886–8912). It captures the running event loop, sets `bot.running = True`, awaits `bot.initialize_services()`, starts `bot.run_sync_trading` in a daemon thread, then awaits `bot.async_run()`.
6. `async_run()` calls `initialize_async_services()`, sets `self.running = True`, creates background tasks for regime monitoring, stale-signal reset, DCA pending checks, and scale-in checks, sends a startup Telegram message, and keeps the event loop alive while `self.running` is true.
7. On exit, `async_run()` attempts to save engine state and shuts down async services.

## Shadowed entry point

The earlier async `main()` at lines 8803–8883 is overwritten by the later `main(bot_instance, args)` definition before the executable guard runs. Its separate parser and calls to `bot.run_continuous()` do not define the active startup path. The active guard calls the later function with two arguments.

## Initialization observations requiring validation

- The active path calls `initialize_services()` before starting the trading thread.
- `async_run()` then calls `initialize_async_services()`. The latter may initialize Telegram if not already initialized, starts the Telegram service connection, and links it to the risk manager. This is a two-stage initialization design; confirm intended idempotency and resource lifecycle before changing it.
- `initialize_bot()` exists (lines 4599–4609), but the active startup path traced above does not call it directly. It is referenced by at least one other script/method path, so do not classify it as unused.
- The constructor invokes `self.setup_logging()`; because of the duplicate definitions, the later class method (lines 4565–4597) is the one bound when an instance is created.
- Top-level imports mix `config.*`, `services.*`, and `src.*` package paths. Runtime import behaviour therefore depends on the intended working directory and package layout; establish that contract before moving modules.

## CLI branch observations

The final guard accepts `--test`, `--single`, `--symbols`, `--test-notifications`, `--check-news`, `--conflict-stats`, `--risk-stats`, `--sentiment-stats`, `--test-auto-trading`, and `--blockchain-dashboard`.

- `--single` runs one analysis (and uses `--symbols` when supplied).
- `--test-notifications`, `--risk-stats`, `--test-auto-trading`, and `--blockchain-dashboard` invoke their corresponding handlers.
- In the current final guard, `--check-news` only prints “Checking news...” and exits; it does not invoke the news-check method.
- `--conflict-stats` and `--sentiment-stats` branches currently only set `bot.running = False`; they do not print the statistics.
- The parser and one-off branch logic in the shadowed first `main()` are not active through the final guard.

These are static code observations, not runtime test results.

## Trading thread: confirmed high-level path

`run_sync_trading()` is the synchronous trading worker started by the active `main()`. The inspected beginning of the loop shows it:

1. Iterates configured symbols and retrieves Bitget and Binance prices.
2. Updates blockchain price history from available prices.
3. Confirms prices when both venues are available and within a 1% difference; otherwise it can fall back to a single available venue.
4. Obtains a volatility state from `self.risk_manager` when available.
5. Checks crash-pause state and crash detection.
6. Fetches comprehensive market data.
7. Generates advanced signals for symbols.
8. Calls `assess_trade_risk` for each signal and skips blocked or invalid assessments before proceeding toward auto-trade checks.

The rest of the order path, including exact auto-trade gating, exchange submission, fills, and reconciliation, remains to be traced in the next audit pass.

## Refactoring implications

- Preserve this two-thread/two-loop architecture until its lifecycle and shared-state behaviour are understood.
- Do not merge, remove, or reorder initialization calls yet.
- Treat the duplicate first `main()` as shadowed code, but remove it only after checking any external import expectations and preserving CLI behaviour.
- Validate signal handling and graceful shutdown with tests before changing the thread/event-loop boundary.
- No source code has been changed or executed against a live exchange as part of this audit.
