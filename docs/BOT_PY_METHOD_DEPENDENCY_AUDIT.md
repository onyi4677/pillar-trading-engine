# Pillar AI — Detailed `src/bot.py` Audit (Stage 1B)

**Baseline:** `crypto-bot-august-2026.tar`  
**Audit date:** 2026-11-04

## Scope

The archive's `crypto-bot/src/bot.py` contains 8,979 lines (444,008 bytes). Python AST parsing succeeds. The `CryptoTradingBot` class contains **154 methods**: 25 asynchronous and 129 synchronous. A complete local method index was generated with inclusive line ranges, method sizes, provisional responsibility labels, and statically visible internal `self.method(...)` call sites.

Responsibility labels are provisional name-based groupings, not verified runtime boundaries. Static call counts are navigation aids only. A zero count does not establish that a method is unused: external callers, callbacks, decorators, dynamic dispatch, reflection, `getattr`, imported aliases, and scheduler registration can be missed.

## Confirmed structural findings

1. **Duplicate class method:** `setup_logging` is defined at lines 555–584 and again at 4565–4597. The later definition overrides the earlier definition in the class namespace.
2. **Duplicate module entry point:** async `main()` is defined at lines 8803–8883 and again at 8886–8912. The second definition replaces the first in the module namespace. The actual executable guard and startup route still need tracing.
3. **Largest methods:**

| Method | Inclusive lines | Size |
|---|---:|---:|
| `execute_trade` | 2646–3415 | 770 |
| `generate_advanced_signal` | 6441–6955 | 515 |
| `run_sync_trading` | 8389–8785 | 397 |
| `__init__` | 196–553 | 358 |
| `generate_base_signal` | 5160–5389 | 230 |
| `run_enhanced_analysis` | 4684 onward | 192 |
| `run_single_analysis` | 7038–7220 | 183 |
| `get_blockchain_validation` | 1179–1351 | 173 |
| `_format_enhanced_signal_message` | 3483–3655 | 173 |

4. **Auxiliary syntax issues:** three other Python files in the archive fail AST parsing: `update_risk_safe.py` (line 139), `src/fix_telegram_error_handling.py` (line 94), and `src/intelligence/fixed_method.py` (line 1). These do not affect the finding that `src/bot.py` itself parses successfully.
5. **External references:** a static scan found references from other archive scripts to `execute_trade`, `detect_signal_conflict`, `initialize_bot`, and `send_signal_notification`. This is not a complete call graph.

## Provisional responsibility groups

The 154 methods were grouped into these working categories for the next dependency pass:

- Startup and configuration
- Market data and intelligence
- Indicators and signal generation
- Strategy validation and decision-making
- Risk and position management
- Trade execution and order management
- Notifications and reporting
- Scheduling and monitoring
- Shared or not-yet-classified utilities

The full per-method index is retained as a separate audit artifact. Categories must be validated against method bodies and runtime call paths before module extraction.

## Next audit operations

1. Trace both `main()` definitions and the final `if __name__ == "__main__"` guard to identify the real startup entry point.
2. Trace the active cycle end-to-end: market data → regime/features → signal generation → validation/conflict resolution → risk checks → exchange order → fill/position reconciliation → notifications.
3. Map `__init__` state: attributes created, read, mutated, and passed to existing services.
4. Inspect the largest methods and map their dependencies before extracting any code.
5. Search dynamic dispatch, scheduler jobs, callbacks, imported aliases, CLI scripts, and tests before making any unused-code classification.
6. Preserve the August baseline and current behaviour. No strategy, risk parameter, or order-execution behaviour is to be changed as part of structural refactoring without explicit approval.

**Status:** Audit only. No source code has been modified, extracted, or deployed.
