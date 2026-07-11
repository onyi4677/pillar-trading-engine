# pillar-trading-engine
What I solved in MY Trading Bot
# Pillar AI Trading Engine v5.1

A fully automated cryptocurrency trading bot with **multi‑pillar decision fusion**, real‑time on‑chain intelligence, and dynamic risk management.  
Built to trade on Binance Futures with $6 positions at 10x leverage.

## Architecture

The bot combines **five independent analytical pillars** in a unified council:

- **Technical Analysis (LSTM + XGBoost + indicators)** – 60% weight  
- **Blockchain On‑Chain (Codex, Moralis, Etherscan)** – 40% weight  
- **Kalman Filter** – noise‑reduced price prediction  
- **Fourier Transform** – cycle detection  
- **Market Sentiment** – Fear & Greed, social volume, news

These pillars vote on direction and confidence. A trade only executes when all guardians approve.

## Key Features

### Intelligent Trade Filtering (Guardians)
- **Direction Agreement Gate** – Blocks any trade where TA and blockchain disagree.
- **Trend Gate** – Trades only when ADX > 25 and price is aligned with the 50‑EMA (avoids chop).
- **Momentum Gate** – Prevents entries when momentum is misaligned and blockchain confidence is low.
- **Codex Pressure Block** – Blocks buy signals during heavy on‑chain selling (and vice‑versa).
- **Harmony Score** – Measures agreement across pillars; minimum 60% confidence required.

### Automated Position Management
- **Dynamic Stop‑Loss & Take‑Profit** – Tightens with ML confidence.
- **Trailing Stop** – Activates after +2% profit.
- **Scale‑In Manager** – Adds to winning positions at predefined dips, with Codex approval.
- **Exchange‑Synced Risk Manager** – Reconciles local state with Binance every 60 s, catching manual trades or bot‑missed fills.

### On‑Chain Intelligence Pipeline
- Real‑time exchange reserves (Binance & Coinbase) for BTC, ETH, SOL, XRP, AVAX, LINK.
- Whale transaction tracking via Etherscan V2 API.
- Codex GraphQL integration for buyer/seller pressure.
- Free tier APIs (CoinGecko, mempool.space, Fear & Greed) with aggressive caching to stay within limits.

### Multi‑Tenant Architecture
- Isolated instances with separate API keys, data directories, and Telegram alerts.
- Shared codebase, updated via a single source.

## Tech Stack
- **Python 3.10**, **asyncio**
- **TensorFlow/Keras** + **ONNX** (LSTM inference)
- **XGBoost**
- **TA‑Lib**, **pandas**, **NumPy**
- **CCXT** (exchange connectivity)
- **python‑telegram‑bot** (alerts)
- **Systemd** for service management on Ubuntu/Vultr

## What I Solved (Engineering Challenges)
- Fixed a momentum gate that was blocking 100% of trades due to a phantom BC confidence variable.
- Resolved an exchange‑sync bug that lost position updates after scale‑ins – implemented a periodic reconciliation daemon.
- Migrated Etherscan from deprecated V1 to V2 API after a sudden breaking change.
- Removed a blockchain‑leadership override that was forcing trades against technical signals, restoring profitability.
- Designed a multi‑layer filter system to survive choppy markets (Fourier + ADX + Moving Average).

## Running the Bot
1. Clone the repo.
2. Install dependencies: `pip install -r requirements.txt`
3. Configure `secret.env` with your Binance API keys.
4. Start: `python src/bot.py` (or use the systemd service file).

## Disclaimer
This bot is experimental. Past performance does not guarantee future results. Use at your own risk.

## Contact
I’m open to remote roles in blockchain engineering, quantitative development, or AI‑driven trading systems.  
[https://www.linkedin.com/in/onyemaechi-w-ab1755420/] | [onyi4677@gmail.com]
