---
name: Discover and inspect a Rug
description: Walk the Rugspull discovery cache from chain configuration to a single Rug's record and event history, checking indexer freshness first and treating a cache miss correctly.
api: openapi/rugspull-read-api-openapi.yml
base_url: https://rugspull.com
auth: none
operations:
  - getConfig
  - getIndexerStatus
  - listRugs
  - getRug
  - listRugEvents
generated: '2026-08-11'
method: generated
source: >-
  Grounded in openapi/rugspull-read-api-openapi.yml (every operationId verified against
  the spec), https://github.com/pqchase/rugspull/blob/main/docs/INTEGRATION.md, and
  https://rugspull.com/integration.json
---

# Discover and inspect a Rug

Read-only. Every operation is an HTTP `GET`, no credential is required, and every call
is safe to retry. There is no published rate limit and no `Retry-After` header, so use
exponential backoff of your own.

## Before you read anything

**BNB Smart Chain contract state is financial truth.** This API is a rebuildable cache.
If your answer matters financially, verify it against the chain. Never present a value
from this API as settled, guaranteed, audited, or endorsed.

## Steps

### 1. Anchor on the current chain and Factory — `getConfig`

```
GET https://rugspull.com/api/config
```

Returns `chainId`, `factory` (the active RugFactory), `factories` (all factories the
indexer tracks), and `financialTruth`. Take the Factory address from here rather than
hard-coding it — the field is nullable and the list can grow.

### 2. Check freshness before you trust anything — `getIndexerStatus`

```
GET https://rugspull.com/api/indexer/status
```

Compare `sync[].last_scanned_block` against `latestBlock`. If the gap exceeds
`staleBlockThreshold` (1200 at the time of writing, but read it from the response),
the cache is stale — say so in your answer instead of reporting a number as current.

Read `warnings[]` on every call. An **empty `warnings` array does not create an SLA and
does not prove complete history** — that is the provider's own language.

### 3. List what is indexed — `listRugs`

```
GET https://rugspull.com/api/rugs?limit=25
GET https://rugspull.com/api/rugs?status=Active&cursor=0&limit=100
```

- `status` is one of `Opening`, `Failed`, `Active`, `Rugged`.
- `limit` is 1–100 (default 25). `cursor` is 0–1000000 (default 0).
- Page by passing the returned `nextCursor` back as `cursor`. Stop when `rugs` comes
  back empty — there is no explicit exhaustion sentinel.
- Out-of-range parameters return `400 {"error": "..."}`.

An empty list is a legitimate answer. At the time this skill was written the mainnet
cache held zero Rugs, consistent with the provider's published NO-GO posture on
organized mainnet activity.

### 4. Read one Rug — `getRug`

```
GET https://rugspull.com/api/rugs/56/{rug}
```

`chainId` is an integer ≥ 1 (56 for BNB Smart Chain mainnet). `rug` must match
`^0x[a-fA-F0-9]{40}$` — a RugInstance address emitted by the configured Factory.

**A `404 {"error":"Rug not indexed"}` means the CACHE does not have it.** It is not
evidence that the contract or its events do not exist. Do not report "this Rug does not
exist"; report "not present in the discovery cache" and, if it matters, check BscScan.

### 5. Read its event history — `listRugEvents`

```
GET https://rugspull.com/api/rugs/56/{rug}/events
```

Events are ordered by block number then log index, and the response is **capped at 100
rows with no cursor**. Beyond 100 events you must read the chain directly; there is no
paging past the ceiling.

Note the inconsistency: this operation returns `200 {"events":[]}` for an address that
`getRug` answers `404` for. Do not infer existence from a 200 here.

## The nine events and what they mean

| Contract | Event | Meaning |
|---|---|---|
| RugFactory | `RugCreated` | A new Instance; binds creator, stake, Opening end, metadata hash, disclosure hash |
| RugInstance | `Contributed` | Contribution flow. Wallet count is **not** a unique-person count |
| RugInstance | `LaunchFailed` | Failed state; refund eligibility is exposed. Refunds are **not** automatic |
| RugInstance | `LaunchSucceeded` | Binds Token, RugPool, accepted contribution, allocation, initial reserves |
| RugInstance | `ClaimedOpening` | A claim + excess refund actually executed by one wallet |
| RugInstance | `ClaimedFailedRefund` | A Failed-state refund actually executed by one wallet |
| RugInstance | `CreatorStakeWithdrawn` | Creator stake withdrawal, separate from contributor refunds |
| RugPool | `Swap` | Side, amounts, protocol fee, post-event reserves, price, WBNB volume |
| RugInstance | `RugPulled` | The one full Founder Allocation exit and the quote received |

## Language rules you must follow when reporting

These are the provider's stated boundaries, not stylistic preferences:

- **`Rugged` is a lifecycle state**, not a scam verdict, safety label, refund
  condition, loss calculation, or proof that anyone stopped trading.
- **Refunds are user-initiated.** Never describe a refund as automatic.
- **RugPool is not PancakeSwap.** It is an internal constant-product AMM with no LP
  token and no reserve-withdraw function. Do not show LP ownership, lock status, or
  PancakeSwap routing for it.
- **Exact-match source is not an audit.** An independent audit has not been completed.
- Token and Pool addresses **exist only after `LaunchSucceeded`**. Never infer,
  reserve, or publish an address that has not been emitted.

## Errors

| Status | Body | What to do |
|---|---|---|
| 400 | `{"error": "..."}` | A parameter is out of range or malformed. Clamp and retry. |
| 404 | `{"error":"Rug not indexed"}` | Not in the cache. Fall back to the chain; do not claim nonexistence. |
| 503 | `{"error": "..."}` | Discovery cache unavailable. Back off and retry; not a statement about protocol state. |

Errors are flat JSON, not RFC 9457 problem details. There is no machine-readable error
code — branch on the status, and string-match only as a last resort.
