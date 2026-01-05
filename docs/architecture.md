# Architecture Notes

## Runtime flow

1. `main.ts` loads config and starts one monitor loop per configured market.
2. `monitor.ts` resolves token IDs and emits snapshots with current prices.
3. `dumpHedgeTrader.ts` consumes snapshots and manages strategy state transitions.
4. `api.ts` handles Gamma/CLOB data access, order submission, and redemption.

## State model

- `marketStates`: per-condition in-memory phase and price history.
- `trades`: per-round trade legs used for closure checks and realized PnL updates.

## Operational assumptions

- Round cadence is fixed at 15 minutes (`ROUND_DURATION_SECONDS`).
- Price discovery relies on CLOB endpoints and can be temporarily unavailable.
- Production mode requires valid wallet credentials and compatible signature mode.
