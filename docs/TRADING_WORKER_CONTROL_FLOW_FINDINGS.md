# Pillar AI — Trading Worker Control-Flow Findings

**Baseline:** `crypto-bot-august-2026.tar`  
**File:** `crypto-bot/src/bot.py` · **Audit date:** 2026-11-04  
**Status:** Static audit observations; not runtime-confirmed. No source changes.

## Trading worker flow

The active main function starts `run_sync_trading()` in a daemon thread. The worker repeatedly obtains prices from Bitget and Binance, updates blockchain price history, checks price agreement, evaluates volatility/crash state, fetches comprehensive market data, generates advanced signals, applies `assess_trade_risk()`, and then evaluates additional trade gates. Orders are scheduled by submitting `execute_trade(signal)` to the captured asyncio event loop.

## Priority findings for controlled validation

### 1. Risk-adjusted quantity is not evidently used by the order-sizing path

In `run_sync_trading()` (approximately lines 8539–8740), the worker computes `adjusted_quantity = signal.quantity * risk_result['position_multiplier']`, temporarily assigns it to `signal.quantity`, schedules `execute_trade(signal)`, and immediately restores the original value.

AST inspection of `execute_trade()` finds no direct read of `signal.quantity`. Instead, that method independently calculates `position_size` from a fixed target notional (currently $110), current price, DCA first-leg fraction, and the market-regime position multiplier. It does not visibly apply `signal.risk_multiplier` or the risk manager's earlier `adjusted_quantity` to this final size.

Therefore, the earlier risk-manager position multiplier appears not to flow into the actual order quantity on this path. The temporary mutation/restoration is also timing-sensitive, but the more important finding is that the final order size is independently recomputed.

**Validation:** with fake exchange calls, vary only the risk manager's `position_multiplier` while keeping signal, price, and regime constant. Record the quantity passed to `binance.place_order()` and verify whether it changes. Do not place a live order.

### 2. Blockchain confidence can persist across signal iterations

Within the per-signal loop, `bc_conf_signal` is assigned only inside the condition that checks for `signal.indicators['blockchain_validation']`. Later, the momentum gate reads it using a `dir()` check. Since this is one function scope across loop iterations, a signal without blockchain validation can reuse the previous signal's local value.

**Validation:** process two synthetic signals in sequence: first with blockchain validation and a high confidence value, then without blockchain validation. Confirm the second signal uses a neutral/default value rather than the first signal's value.

### 3. Blockchain directional agreement variable is not assigned in this worker

The worker checks whether `blockchain_signal` exists in `dir()` and uses it to derive `bc_dir`. AST inspection finds no assignment to `blockchain_signal` within `run_sync_trading()`. Assignments elsewhere in the file are local to other functions. Consequently, the worker's directional agreement check appears to fall back to `neutral` rather than comparing the current technical signal with a blockchain direction.

**Validation:** trace where blockchain direction is produced in signal generation and confirm whether it is carried in `signal.indicators` or another field. Test the intended directional block with opposing technical and blockchain directions.

### 4. Daily trade counter and asynchronous completion need concurrency review

The worker checks `trades_today < max_trades` before scheduling execution. The performance counter is incremented in the future's completion callback, after the execution coroutine completes. Multiple signals can be processed while earlier trades are still pending. This may allow more than the configured daily maximum to be scheduled before the counter reflects those pending trades.

**Validation:** simulate several eligible signals in one cycle with a deliberately delayed fake execution coroutine. Verify the number of scheduled orders cannot exceed the configured cap. Inspect whether exchange position/open-order state independently prevents duplicates.

## Important distinction

These are static risks in control flow, not proof that losses were caused by these paths. They must be tested against the baseline before any correction. Any fix should be isolated, reviewed, and tested for decision equivalence except for the specific defect being corrected.

## Next audit operation

Trace the complete `execute_trade()` path: pre-order validation, quantity/precision handling, leverage, Binance API submission, protective stop/take-profit placement, response/error handling, position state updates, and notification. Then build controlled regression tests for the four findings above.

**No source code was modified, and no live exchange order was submitted.**
