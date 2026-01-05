# Polymarket Arbitrage Bot

**Automated dump-and-hedge trading for Polymarket’s 15-minute crypto Up/Down markets** — TypeScript, official CLOB client, hands-free across **BTC, ETH, SOL, and XRP**.

[![Node.js](https://img.shields.io/badge/node-%3E%3D16-brightgreen)](https://nodejs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.3-blue)](https://www.typescriptlang.org/)

## Contents

- [Why this exists](#why-this-exists)
- [What it does](#what-it-does)
- [Strategy flow](#strategy-flow)
- [Tech stack](#tech-stack)
- [Quick start](#quick-start)
- [Configuration](#configuration-env)
- [Project layout](#project-layout)
- [Disclaimer](#disclaimer)
- [Operational safety](#operational-safety-checklist)
- [Contributors](#contributors)
- [Contributing](#contributing)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## Why this exists

Short-dated prediction markets move fast. When one side **dumps** in seconds, the other side often lags — and if you can buy both legs cheaply enough, their combined cost can sit **below $1 per paired share**, locking in a structural edge before resolution.

This bot **watches** those markets continuously, **detects** sharp moves that match your thresholds, **executes** a two-leg cycle (dump capture → hedge), and **tracks** P&L — with optional **simulation** so you can validate behavior before risking capital.

---

## What it does

| Capability | Description |
|------------|-------------|
| **Multi-asset** | Trade one or many markets: `btc`, `eth`, `sol`, `xrp` (comma-separated). |
| **Auto-discovery** | Resolves the active **15m Up/Down** market per asset from Polymarket’s Gamma API and rolls forward each new period. |
| **Dump detection** | Uses recent ask history to flag a leg when price falls by your **move threshold** within a short time window. |
| **Hedge logic** | After leg 1, waits for leg 2 when **leg1 entry + opposite ask ≤ your sum target** (e.g. 0.95). |
| **Risk controls** | Configurable **stop-loss hedge** if the favorable hedge does not appear within **N minutes**. |
| **Settlement** | On market close, reconciles winners/losers and can **redeem** winning positions on-chain (production). |
| **Logging** | Streams activity to stderr and appends a **history** file for review and auditing. |

---

## Strategy flow

```text
New 15m round
     │
     ▼
┌─────────────────┐     rapid drop on Up or Down     ┌──────────────┐
│ Watch window    │ ───────────────────────────────► │ Buy dumped   │
│ (first N min)   │                                  │ leg (Leg 1)  │
└─────────────────┘                                  └──────┬───────┘
                                                            │
                              opposite ask cheap enough     │
                              (sum ≤ target)                ▼
                                                    ┌──────────────┐
                                                    │ Buy hedge    │
                                                    │ (Leg 2)      │
                                                    └──────┬───────┘
                                                           │
                    timeout? ─────────────────────────────┤
                                                           ▼
                                                Stop-loss hedge path
```

*Simplified view of the trader module; tune thresholds with environment variables.*

---

## Tech stack

- **Runtime:** Node.js 16+
- **Language:** TypeScript
- **Polymarket:** [`@polymarket/clob-client`](https://www.npmjs.com/package/@polymarket/clob-client) (orders, auth), Gamma + CLOB HTTP APIs (markets, prices)
- **Chain:** Polygon (USDC, CTF redemption flow)

---

## Quick start

### Prerequisites

- [Node.js](https://nodejs.org/) 16 or newer
- A Polymarket-compatible wallet and (for live trading) USDC on **Polygon**, plus a clear choice of **EOA vs proxy** signing (`SIGNATURE_TYPE` in `.env`)

### Install

```bash
git clone https://github.com/jaunita72f/polymarket-arbitrage-bot.git
cd polymarket-arbitrage-bot
npm install
cp .env.example .env
# Edit .env — see [Configuration](#configuration-env)
npm run build
```

### Run modes

| Command | Purpose |
|---------|---------|
| `npm run dev` | Run from source with `ts-node` (development). |
| `npm run typecheck` | TypeScript check without emitting `dist/`. |
| `npm run clean` | Remove compiled output in `dist/`. |
| `npm run sim` | **Simulation** — logs trades, no real orders (`--simulation`). |
| `npm run prod` | **Production** — real CLOB orders (`--production`). |
| `npm start` | Run `dist/main.js`; combine with your flags and `.env` as below. |

**Live trading:** set `PRODUCTION=true` in `.env` and run `npm run prod` (or `node dist/main.js --production`) so the process is not left in simulation mode.

---

## Configuration (`.env`)

Copy `.env.example` to `.env` and adjust.

| Variable | Role |
|----------|------|
| `PRIVATE_KEY` | Required for real orders and redemption. |
| `PROXY_WALLET_ADDRESS` | Polymarket proxy/profile address if applicable. |
| `SIGNATURE_TYPE` | `0` EOA, `1` Proxy, `2` Gnosis Safe (default in example: `2`). |
| `GAMMA_API_URL` | Optional; default `https://gamma-api.polymarket.com`. |
| `CLOB_API_URL` | Optional; default `https://clob.polymarket.com`. |
| `API_KEY` / `API_SECRET` / `API_PASSPHRASE` | Optional; use if your setup requires CLOB API credentials. |
| `MARKETS` | e.g. `btc` or `btc,eth,sol,xrp` (comma-separated, lowercase). |
| `CHECK_INTERVAL_MS` | Price polling interval in ms (default `1000`). |
| `MARKET_CLOSURE_CHECK_INTERVAL_SECONDS` | How often to check for round closure (default `20`). |
| `DUMP_HEDGE_SHARES` | Size per leg (shares). |
| `DUMP_HEDGE_SUM_TARGET` | Max combined price for hedge (e.g. `0.95`). |
| `DUMP_HEDGE_MOVE_THRESHOLD` | Min fractional drop to count as a dump (e.g. `0.15` = 15%). |
| `DUMP_HEDGE_WINDOW_MINUTES` | Only look for dumps in the first *N* minutes of the round. |
| `DUMP_HEDGE_STOP_LOSS_MAX_WAIT_MINUTES` | Force the stop-loss path if no favorable hedge within *N* minutes. |
| `DUMP_HEDGE_STOP_LOSS_PERCENTAGE` | Cap on the forced hedge sum: max(leg1 + opposite) = 1 + this fraction (default `0.2`). |
| `PRODUCTION` | `false` = simulation; use `true` with `--production` for live execution. |

---

## Project layout

```text
src/
  main.ts            # Entry: discovery, monitors, period rollover
  monitor.ts         # Price polling and snapshots
  dumpHedgeTrader.ts # Dump → hedge → stop-loss → settlement tracking
  api.ts             # Gamma, CLOB, orders, redemption, activity
  config.ts          # Environment loading
  models.ts          # Shared types
  logger.ts          # History file + stderr
```

---

## Disclaimer

This software is provided **for educational and research purposes only**. Prediction markets and automated trading involve **substantial financial risk**, including possible **total loss**. Past or simulated behavior does **not** guarantee future results. You are solely responsible for compliance with applicable laws, exchange terms, and tax obligations. **Nothing here is investment, legal, or tax advice.**

---

## Operational safety checklist

- Run in simulation first and inspect `history.toml` over multiple rounds.
- Start with low `DUMP_HEDGE_SHARES` and conservative thresholds in production.
- Keep private keys out of version control; rotate any exposed credentials.

---

## Contributors

Core development (Q1 2026): [buffalu](https://github.com/buffalu), [bunghi](https://github.com/bunghi), [calvinzhou-rockx](https://github.com/calvinzhou-rockx), [Carlosted](https://github.com/Carlosted), [cavemanloverboy](https://github.com/cavemanloverboy), [ch9xy](https://github.com/ch9xy), [ChewingGlass](https://github.com/ChewingGlass), [codemonkey6969](https://github.com/codemonkey6969), [cryptogosu](https://github.com/cryptogosu), [cryptopapi997](https://github.com/cryptopapi997), [danpaul000](https://github.com/danpaul000).

---

## Contributing

Issues and pull requests are welcome: strategy ideas, safer defaults, tests, and clearer documentation help everyone.  
See [CONTRIBUTING.md](CONTRIBUTING.md) for the local workflow and commit guidelines.

---

## Troubleshooting

- **`Failed to fetch market/orderbook`:** Often transient API or network issues; confirm endpoints and retry behavior.
- **Order failures in production:** Verify `PRIVATE_KEY`, `SIGNATURE_TYPE`, and proxy wallet settings match your Polymarket account.
- **No market found:** Ensure `MARKETS` only lists supported assets (`btc`, `eth`, `sol`, `xrp`) and wait for the next 15-minute round.

---

## License

See [LICENSE](LICENSE) in this repository for terms.
