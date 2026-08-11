---
name: Reconstruct market data correctly
description: Pull event-derived price series and sparklines from the Rugspull Read API and compute prices, volumes and fees with the integer arithmetic the provider specifies — without presenting the result as an oracle or a candle feed.
api: openapi/rugspull-read-api-openapi.yml
base_url: https://rugspull.com
auth: none
operations:
  - getRugMarket
  - listMarketSparklines
  - getIndexerStatus
generated: '2026-08-11'
method: generated
source: >-
  Grounded in openapi/rugspull-read-api-openapi.yml (every operationId verified against
  the spec), the marketReconstruction block of https://rugspull.com/integration.json,
  and https://github.com/pqchase/rugspull/blob/main/docs/INTEGRATION.md
---

# Reconstruct market data correctly

This is the flow most likely to be got wrong, because the numbers look like prices and
are not one. Read the whole skill before rendering anything.

## What this data is

Every point is **derived from indexed BNB Smart Chain events** — recomputed after each
`LaunchSucceeded` or `Swap`. It is:

- **not an oracle**, and must not replace direct reserve reconciliation;
- **not OHLCV** — the API publishes no candle endpoint;
- **a cache**, which can be stale, incomplete, or rebuilt.

## Steps

### 1. Establish freshness — `getIndexerStatus`

```
GET https://rugspull.com/api/indexer/status
```

Compare `sync[].last_scanned_block` to `latestBlock` against `staleBlockThreshold`, and
read `warnings[]`. Any market number you report should carry the block it was computed
at. Do not render a chart from a stale cache without saying it is stale.

### 2. Pull the series — `getRugMarket`

```
GET https://rugspull.com/api/rugs/56/{rug}/market?limit=240
```

`limit` is 20–500, default 240. The response carries:

- `points[]` — the price observations;
- `markers[]` — the `RugPulled` marker overlaid on the series;
- `stats` — cached aggregates. The live shape observed on 2026-08-11 was
  `tradeCount`, `buyQuoteVolume`, `sellQuoteVolume`, `protocolFeeQuote`,
  `latestPriceX18`, `updatedBlock`, `complete`, `priceChangeBps`, `visiblePointCount`.
- `source` is the const string `indexed BSC events`.

`complete: true` refers to the cached series, not to chain history. Do not translate it
as "all trades ever".

### 3. Pull many at once — `listMarketSparklines`

```
GET https://rugspull.com/api/market/sparklines?chainId=56&rugs=0xAAA...,0xBBB...
```

At most **24 unique valid addresses** per call, and at most **16 recent prices per
Rug**. Over that, you get `400 {"error": "..."}`. The response is an address-keyed map
of arrays of integer strings.

## The arithmetic — integers only

```
priceX18 = reserveQuote * 1e18 / reserveToken     # after each LaunchSucceeded or Swap
buyQuoteVolume  = Swap.amountIn
sellQuoteVolume = Swap.amountOut + Swap.protocolFeeQuote
protocolFeeVolume = sum(Swap.protocolFeeQuote)
```

**Every price and volume is an integer STRING scaled by 1e18** (`pattern: ^[0-9]+$`).
Parse them with a big-integer type. Do not `JSON.parse` them into IEEE-754 doubles, do
not round for storage, and only convert to a decimal at the moment of display.

The quote asset is **WBNB only**
(`0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c`) on chain **56**.

## Fees

Nominal trade fee is **30 bps**: **25 bps stays in the pool**, **5 bps of WBNB** goes to
the immutable protocol treasury, subject to integer rounding. If you display a fee, show
the split — a single 0.30% figure implies the whole fee leaves the pool, which it does
not.

## Reconciliation is mandatory for any financial claim

Compare `RugPool.getReserves()` against the actual RugToken and WBNB balances on chain.
**A chart or a cached row cannot substitute for balance reconciliation.** If you cannot
reconcile, qualify the answer.

## If you build candles

The provider requires that a third party constructing candles **discloses** its
interval, missing-block, timestamp, reorg, rounding, and completeness rules. Publish
those rules alongside the chart or do not publish the chart.

## Do not model RugPool as PancakeSwap

RugPool is Rugspull's internal, non-upgradeable constant-product pool. It issues **no LP
token** and exposes **no reserve-withdraw function**. Never display PancakeSwap routing,
LP ownership, or lock status for it merely because both use a constant-product formula.
Third parties can create alternative pools; Rugspull cannot remove MEV, guarantee price
quality, identify related wallets, or prevent total loss.

## Errors

`getRugMarket` declares only a `200` — a malformed address or chainId has no documented
failure response. `listMarketSparklines` returns `400 {"error": "..."}` for an
over-length or malformed `rugs` list. Both are safe GETs; retry with backoff.

## Before you publish

State the cache delay and any incomplete history rather than hiding it, keep the risk,
source, Factory and security-contact links visible, and never label exact-match source
as an audit, endorsement, or safety score.
