# Implementation Plan

Sequencing rule: nothing that can place an order ships before the thing that can recover
one. Milestones are gated on runtime evidence, not on unit tests passing.

## Milestone 0 — Execution core (DONE, this deliverable)

Domain, broker contract, simulator, risk engine, order coordinator, recovery worker,
bar hygiene, one strategy, 59 tests including a 100-case fault matrix.
Exit criterion met: no fault combination yields a duplicate or phantom order.

## Milestone 1 — Infrastructure + Alpaca read-only

- Docker Compose: Postgres, Redis, API, worker, web
- Alembic schema for every canonical entity; `client_order_id` unique per environment
- SQLAlchemy repository behind the existing `Repository` protocol, with row locking
- Alpaca adapter: account, instruments, clock, quotes, completed bars, positions,
  open orders, lookup-by-client-id. **No submit path enabled.**
- Contract test suite run against BOTH simulator and real Alpaca paper (read-only calls)
- Remaining seven strategies + opportunity ranking
- Structured JSON logging, UTC internally
- Minimal operator dashboard (status, financials, positions, orders, opportunities,
  decisions, incidents)

Exit criterion: real Alpaca paper quotes and bars persisted with provenance; ranked
opportunities rendered; zero orders submitted.

## Milestone 2 — Alpaca paper lifecycle

Enable submission against paper only. Full lifecycle verified against broker paper truth:
submit → fill → position → reconcile → exit. Restart recovery exercised against the real
paper endpoint by killing the process mid-submission. Protective-exit monitor with a lag
ceiling; breaching the ceiling blocks entries and raises an infrastructure incident rather
than pretending the position is protected.

Exit criterion: local ledger reconciles to Alpaca paper truth to the cent, across a
process kill.

## Milestone 3 — Alpaca live (gated)

Explicit environment configuration, endpoint allowlist, restricted keys with no withdrawal
capability. Constitution defaults stay at $100 authorized / $20 position / $40 exposure.
Requires human approval; never enabled during development or tests.

## Milestone 4–6 — OANDA, then Coinbase, then Polymarket

Each reuses the same contract, coordinator and constitution. Practice/read-only validation
first. Polymarket only after jurisdiction and account eligibility are confirmed; wallet
signing isolated from the application; no key in the database or logs; heartbeat fail-safe
cancellation.

## Milestone 7 — Cross-market

Unified ranking, correlation-aware aggregate exposure, production dashboard.
