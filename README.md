# copy-engine-otmo Architecture

This document explains the current copy engine implementation, data flow, and how GMX and Ostium are integrated. All references are grounded in code under `copy-engine-otmo/`.

## 1) High-level overview

The copy engine ingests OTMO trade events, normalizes them, applies eligibility and rule gates, then routes execution to a venue adapter (GMX or Ostium). It persists idempotency and trade mappings in a small database and exposes a lightweight status API.

```
OTMO Trade Events (JSON file)
        |
        v
   FileEventSource
        |
        v
   processTradeEvent
   - idempotency
   - rule checks
   - eligibility gate
        |
        v
   ExecutionRouter
     |        |
     v        v
   GMX     Ostium
  Adapter   Adapter
        |
        v
  Persist processed_events + trade_mappings
        |
        v
      API (/health, /metrics, /events)
```

References:
- Worker: `apps/worker/src/index.ts`, `apps/worker/src/worker.ts`
- Pipeline: `packages/core/src/pipeline.ts`
- Router: `packages/core/src/router.ts`
- Adapters: `packages/adapters/gmx/src/index.ts`, `packages/adapters/ostium/src/index.ts`
- Repo interfaces: `packages/shared/src/types.ts`

## 2) Repo map (copy-engine-otmo)

Top-level structure:

- `apps/worker/`: main copy engine runner (polls OTMO events, processes them).
- `apps/api/`: simple Fastify API for health + metrics.
- `packages/shared/`: types, config loader, logger, utilities.
- `packages/core/`: normalization, eligibility, routing, pipeline.
- `packages/adapters/gmx/`: GMX adapter using SDK + ticker endpoint.
- `packages/adapters/ostium/`: Ostium adapter using REST client + price feed.
- `packages/db/`: repository implementations (memory, sqlite, postgres).
- `data/events.sample.json`: sample OTMO event payloads.

## 3) Core data model (shared types)

Defined in `packages/shared/src/types.ts`:

- `SourceTradeEvent`
  - Raw OTMO event format (id, traderId, type, symbol, positionSide, sizeUsd, price, leverage, tp/sl, timestamp).
- `NormalizedTradeEvent`
  - Adds venue + idempotency key and normalizes missing fields.
- `ExecutionIntent`
  - Intent types: `OPEN`, `INCREASE`, `DECREASE`, `CLOSE`, `UPDATE_TP_SL`, `CANCEL_ORDER`.
- `ExecutionResult`
  - Status: `SUCCESS`, `FAILED`, `SKIPPED` plus venue ids and errors.
- `CopyRuleConfig`, `EligibilityContext`
  - Rule gating and eligibility thresholds.
- `CopyRepository`
  - Idempotency + mapping storage interface.

## 4) Pipeline and rules (core)

Implemented in `packages/core/src/pipeline.ts`:

1. Normalize event
   - `normalizeTradeEvent` (`packages/core/src/normalize.ts`)
   - Adds `idempotencyKey = sourceTradeId:type:venue`.
2. Idempotency check
   - `repository.hasProcessedEvent(key)` skips duplicates.
3. Rule gates
   - allowlist markets
   - max size
   - max leverage
4. Eligibility gate
   - `evaluateEligibility` (`packages/core/src/eligibility.ts`) checks ROI and flags.
5. Build intent + execute
   - `ExecutionRouter.execute` (`packages/core/src/router.ts`)
6. Persist
   - `processed_events` record (success/failure/skip)
   - `trade_mappings` upsert for venue order/position ids

## 5) Worker runtime

Defined in `apps/worker/src/index.ts`:

- Loads config (`packages/shared/src/config.ts`).
- Creates repository via `createRepository`.
- Instantiates `GmxAdapter` + `OstiumAdapter`.
- Uses `FileEventSource` with `OTMO_EVENTS_PATH`.
- Runs `CopyWorker` loop with `pollIntervalMs`.

Event ingestion:
- `FileEventSource` reads JSON array from `OTMO_EVENTS_PATH`.
- Fallback to `*.sample.json` and relative paths if primary path is missing.
- Keeps `lastIndex` to process only new events.

References:
- `apps/worker/src/event-source.ts`
- `apps/worker/src/worker.ts`

## 6) Persistence and idempotency

Repository interface: `packages/shared/src/types.ts`

Storage:
- SQLite (default): `packages/db/src/sqlite.ts`
- Postgres: `packages/db/src/postgres.ts`
- Memory: `packages/db/src/memory.ts`

Tables (SQLite/Postgres):
- `processed_events`
  - `idempotency_key`, `source_trade_id`, `status`, `error`, `created_at`
- `trade_mappings`
  - `source_trade_id`, `venue`, `venue_order_id`, `venue_position_id`, `last_intent_type`, `updated_at`

## 7) GMX integration

Files:
- `packages/adapters/gmx/src/index.ts`
- `packages/adapters/gmx/src/smoke.ts`

What is called:
- SDK markets (best-effort):
  - `fetchGmxMarkets` attempts `sdk.getMarkets` or `sdk.markets.getMarkets`.
  - Uses config: `GMX_CHAIN_ID`, `GMX_RPC_URL`, `GMX_ORACLE_URL`, `GMX_SUBSQUID_URL`.
  - If SDK signature is unknown, logs a warning and returns empty list.
- Ticker API (price snapshot):
  - `fetchGmxTickers` calls `GMX_TICKER_URL`.
  - Default: `https://arbitrum-api.gmxinfra.io/prices/tickers`.
  - `pickGmxTicker` filters tickers by symbol (e.g., `BTC-USD` -> `BTC`).

Execution:
- `GmxAdapter` supports `openPosition`, `closePosition`, `updateTpSl`, `cancelOrder`.
- If `GMX_DRY_RUN=true`, the adapter logs intent + ticker snapshot and returns `SUCCESS`.
- If `GMX_DRY_RUN=false`, methods return `FAILED` with `gmx_*_not_implemented`.

TODO / Not Implemented Yet:
- Real GMX order execution using `@gmx-io/sdk`.

## 8) Ostium integration

Files:
- `packages/adapters/ostium/src/index.ts`
- `packages/adapters/ostium/src/client.ts`

What is called:
- Price snapshot:
  - `fetchOstiumPrices` calls `OSTIUM_PRICE_URL`.
  - Default: `https://metadata-backend.ostium.io/PricePublish/latest-prices`.
  - `pickOstiumPrice` selects by `from/to` symbol (e.g., BTC/USD).
- REST client:
  - `OstiumClient` uses `OSTIUM_API_BASE_URL`.
  - Default paths:
    - `GET /markets`
    - `POST /orders`
    - `GET /orders/:id`
    - `POST /positions/:id/close`
    - `POST /orders/:id/cancel`
  - Retries with backoff; supports `Authorization: Bearer` if `OSTIUM_API_KEY` provided.

Execution:
- `OstiumAdapter` supports `openPosition`, `closePosition`, `updateTpSl`, `cancelOrder`.
- If `OSTIUM_DRY_RUN=true`, logs intent + price snapshot and returns `SUCCESS`.
- If `OSTIUM_DRY_RUN=false`, calls `OstiumClient` REST endpoints.

TODO / Not Implemented Yet:
- Confirm Ostium API contract for payload shape and endpoint paths.

## 9) API service (status/metrics)

File: `apps/api/src/index.ts`

Endpoints:
- `GET /health`: `{ status, env, timestamp }`
- `GET /metrics`: processed events stats
- `GET /events?limit=25`: latest processed events (descending by time)

## 10) Configuration (env-driven)

Loaded via `packages/shared/src/config.ts`:

Worker:
- `OTMO_EVENTS_PATH` (default `./data/events.sample.json`)
- `WORKER_POLL_INTERVAL_MS`
- `VENUE_DEFAULT`
- `GMX_DRY_RUN`, `OSTIUM_DRY_RUN`

Eligibility:
- `ELIGIBILITY_MIN_ROI`
- `ELIGIBILITY_REQUIRE_CONSISTENCY`
- `ELIGIBILITY_DISALLOW_HEDGE`

DB:
- `DB_DIALECT` (`sqlite` default)
- `DATABASE_URL` (postgres)
- `SQLITE_PATH`

GMX:
- `GMX_CHAIN_ID`
- `GMX_RPC_URL`
- `GMX_ORACLE_URL`
- `GMX_SUBSQUID_URL`
- `GMX_TICKER_URL`
- `GMX_WALLET_PRIVATE_KEY`

Ostium:
- `OSTIUM_API_BASE_URL`
- `OSTIUM_PRICE_URL`
- `OSTIUM_API_KEY`
- `OSTIUM_WALLET_PRIVATE_KEY`

API:
- `API_PORT`

## 11) Observability

- Logging via `pino` (`packages/shared/src/logger.ts`).
- Worker logs each processed event (status + error).
- API logs startup or failure.


