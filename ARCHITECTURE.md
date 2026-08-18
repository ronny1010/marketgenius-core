# MarketGenius — Architecture

**No claim of profitability is made anywhere in this system.** Signal strength is a
rule-alignment score, not a probability of profit.

## Why this shape

The previous system failed for four reasons, and each one has a structural answer here:

| Previous failure | Structural answer |
|---|---|
| Duplicated state | One durable reservation per intent; broker truth wins on every conflict |
| Contradictory safety gates | One `RiskConstitution`, immutable, checksummed, evaluated in one place |
| Fake data mixed with broker data | Every record carries `source`; the simulator is only reachable from tests |
| UI before a proven engine | No dashboard until the coordinator survives the full fault matrix |

## Layers

```
market data ──> strategies ──> opportunity engine ──> risk engine ──> order coordinator ──> broker
   (bars)        (signals)        (ranking)          (decisions)     (the ONLY writer)
                                                                            │
                                              reconciliation & recovery ◄───┘
```

Dependencies point one direction only. `packages/strategies` imports no broker module —
asserted by a test that scans import lines, so a strategy structurally cannot reach a venue.

## The uncertainty model

This is the core of the design. A broker lookup returns exactly three values:

- `FOUND` — the venue holds the order
- `CONFIRMED_NOT_FOUND` — the venue durably does not
- `UNKNOWN` — everything else

Timeouts, 5xx, 401/403, 429, malformed payloads and network errors all collapse to
`UNKNOWN`. **`UNKNOWN` never authorises a resubmission.** A stalled order is recoverable;
a duplicate live order is not.

`BrokerRejected` is the one unambiguous failure — a durable refusal — and is the only
error that closes an intent out locally without consulting the venue.

## Submission sequence

```
0. reconciliation probe   lookup by client_order_id, BEFORE the risk gate
1. risk evaluation        fresh account/position/quote state, not cached
2. RESERVED               durable write before any write-side broker call
3. SUBMITTING             the uncertainty window, made explicit and durable
4. submit
5. SUBMITTED              durable write; if THIS fails the order exists and we
                          don't have it recorded → RECOVERY_REQUIRED + incident
```

Step 0 runs before risk deliberately. Adopting an order the venue already holds is
*bookkeeping*, not a trading decision. If risk ran first, an intent that had already
reached the broker could be marked `REJECTED` locally while the real order sat open —
local state contradicting broker truth, which is exactly the original failure. Risk still
gates every new submission at step 1.

`client_order_id` is `sha256(environment:broker:intent_id)[:24]`, so a retry, a restart
and a recovery pass all interrogate the same identifier.

## Recovery

The recovery worker runs on boot and on schedule over `CREATED`, `RESERVED`, `SUBMITTING`
and `RECOVERY_REQUIRED` reservations:

- `FOUND` → adopt broker truth, sync fills
- `CONFIRMED_NOT_FOUND` → `ABANDONED`; a *new decision* may later create a *new intent*
- `UNKNOWN` → leave it stuck, count the attempt, retry later

The worker never submits anything. Ever.

## Environment isolation

`PAPER` and `LIVE` are separate partitions at the storage boundary, not a convention.
The coordinator refuses a cross-environment intent (`EnvironmentMismatch`). Paper and live
share identical strategy, risk, order and reconciliation code — only credentials and
endpoints differ.

## Money

`Decimal` throughout. `money.D()` raises `TypeError` on a float, so a float cannot reach
the ledger by accident. Quantities round **down** so we never buy more than we sized.
Fees are modelled in both forms: charged in quote currency, and charged in the purchased
asset (common on crypto venues) — both reconcile to the smallest supported unit.

## Market data

Only completed bars reach strategies. `market_data.prepare()` drops forming bars,
de-duplicates by close time, orders ascending, and *reports* gaps rather than filling them.
`assert_all_complete()` raises if a forming bar reaches a signal producer.

## What is deliberately not built yet

No dashboard, no scheduler, no live adapter, no news pipeline. Those come after the
execution engine is proven, not before. See IMPLEMENTATION_PLAN.md.
