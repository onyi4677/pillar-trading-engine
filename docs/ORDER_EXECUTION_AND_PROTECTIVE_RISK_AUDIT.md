# Pillar AI — Order Execution and Protective-Risk Path Audit

**Baseline:** `crypto-bot-august-2026.tar`  
**Files:** `src/bot.py`, `src/services/binance_futures_service.py`, `src/risk_manager_local.py`  
**Audit date:** 2026-11-04  
**Status:** Static inspection only. No live orders or code changes.

## 1. Active order path

The worker schedules `execute_trade(signal)` onto the main asyncio event loop. Within `execute_trade()`, the observed sequence is:

1. Harmony validation and conflict checks.
2. Crash, concurrent-position, daily-loss, and cooldown checks.
3. Optional second call to `assess_trade_risk()`.
4. Signal conflict resolution and on-chain sanity check.
5. Signal notification logic and signal-only-mode check.
6. Side selection and regime-adaptive settings.
7. DCA decision and quantity calculation.
8. Stop-loss / take-profit price calculation.
9. DCA first-leg execution through `dca_manager.start_dca()`, or normal execution through `binance.place_order()`.
10. On success, local position registration, metrics, and trade notifications.

## 2. Event-loop blocking risks

### 2.1 Synchronous notification wait from inside the event loop

At approximately lines 3065–3073, `execute_trade()` checks that `self.main_loop` is running, submits `send_enhanced_signal_notification()` to that loop using `asyncio.run_coroutine_threadsafe()`, and then calls `future.result(timeout=10)`.

In the active architecture, `execute_trade()` itself was submitted by `run_sync_trading()` to the same `self.main_loop`. Therefore, this code can block the event-loop thread while waiting for a coroutine that needs that same loop to run. The likely outcome is a timeout after up to 10 seconds, with other async tasks stalled during the wait.

**Validation:** test the notification path with an instrumented event loop and confirm the execution context/thread identity. Replace only after a regression test establishes the intended notification behaviour.

### 2.2 Synchronous Binance HTTP request inside async execution

At approximately line 3305, async `execute_trade()` calls synchronous `self.binance.place_order()`.

The inspected `BinanceFuturesService.place_order()` is a normal synchronous method. It calls `_make_request()`, then `_send_request()`, which uses a synchronous `requests.Session.request(..., timeout=10)`. Error branches also call `time.sleep()` (including 5 seconds for HTTP 429, 300 seconds for HTTP 418, and 2 seconds for HTTP 500).

Because the normal order path calls this synchronous method from the event loop, network latency and backoff sleeps can block Telegram, background tasks, and other coroutines. The DCA path also needs a separate inspection to determine whether its awaited methods call synchronous exchange operations internally.

**Validation:** fake the HTTP layer and measure event-loop responsiveness during success, timeout, 429, 418, and 500 cases. No real API requests are required.

## 3. Protective stop-loss and take-profit handling

### 3.1 The normal order request does not submit SL/TP parameters

The bot passes `stop_loss` and `take_profit` into `BinanceFuturesService.place_order()`. In the inspected method, the actual order request parameters contain symbol, side, type, quantity, timestamp, position side, and optional limit-price fields. The stop-loss and take-profit arguments are not added to the Binance request parameters.

On a successful entry, those values are copied into the returned `order_info` object and passed to `_track_position()`. The latter stores `current_stop` as local in-memory metadata; it does not place an exchange stop order.

### 3.2 LocalRiskManager is the initial SL/TP enforcement path

The bot also calls `LocalRiskManager.add_position()` after successful entry. The inspected local risk manager starts a daemon monitor thread. Every five seconds it checks local positions, obtains a current price using `binance.get_simple_price()`, and calls `_close_position()` when the local stop-loss or take-profit condition is reached. `_close_position()` sends a market close order through the Binance service.

This means the initial SL/TP enforcement visible in this code path is application-side monitoring and market closure, not an exchange-hosted protective order. If the process, host, network, or monitor thread is unavailable, this local monitor cannot execute the close. The Binance service has a separate trailing-stop path, but that does not establish that the initial SL/TP is exchange-hosted.

**Validation:** on Binance testnet or with a mocked exchange, inspect the actual open orders immediately after entry. Confirm whether stop-market and take-profit orders exist at the exchange, and test behaviour when the local monitor is stopped. Do not assume local position metadata proves exchange protection.

## 4. Regime and position-size observations

- `execute_trade()` computes `adaptive_min_confidence` from the regime detector, but the inspected use is logging only; no subsequent comparison against that variable was found in this method.
- Final normal-entry sizing is recalculated from a fixed target notional of $110, current price, DCA first-leg fraction, and regime position multiplier.
- The earlier worker's risk-adjusted `signal.quantity` is not directly read in `execute_trade()`. The final quantity therefore does not visibly inherit the risk manager's position multiplier from the worker. See `TRADING_WORKER_CONTROL_FLOW_FINDINGS.md`.
- `execute_trade()` also performs another risk assessment and stores `signal.risk_multiplier`, but no use of that field in its final position-size calculation was found in this method.

These are code-path observations, not conclusions about profitability or the causes of prior losses.

## 5. Priority test sequence

1. Verify execution coroutine thread identity and reproduce the notification wait.
2. Measure event-loop responsiveness while fake synchronous Binance requests are delayed or return rate-limit/server errors.
3. Verify exchange open orders after a test entry: are protective SL/TP orders present, or only local metadata?
4. Stop the local risk monitor in a test environment and verify the expected protection gap.
5. Vary the risk manager multiplier and regime multiplier independently; record the actual quantity passed to the exchange adapter.
6. Repeat the above with DCA enabled and disabled.

**No strategy or risk parameters were changed. No live exchange order was submitted.**
