---
name: Verify before you integrate or list
description: Run the provider's own external-display review checklist before describing a wallet, terminal, directory or data product as integrated with Rugspull, using the machine-readable discovery surface to check each claim.
api: openapi/rugspull-read-api-openapi.yml
base_url: https://rugspull.com
auth: none
operations:
  - getConfig
  - getIndexerStatus
  - getRug
  - getPublicObject
generated: '2026-08-11'
method: generated
source: >-
  Grounded in openapi/rugspull-read-api-openapi.yml (operationIds verified against the
  spec) and the external-display review checklist published in
  https://github.com/pqchase/rugspull/blob/main/docs/INTEGRATION.md
---

# Verify before you integrate or list

Rugspull publishes an explicit checklist that any third party must satisfy **before**
describing itself as integrated. This skill turns it into calls you can make. Everything
here is a `GET`, anonymous, and safe to retry.

## Start from the discovery documents, not from a blog post

| Document | URL |
|---|---|
| APIs.json index | `https://rugspull.com/.well-known/apis.json` |
| RFC 9727 API Catalog | `https://rugspull.com/.well-known/api-catalog` |
| OpenAPI 3.1 | `https://rugspull.com/openapi.json` |
| Onboarding descriptor | `https://rugspull.com/.well-known/api-onboarding` |
| security.txt | `https://rugspull.com/.well-known/security.txt` |
| Integration package | `https://rugspull.com/integration.json` |
| llms.txt | `https://rugspull.com/llms.txt` |

Every `/api/*` response also carries an RFC 8288 `Link` header with `rel="api-catalog"`,
`rel="service-desc"` and `rel="service-doc"`, so an agent can rediscover all of this
from any single call.

> **Trap.** The site is a single-page app whose catch-all answers **HTTP 200 with an
> HTML shell** for any unrecognised path, including `/.well-known/*`. A 200 is not a
> document here. Accept a response only if the `Content-Type` is not `text/html` **and**
> the body parses as the expected type. `/.well-known/agent-card.json`,
> `/.well-known/ai-plugin.json`, `/.well-known/openid-configuration` and `/mcp` all
> return the shell — none of those surfaces exists.

## The checklist, as calls

### 1. Addresses are real and traceable — `getConfig`

```
GET https://rugspull.com/api/config
```

Take `factory` and `factories` from the response. Show full Factory and Instance
addresses, traceable to a `RugCreated` event. Never infer, reserve, or display a Token
or Pool address that has not been emitted — those exist only after `LaunchSucceeded`.

### 2. Chain and quote asset are correct

Chain id must be **56** (BNB Smart Chain mainnet). The quote asset is **WBNB only**,
`0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c`. Both are confirmable from `getConfig` and
`integration.json`.

### 3. Source verification is not an audit

The Factory has exact-match source on BscScan and Sourcify. That is **not** an audit,
endorsement, or safety score, and an independent audit **has not been completed**. Label
it accurately. The provider maintains a page on exactly this distinction:
`https://rugspull.com/verified-source-code-does-not-mean-audited`.

### 4. Do not label RugPool as PancakeSwap

No LP token exists. No reserve-withdraw function exists. Do not display LP ownership,
lock status, or PancakeSwap routing.

### 5. Lifecycle semantics match the contracts — `getRug`

```
GET https://rugspull.com/api/rugs/56/{rug}
```

`Opening`, `Failed`, `Active`, `Rugged` are contract states and the progression is
monotonic. Refunds are **user-initiated** — never render them as automatic. `Rugged` is
a state, not a scam verdict.

A `404 {"error":"Rug not indexed"}` is a cache miss, not proof of nonexistence.

### 6. Market formulas use the correct integer fields

See the `Reconstruct market data correctly` skill. Integer strings scaled by 1e18;
`sellQuoteVolume = amountOut + protocolFeeQuote`; fee split 25/5 bps.

### 7. Disclose cache delay — `getIndexerStatus`

```
GET https://rugspull.com/api/indexer/status
```

Publish the lag between `sync[].last_scanned_block` and `latestBlock`, surface
`warnings[]`, and never hide direct-chain evidence behind your own cached view. There is
no uptime SLA and no numeric rate limit; back off exponentially and cache responsibly.

### 8. Risk, source, Factory and security-contact links are visible

Link `https://rugspull.com/docs/risk`, the BscScan Factory page, the source repository,
and `info@rugspull.com`. Total loss remains possible and organized new mainnet activity
is recorded as **NO-GO** while the published Stage 0 gates are unresolved
(`https://rugspull.com/stage-0-review`).

### 9. No consideration was exchanged

Confirm no paid placement, expedited review, rebate, token allocation, stake, gas, or
other promotional consideration changed hands.

### 10. Inspect the third party's public record

Before any listing, integration, partnership, recommendation, or endorsement claim is
made — in either direction.

## Metadata and images — `getPublicObject`

```
GET https://rugspull.com/api/r2/metadata/{hash}.json
GET https://rugspull.com/api/r2/assets/{hash}.{ext}
```

Content-addressed and immutable. Keys outside the public-key policy are rejected with a
`400 {"error":"Invalid R2 object key"}` — a status the spec does not declare, so handle
it explicitly rather than trusting a generated client.

## Reporting a problem

Security and correction contact: `info@rugspull.com` (also the `Contact` in
security.txt, Policy at `https://rugspull.com/community-safety`). Send funds-at-risk
reports **privately**. Do not publish an unpatched exploit, a private key, a seed
phrase, or personal data in a public issue. **No bounty and no response-time SLA is
offered.**

## What you may not claim

Publication of Rugspull's integration package does not claim wallet, terminal,
aggregator, directory, BNB Chain, or other third-party integration, review, partnership,
recommendation, or endorsement — and neither does reading this API. Do not infer an
audit, endorsement, partnership, user count, TVL, returns, safety, or regulatory
approval from a directory submission, an automated review, a social follow, source
verification, or passing project-authored tests.
