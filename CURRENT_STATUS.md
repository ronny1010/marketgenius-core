# CURRENT_STATUS

Updated: 2026-08-18

## Was any broker order submitted?

**No.** Zero orders were submitted to any broker, paper or live. No broker endpoint was
contacted at all — the sandbox that produced this code has no network route to Alpaca and
no credentials. Every order in the test suite went to the in-process simulator.

## Completed

- `packages/domain` — enums, Decimal money primitives, canonical entities
- `packages/brokers/contract.py` — adapter protocol, error taxonomy, three-valued lookup
- `packages/brokers/simulator.py` — deterministic fake broker with 10 submit faults,
  5 lookup faults, venue-side idempotency, both fee models
- `packages/risk_engine` — checksummed RiskConstitution + deterministic engine
- `packages/order_coordinator` — coordinator (sole order writer) + recovery worker
- `packages/persistence` — repository protocol, in-memory impl with write-fault injection
- `packages/market_data/bars.py` — completed-bar guard, dedup, gap detection
- `packages/strategies/breakout.py` — one versioned deterministic strategy
- `tests/` — 59 tests

## Commands that pass

```
python3 -m pytest -q     ->  59 passed in 0.10s
```

Includes a 100-combination fault matrix (every submit fault x every lookup fault x
write-failure on/off), asserting zero duplicate orders and zero phantom SUBMITTED states.

## Not built

Everything else. Specifically: no Alpaca adapter, no FastAPI app, no Postgres/Alembic,
no Redis/worker, no Next.js dashboard, no Docker Compose, no news pipeline, no
opportunity ranking, no protective-exit monitor, no OANDA/Coinbase/Polymarket.

## Open defects / known limitations

1. `InMemoryRepository` has no locking. Postgres migration must add
   `SELECT ... FOR UPDATE` (or a unique index on `client_order_id`) or two workers can
   race the same intent. The single-order invariant currently rests on single-process
   execution plus venue-side idempotency.
2. `_force_persist` writes through to the in-memory dict when the durable write fails.
   Against Postgres this must become an append-only incident journal on separate
   infrastructure — otherwise a database outage loses the recovery record.
3. Risk engine reads exposure from passed-in `RiskContext`; nothing yet *guarantees* that
   context was freshly fetched. Milestone 1 must make freshness a checked precondition.
4. Only one strategy exists; the other seven and the ranking layer are unwritten.
5. No protective-exit monitor. Nothing would close a position today.

## Next action

Milestone 1, in this order: Postgres schema + Alembic → SQLAlchemy repository behind the
existing `Repository` protocol (with row locking) → Alpaca **read-only** adapter against
the paper endpoint → verify the real adapter satisfies the same contract tests as the
simulator → scheduler → minimal dashboard.

Order submission stays disabled by configuration until the coordinator has been exercised
against real Alpaca paper responses.
