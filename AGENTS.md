# agents.md

> Machine-readable contract for agents working on or integrating with the Loot platform.
> Keep this file updated when any endpoint, payload shape, or enum changes.
> Source of truth lives here — not in comments scattered across screens.

---

## Platform Summary

Loot is a loyalty-funded digital goods ecosystem. Consumers earn game items,
crafting materials, and points by spending at partner merchants — via QR scan
today, card-linked transactions in the future. Items are usable across a network
of Loot-integrated games and apps. The consumer-facing surface is a React Native
(Expo) app with a central camera button as the primary action. All camera output
flows through a single versioned ingest endpoint before fanning out to inventory,
reward, or event logic.

The platform runs on three AWS primitives chosen for near-zero idle cost: **S3**
for static hosting and asset storage, **Lambda** for all backend compute, and
**CloudFront** as the single public edge. See [Infrastructure](#infrastructure).

### System Actors

| Actor              | Description                                                    | Auth scope        |
|--------------------|----------------------------------------------------------------|-------------------|
| **Consumer**       | End user scanning QR codes, collecting items, browsing market  | `consumer`        |
| **Merchant**       | Business partner funding rewards, viewing analytics            | `merchant`        |
| **Game Developer** | Third-party integrator consuming item data via partner API     | `partner`         |
| **Admin**          | Internal operator managing catalog, listings, campaigns        | `admin`           |

Only the Consumer scope is implemented in Phase 1. Other scopes are defined here
so agents build with expansion seams, not refactors.

---

## Infrastructure

The cornerstone of the stack is a serverless, pay-per-use triad. Nothing runs
when nobody scans, which is the entire cost strategy for Phase 1.

```
                        ┌───────────────┐
  Expo app (iOS/Android)│               │ /v1/*      → API Gateway → Lambda → DynamoDB
  Web / marketing SPA  ─┤  CloudFront   ├ /assets/*  → S3 assets bucket (private, OAC)
  Merchant dashboard   ─┤ (one distro)  │ /*         → S3 web bucket    (private, OAC)
                        └───────────────┘
```

| Layer            | Service     | Why it was chosen                                                                 |
|------------------|-------------|-----------------------------------------------------------------------------------|
| Static delivery  | S3          | Storage is ~$0.023/GB-month with no server to run. Versioning gives instant rollback. |
| Compute          | Lambda      | Billed per millisecond of execution. A day with 200 scans costs cents; a day with 200k autoscales without intervention. |
| Edge             | CloudFront  | One TLS endpoint, one domain, one CORS/security-header policy in front of both origins. Caches item art near the user. |

The important property is not that these are cheap — it's that they are cheap
*while idle*. A container or an RDS instance charges the same whether the app has
10 users or 10,000. That difference is what makes a pre-revenue loyalty network
viable to run.

### S3

Three buckets per stage:

| Bucket                  | Contents                                          | Public? |
|-------------------------|---------------------------------------------------|---------|
| `loot-web-<stage>`      | Marketing site, merchant dashboard SPA build       | No      |
| `loot-assets-<stage>`   | Item art, merchant logos, avatars, reward images   | No      |
| `loot-artifacts-<stage>`| Lambda bundles, IaC output, build artifacts        | No      |

Rules:

- **Do not enable S3 static website hosting.** That feature exposes an HTTP-only
  public endpoint and requires a world-readable bucket policy. Instead, keep
  Block Public Access fully on and let CloudFront read the bucket through an
  **Origin Access Control (OAC)**. The bucket policy grants exactly one principal:
  the distribution. You get HTTPS, WAF compatibility, and no path to a leaked
  bucket URL.
- Enable SSE-S3 encryption and versioning on the web and assets buckets. Versioning
  turns a bad deploy into a one-command rollback rather than a rebuild.
- **Content-hash every asset filename** — `sku_abc123.a91f3c.webp`, not
  `sku_abc123.webp`. Hashed files get `Cache-Control: public, max-age=31536000, immutable`;
  `index.html` gets `no-cache`. This is the whole trick to a fast, inexpensive CDN:
  because the name changes when the bytes change, you never invalidate the cache,
  and CloudFront invalidations past the free tier are billed per path.
- Item art should be WebP or AVIF, resized at build/upload time to the largest size
  the app actually renders. Egress is the dominant cost line at this scale, so image
  weight matters more than any backend optimization.

### CloudFront

One distribution, three path behaviors:

| Path pattern | Origin                | Cache policy        | Notes                                                                 |
|--------------|-----------------------|---------------------|-----------------------------------------------------------------------|
| `/v1/*`      | API Gateway           | `CachingDisabled`   | Forward `Authorization`, `x-api-key`, `x-idempotency-key`, `x-app-version`. Authenticated responses must never be cached. |
| `/assets/*`  | S3 assets (OAC)       | `CachingOptimized`  | Long TTL, immutable filenames, Brotli/gzip on                          |
| `/*`         | S3 web (OAC)          | `CachingOptimized`  | 403/404 → `/index.html` with 200 for SPA client-side routing            |

- Start on **Price Class 100** (North America + Europe). Widen when partner or
  merchant traffic justifies the additional edge locations.
- Attach a response headers policy carrying HSTS, `X-Content-Type-Options: nosniff`,
  a CSP, and the CORS allowlist for the app origins. Setting these once at the edge
  beats setting them in every Lambda handler.
- The `imageUrl` and `logoUrl` fields throughout this contract
  (`https://cdn.example.com/...`) resolve to the `/assets/*` behavior. Never serve
  binary assets through Lambda — it is roughly two orders of magnitude more expensive
  per byte and adds cold-start latency to an image load.
- Routing the API through the same distribution means one domain, one certificate,
  and no CORS preflight from the web clients. It also gives you a single place to
  attach AWS WAF when abuse starts.

### Lambda

- Runtime Node 20+ on **`arm64` (Graviton)** — same code, roughly 20% lower cost per
  GB-second.
- **One function per endpoint group, not per route and not a monolith:** `scan`,
  `inventory`, `market`, `redeem`, `profile`. Per-route functions fragment your warm
  pool so every request pays a cold start; a monolith forces one memory setting and
  one concurrency limit on workloads with very different shapes. Grouping keeps the
  hot path warm and lets you tune each group independently.
- `POST /v1/scan` is the latency-critical path — a user is standing at a counter
  waiting for it. Give it **reserved concurrency** so a marketplace traffic spike
  cannot starve it, and add **provisioned concurrency** when p99 cold start becomes
  the complaint.
- **Memory is the CPU dial.** Lambda scales vCPU with memory, so 1024 MB is frequently
  both faster *and* cheaper than 512 MB for JSON parsing plus HMAC verification,
  because it more than halves the duration. Measure with Lambda Power Tuning rather
  than guessing.
- Initialize SDK clients and load per-merchant HMAC secrets **outside** the handler
  function. That code runs once per execution environment, not once per request.
- Keep handlers thin: parse → validate → call a plain domain function → serialize.
  Domain logic that has no knowledge of the Lambda event shape is testable with a
  unit test runner and portable if any part of it later moves to Fargate or a
  container.
- Authorization belongs in an **API Gateway Lambda authorizer**, which validates the
  JWT once and injects `sub` and `scope` into the request context. This is the
  middleware layer referenced in [Auth](#auth) — handlers read
  `context.authorizer.scope` and never parse tokens themselves, so adding the
  `merchant` and `partner` scopes in Phase 2 is configuration rather than a rewrite.
- Timeouts: 5s for reads, 10s for writes. Generous timeouts hide bugs and bill you
  for the time it takes to hide them.
- **Idempotency:** the `x-idempotency-key` header maps to a conditional write on an
  idempotency record with a 24-hour TTL. A repeated key returns the stored response
  instead of re-running the claim. This is what makes the retry guidance in
  [Rate Limits](#rate-limits) and [CameraScreen Integration Point](#camerascreen-integration-point)
  actually safe.

### Secrets

Per-merchant HMAC secrets used to verify the QR `sig` field live in AWS Secrets
Manager (or SSM Parameter Store for lower cost), fetched at cold start and cached in
module scope. They must not sit in Lambda environment variables — env vars are visible
to anyone with `GetFunctionConfiguration` and cannot be rotated without a redeploy.

### Data layer

Not yet part of the contract, but the natural pair: **DynamoDB on-demand** has the
same cost shape as Lambda — per-request billing, zero idle charge — and no connection
pool to exhaust from a scaling fleet of Lambdas, which is the standard failure mode
when serverless compute is put in front of a relational database.

Single-table design, `PK`/`SK`:

| PK                | SK                | Purpose                                                        |
|-------------------|-------------------|----------------------------------------------------------------|
| `USER#<userId>`   | `ITEM#<sku>`      | Inventory rows; supports the paginated inventory query          |
| `USER#<userId>`   | `PROFILE`         | Balance, stats                                                  |
| `QR#<qrId>`       | `USER#<userId>`   | One-claim-per-code uniqueness via conditional write             |
| `IDEMP#<key>`     | `-`               | Idempotency records, TTL 24h                                    |
| `SKU#<sku>`       | `SUPPLY`          | Atomic counter for `maxSupply` enforcement                      |

Read amplification is handled by the tiers in [Caching Strategy](#caching-strategy)
rather than by a separate cache cluster — see that section for why DAX and ElastiCache
are deferred.

The economy constraints in [Economy Constraints](#economy-constraints) are conditional
writes, not application-level checks — a `ConditionExpression` failure is the
authoritative "already claimed," and it holds under concurrent scans in a way that a
read-then-write never will.

### Observability

Structured JSON logs from every handler, with the request ID, `sub`, and idempotency
key on every line. Set CloudWatch log retention explicitly (14 days for dev, 30–90 for
prod) — the default is "never expire," and log storage quietly becomes the largest
line on a small serverless bill.

### Environments and deployment

`dev`, `staging`, and `prod` as separate AWS accounts where possible, separate stacks
at minimum. Define everything with IaC — AWS CDK or SAM — and never through the
console. The stack is small enough today that doing this correctly is nearly free;
retrofitting IaC onto hand-built infrastructure is one of the more expensive cleanups
in this category of system.

`API_BASE_URL` and `CDN_BASE_URL` are the only per-stage values the client needs.

### Cost expectations

Below roughly 100k scans/month this stack lands in single-digit dollars per month:
Lambda and DynamoDB largely within free-tier allowances, S3 storage negligible, and
CloudFront charging mostly for asset egress. Plan capacity against image weight and
CloudFront transfer, not against compute.

---

## Caching Strategy

Caching is a first-class cost control here, not a performance afterthought. The
cheapest request is the one that never leaves the device; the second cheapest is the
one CloudFront answers without waking a Lambda. Every tier below exists to keep
traffic away from the tier beneath it.

```
  L0  Client (Expo app)      in-process + persisted   ── requests never made
  L1  CloudFront edge        shared, public reads     ── no Lambda invocation
  L2  Lambda module scope    per-execution-env memory ── no DynamoDB read
  L3  DynamoDB               source of truth
  L4  Redis / Valkey         NOT IN PHASE 1 — see below
```

### What is cacheable, and where

| Data                                            | Volatility     | Tier(s)          | TTL                        |
|-------------------------------------------------|----------------|------------------|----------------------------|
| Item catalog metadata (name, art, rarity, attributes) | near-immutable | L1 + L2      | 24h edge / 15m memory      |
| Merchant metadata (name, logo, active flag)     | hours          | L2               | 5m                         |
| Campaign config and drop tables                 | hours          | L2               | 5m                         |
| Per-merchant HMAC secrets                       | rotation-driven| L2               | 15m                        |
| `GET /v1/market` listings                       | minutes        | L1 + L0          | `s-maxage=60`, SWR 300     |
| `GET /v1/redeem/options` catalog                | hours          | L1 + L0          | `s-maxage=300`             |
| Static assets (`/assets/*`)                     | immutable      | L1 + L0          | 1 year (hashed filenames)  |
| `GET /v1/inventory`                             | per-user       | L0 only          | `private, max-age=30`      |
| `GET /v1/profile` (balance, stats)              | per-user       | L0 only          | `private, max-age=15`      |
| `POST /v1/scan`, `/market/purchase`, `/redeem`  | never          | —                | `no-store`                 |

**Never cache an authenticated response at a shared tier.** Any response that varies
by `sub` must carry `Cache-Control: private` and must be served through the
`CachingDisabled` behavior. A shared cache that keys on URL alone and forgets the
identity dimension serves one user's inventory to another — the single highest-severity
failure mode in this design.

### L0 — Client cache

The largest cost lever, and the one most teams skip. Use TanStack Query (or equivalent)
in the Expo app with a persisted store:

- `staleTime` per resource matching the table above; catalog data can be effectively
  permanent.
- Inventory and balance revalidate on focus and after any mutation, not on a timer.
- After a successful scan, **write the returned item into the local inventory cache
  directly** rather than refetching `/v1/inventory`. The scan response already contains
  the item and the new totals. Refetching after every scan roughly doubles read traffic
  for zero information gain.
- Cache item art on device (`expo-image` with a disk cache). Image bytes are the
  dominant CloudFront cost, and the same item art is re-rendered on every inventory view.

### L1 — CloudFront

- Public catalog reads (`/v1/market`, `/v1/redeem/options`) go through a cache policy
  keyed on the query string only — deliberately **not** on `Authorization`. That
  requires these responses to stay user-agnostic. Keep balances and ownership flags out
  of them; the client already knows the balance from `/v1/profile`.
- Use `stale-while-revalidate` so an expired entry is served immediately while one
  request refreshes it. Without it, TTL expiry on a popular key sends every concurrent
  request to origin at once.
- Support `ETag` / `If-None-Match` on all `GET` endpoints. A `304` costs almost nothing
  in egress and is the difference between a cheap poll and an expensive one.
- Return `Vary: Accept-Encoding` and nothing else on public responses. Every extra
  `Vary` header multiplies the number of cache entries and lowers hit rate.

### L2 — Lambda module scope

A plain `Map` with expiry, declared outside the handler, surviving across warm
invocations. It costs nothing, adds zero network latency, and covers the highest-value
case: reference data read on every single scan.

```js
// module scope — persists across warm invocations
const cache = new Map();

async function cached(key, ttlMs, loader) {
  const hit = cache.get(key);
  if (hit && hit.expires > Date.now()) return hit.value;
  const value = await loader();
  // ±20% jitter so warm environments don't all expire together
  const jitter = ttlMs * (0.8 + Math.random() * 0.4);
  cache.set(key, { value, expires: Date.now() + jitter });
  return value;
}
```

Constraints agents must respect:

- It is **per execution environment**, not global. Ten warm Lambdas hold ten copies.
  That makes it eventually consistent with a bound of one TTL — acceptable for catalog
  and merchant data, unacceptable for balances, inventory, or supply counters.
- There is **no invalidation**. A catalog edit propagates within one TTL, not instantly.
  Keep TTLs short enough that this is tolerable (5–15 minutes) rather than reaching for
  a distributed cache to solve it.
- Bound the map size. An unbounded cache in a long-lived execution environment is a
  memory leak that ends in an OOM kill on the hot path.
- Never cache anything keyed by user in module scope. Execution environments are reused
  across users, and a stale per-user entry is a data-leak bug, not a performance bug.

### Negative caching

Cache misses too: unknown SKUs, inactive merchants, and expired campaigns. Scan
endpoints are publicly reachable and will be probed with garbage IDs; without negative
caching every probe becomes a DynamoDB read. Short TTL (60s) so a genuinely new catalog
entry appears quickly.

### L4 — Redis / Valkey: not in Phase 1

**Recommendation: do not add Redis now.** It is the wrong shape for this workload at
this stage, for three reasons.

1. **It reintroduces idle cost, which is the thing this stack was built to avoid.**
   ElastiCache has no scale-to-zero. As of mid-2026 in `us-east-1`, the cheapest node
   is a `cache.t4g.micro` on Valkey at roughly $9/month; ElastiCache Serverless bills a
   storage minimum around the clock — about $6/month on Valkey (100 MB floor) and about
   $91/month on Redis OSS (1 GB floor) even holding nothing. Verify current rates before
   committing; the direction of the argument is stable even if the numbers move.
2. **The thing it would replace is nearly free.** DynamoDB on-demand eventually
   consistent reads run about $0.125 per million. A $6/month cache floor is roughly
   50 million cached reads before it breaks even — and that ignores the cache's own
   compute charges. Phase 1 will not come close.
3. **It drags the compute into a VPC.** ElastiCache is VPC-only, so every Lambda that
   touches it must be VPC-attached. Gateway endpoints for S3 and DynamoDB are free, but
   Secrets Manager, KMS, and any other AWS API then need interface endpoints (~$7/month
   each) or a NAT gateway (~$32/month plus per-GB processing). The supporting plumbing
   can cost more than the cache.

**What would actually justify it.** Redis earns its place when you need something
DynamoDB and in-process memory genuinely cannot do:

| Trigger                                                        | Phase | Why Redis specifically                                    |
|----------------------------------------------------------------|-------|-----------------------------------------------------------|
| Leaderboards (seasonal + all-time)                             | 3     | Sorted sets give O(log n) rank queries; DynamoDB has no native ranked read |
| Sliding-window rate limits beyond API Gateway's fixed quotas    | 2–3   | Atomic counters shared across all concurrent Lambdas       |
| High-contention supply counters on a viral drop                | 2–3   | A single hot DynamoDB partition key throttles; `INCR` does not |
| Real-time presence, guild state, or pub/sub fan-out            | 3     | DynamoDB has no pub/sub primitive                          |

Note that three of these map directly to Phase 3 roadmap items. Redis is a **Phase 3
capability enabler**, not a Phase 1 cost optimization — which is the inverted reason
most teams reach for it.

**When you do add it:** choose **Valkey**, not Redis OSS — same API, priced roughly 20%
below on nodes and about a third below on serverless, with a 15× lower serverless
storage floor. If keeping Lambda out of a VPC is worth more than staying inside AWS,
an HTTP-addressable serverless Redis (Upstash, Momento) preserves the pay-per-request
cost shape and the zero-VPC deployment; the trade is a third-party dependency in the
request path. Either way, treat it as an L4 behind the tiers above — a cache that the
system works correctly without, never a store the system depends on.

### DAX

Same verdict as Redis, same reasoning: DAX is a provisioned in-VPC cluster with no
scale-to-zero. It becomes interesting only when DynamoDB read spend clearly exceeds the
cluster cost and single-digit-millisecond reads are demonstrably too slow — measure
before assuming either.

### Rules for agents

1. Add the tier closest to the client that solves the problem. Reach for a distributed
   cache only after L0, L1, and L2 are exhausted and instrumented.
2. Every cache entry needs an explicit TTL and an owner-documented staleness budget.
   "How wrong is this allowed to be, for how long" is a product decision, not an
   implementation detail.
3. Correctness never depends on a cache hit. Every path must be correct, if slower,
   with all caches empty.
4. Instrument hit rate per tier from day one. An uninstrumented cache is an unfalsifiable
   claim about performance.

---

## API Versioning

All endpoints are prefixed with `/v1/`. Breaking changes require a new version
prefix. Additive changes (new optional fields, new endpoints) do not.

```
{API_BASE_URL}/v1/scan
{API_BASE_URL}/v1/inventory
{API_BASE_URL}/v1/market
...
```

When a new version ships, the previous version remains live for a minimum of
90 days with a `Sunset` response header indicating the deprecation date.

---

## Environment Variables

```
API_BASE_URL=https://<id>.execute-api.<region>.amazonaws.com/<stage>
API_KEY=<x-api-key header value>
CDN_BASE_URL=https://cdn.example.com          # CloudFront /assets/* behavior
WEB_BASE_URL=https://example.com              # CloudFront SPA behavior
```

In production, `API_BASE_URL` should point at the CloudFront `/v1/*` behavior
(`https://example.com/v1`) rather than the raw API Gateway hostname, so that clients
never depend on a generated AWS domain. The execute-api URL stays valid for `dev`.

All requests include:
```
Content-Type: application/json
Authorization: Bearer <jwt>
x-api-key: <API_KEY>
x-app-version: <semver>
x-platform: ios | android
x-idempotency-key: <uuid>       # required on all POST/PUT — enables safe retries
```

---

## Auth

### Consumer Auth (Phase 1)

All consumer endpoints derive `userId` from the `sub` claim in the Bearer JWT
issued by the auth provider (Cognito, Auth0). **The client never sends `userId`
as a parameter.** The server extracts identity exclusively from the token. Any
request where a body or query contains a `userId` field will be ignored in favor
of the JWT claim — agents must not rely on client-supplied identity.

```
Authorization: Bearer <jwt>
```

### Future Auth Scopes (Phase 2+)

| Scope        | Token issuer     | Access                                       |
|--------------|------------------|----------------------------------------------|
| `consumer`   | Cognito / Auth0  | Scan, inventory, market, redeem, profile      |
| `merchant`   | Cognito / Auth0  | Merchant dashboard, campaign management       |
| `partner`    | API key + secret | Read user inventory (with user consent OAuth) |
| `admin`      | Internal SSO     | Full catalog, listing, and economy management |

Merchant and partner auth is not yet implemented. Agents building server-side
handlers should structure authorization checks as a middleware layer that reads
a `scope` claim from the JWT, so adding new scopes requires config, not code changes.

---

## Rate Limits

Limits are per-authenticated-user unless noted. Exceeding a limit returns `429`
with a `Retry-After` header (seconds).

| Endpoint group   | Limit              | Window  | Notes                                    |
|------------------|--------------------|---------|------------------------------------------|
| `POST /v1/scan`  | 30 requests        | 1 min   | Burst protection — normal use is <5/min  |
| `GET` (all)      | 120 requests       | 1 min   | Covers inventory, market, profile, etc.  |
| `POST /v1/market/*` | 10 requests     | 1 min   | Purchase rate cap                        |
| `POST /v1/redeem`| 5 requests         | 1 min   | Redemption rate cap                      |
| Global (per IP)  | 300 requests       | 1 min   | Unauthenticated / pre-auth fallback      |

Client agents should implement exponential backoff with jitter on 429 and 500
responses. Safe to retry any request that includes an `x-idempotency-key`.

---

## QR Code Ingest

### What the camera produces

After capture, the app decodes the QR payload string before posting. QR codes
in this system always encode a compact JSON object — not a URL.

```json
{
  "t": "item",
  "id": "sku_abc123",
  "m": "mrc_cafe_01",
  "c": "cmp_spring25",
  "sig": "hmac-sha256-hex"
}
```

| Field | Type   | Values                                | Required | Purpose                              |
|-------|--------|---------------------------------------|----------|--------------------------------------|
| `t`   | string | `"item"` \| `"reward"` \| `"event"`  | yes      | Discriminator — drives response shape |
| `id`  | string | opaque SKU or reward ID               | yes      | Server-side lookup key               |
| `m`   | string | merchant ID prefixed `mrc_`           | yes      | Originating merchant — drives billing, analytics, drop tables |
| `c`   | string | campaign ID prefixed `cmp_`           | no       | Campaign attribution — nullable for evergreen codes |
| `sig` | string | HMAC-SHA256 hex, server-side secret   | yes      | Anti-counterfeiting; covers `t`, `id`, `m`, `c` |

The `sig` is computed over the concatenation of `t`, `id`, `m`, and `c` (empty
string if absent) using a per-merchant HMAC secret. The client never reads `sig`.

---

### POST /v1/scan

Submits a decoded QR payload for server-side validation and processing.

**Request**
```
POST {API_BASE_URL}/v1/scan
```
```json
{
  "qr": {
    "t": "item",
    "id": "sku_abc123",
    "m": "mrc_cafe_01",
    "c": "cmp_spring25",
    "sig": "a3f9..."
  },
  "scannedAt": "2025-03-02T14:22:00Z",
  "location": {
    "lat": 40.7128,
    "lng": -74.0060,
    "accuracy": 12.5
  }
}
```

| Field        | Type            | Required | Notes                                          |
|--------------|-----------------|----------|-------------------------------------------------|
| `qr`         | object          | yes      | Raw decoded QR object, unmodified               |
| `scannedAt`  | ISO 8601 string | yes      | Device-local timestamp                          |
| `location`   | object          | no       | GPS at scan time — used for geo-fencing validation and merchant analytics |

`userId` is **not** in the request body. It is derived server-side from the JWT.

**Response — `t: "item"`**
```json
{
  "status": "claimed",
  "type": "item",
  "item": {
    "id": "sku_abc123",
    "name": "Obsidian Blade",
    "description": "A rare tier-3 weapon forged in volcanic glass.",
    "imageUrl": "https://cdn.example.com/items/sku_abc123.webp",
    "rarity": "rare",
    "category": "weapon",
    "quantity": 1,
    "attributes": {
      "damage": 120,
      "durability": 80
    },
    "tags": ["craftable", "tradeable"],
    "usableIn": ["loot_core", "partner_game_xyz"]
  },
  "source": {
    "type": "merchant_scan",
    "merchantId": "mrc_cafe_01",
    "merchantName": "Copper Kettle Cafe",
    "campaignId": "cmp_spring25"
  },
  "inventory": {
    "totalItems": 14,
    "updatedAt": "2025-03-02T14:22:01Z"
  }
}
```

**Response — `t: "reward"`**
```json
{
  "status": "claimed",
  "type": "reward",
  "reward": {
    "id": "rwd_spring25",
    "name": "Spring Campaign Bonus",
    "description": "500 bonus points.",
    "rewardType": "points",
    "value": 500,
    "currency": "pts",
    "expiresAt": null
  },
  "source": {
    "type": "merchant_scan",
    "merchantId": "mrc_cafe_01",
    "merchantName": "Copper Kettle Cafe",
    "campaignId": "cmp_spring25"
  },
  "balance": {
    "points": 3240,
    "updatedAt": "2025-03-02T14:22:01Z"
  }
}
```

**Response — `t: "event"`**
```json
{
  "status": "registered",
  "type": "event",
  "event": {
    "id": "evt_summit25",
    "name": "Developer Summit 2025",
    "checkedInAt": "2025-03-02T14:22:01Z",
    "bonusAwarded": {
      "rewardType": "points",
      "value": 100
    }
  },
  "source": {
    "type": "event_checkin",
    "merchantId": "mrc_events_co",
    "merchantName": "Loot Events",
    "campaignId": null
  }
}
```

**Error responses**
```json
{ "status": "already_claimed", "message": "This code was already redeemed." }
{ "status": "invalid_signature", "message": "QR code signature mismatch." }
{ "status": "expired", "message": "This code expired on 2025-01-01." }
{ "status": "not_found", "message": "Unknown item ID." }
{ "status": "merchant_inactive", "message": "Merchant is not currently active." }
{ "status": "geofence_rejected", "message": "Scan location outside merchant radius." }
```

| HTTP code | Meaning                                      |
|-----------|----------------------------------------------|
| 200       | Claimed / registered successfully            |
| 409       | Already claimed by this user                 |
| 422       | Invalid signature, malformed payload, or geofence failure |
| 404       | ID not found in catalog                      |
| 410       | Code expired                                 |
| 403       | Merchant inactive or auth mismatch           |
| 429       | Rate limited — respect `Retry-After` header  |
| 500       | Server fault — safe to retry with same idempotency key |

---

## Inventory

### GET /v1/inventory

Returns the authenticated user's item list, paginated. Identity derived from JWT.

**Caching:** per-user, therefore `Cache-Control: private, max-age=30` and never edge-cached.
Clients should not refetch this after a scan — the scan response already returns the
claimed item and updated totals, which the client writes into its local cache directly.

**Request**
```
GET {API_BASE_URL}/v1/inventory?page=1&limit=20&category=weapon&rarity=rare
```

All query params except `page` and `limit` are optional filters.

| Param      | Type   | Default   | Notes                              |
|------------|--------|-----------|------------------------------------|
| `page`     | int    | 1         |                                    |
| `limit`    | int    | 20        | Max 100                            |
| `category` | string | (all)     | Filter by category enum            |
| `rarity`   | string | (all)     | Filter by rarity enum              |
| `tag`      | string | (all)     | Filter by tag (e.g. `craftable`)   |
| `sort`     | string | `newest`  | `newest` \| `rarity` \| `name`    |

**Response**
```json
{
  "items": [
    {
      "id": "sku_abc123",
      "name": "Obsidian Blade",
      "imageUrl": "https://cdn.example.com/items/sku_abc123.webp",
      "rarity": "rare",
      "category": "weapon",
      "quantity": 1,
      "acquiredAt": "2025-03-02T14:22:01Z",
      "source": {
        "type": "merchant_scan",
        "merchantId": "mrc_cafe_01",
        "merchantName": "Copper Kettle Cafe"
      },
      "attributes": {
        "damage": 120,
        "durability": 80
      },
      "tags": ["craftable", "tradeable"],
      "usableIn": ["loot_core", "partner_game_xyz"]
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 14,
    "hasNext": false
  }
}
```

---

## Market

### GET /v1/market

Returns active marketplace listings. Phase 1: listings are admin-created only.
Phase 2 will add player-to-player listings (see Roadmap Extensions).

**Request**
```
GET {API_BASE_URL}/v1/market?category=weapon&sort=newest&page=1&limit=20
```

| Param      | Type   | Default   | Notes                                      |
|------------|--------|-----------|--------------------------------------------|
| `page`     | int    | 1         |                                              |
| `limit`    | int    | 20        | Max 100                                      |
| `category` | string | (all)     | Filter by category enum                      |
| `rarity`   | string | (all)     | Filter by rarity enum                        |
| `sort`     | string | `newest`  | `newest` \| `price_asc` \| `price_desc` \| `ending_soon` |
| `currency` | string | (all)     | Filter by price currency                     |

**Response**
```json
{
  "listings": [
    {
      "listingId": "lst_001",
      "item": {
        "id": "sku_xyz789",
        "name": "Flame Shield",
        "imageUrl": "https://cdn.example.com/items/sku_xyz789.webp",
        "rarity": "epic",
        "category": "armor",
        "tags": ["tradeable"],
        "usableIn": ["loot_core"]
      },
      "price": {
        "amount": 750,
        "currency": "pts"
      },
      "stock": 3,
      "seller": {
        "type": "platform"
      },
      "endsAt": "2025-04-01T00:00:00Z"
    }
  ],
  "pagination": { "page": 1, "limit": 20, "total": 42, "hasNext": true }
}
```

The `seller.type` field is `"platform"` for admin-created listings. Phase 2
introduces `"player"` with a `seller.displayName` field for user-to-user trades.

**Caching:** this response is user-agnostic by design and is served from the CloudFront
edge with `Cache-Control: public, s-maxage=60, stale-while-revalidate=300` plus an
`ETag`. Do not add user-specific fields (affordability flags, ownership state, personal
pricing) to this payload — doing so forces the identity dimension into the cache key
and collapses the hit rate to near zero. The client already holds the balance from
`/v1/profile` and can derive affordability locally.

### POST /v1/market/purchase

**Request**
```json
{
  "listingId": "lst_001",
  "quantity": 1
}
```

**Response**
```json
{
  "status": "purchased",
  "item": {
    "id": "sku_xyz789",
    "name": "Flame Shield",
    "quantity": 1
  },
  "cost": {
    "amount": 750,
    "currency": "pts"
  },
  "balance": {
    "points": 2490
  }
}
```

---

## Redeem

### GET /v1/redeem/options

Returns available redemption options for the user's current point balance.
Identity derived from JWT.

**Request**
```
GET {API_BASE_URL}/v1/redeem/options?category=giftcard
```

**Response**
```json
{
  "balance": { "points": 2490 },
  "options": [
    {
      "id": "rdm_gc10",
      "name": "$10 Gift Card",
      "description": "Redeemable at checkout.",
      "cost": 2000,
      "currency": "pts",
      "category": "giftcard",
      "available": true,
      "imageUrl": "https://cdn.example.com/rewards/gc10.webp"
    }
  ]
}
```

### POST /v1/redeem

**Request**
```json
{
  "optionId": "rdm_gc10"
}
```

**Response**
```json
{
  "status": "redeemed",
  "redemption": {
    "id": "rxn_abc",
    "optionId": "rdm_gc10",
    "name": "$10 Gift Card",
    "redeemedAt": "2025-03-02T15:00:00Z",
    "deliveryMethod": "in_app",
    "code": "GIFT-XXXX-XXXX"
  },
  "balance": { "points": 490 }
}
```

---

## Profile

### GET /v1/profile

Identity derived from JWT. No query params required for own profile.

**Request**
```
GET {API_BASE_URL}/v1/profile
```

**Response**
```json
{
  "userId": "usr_xyz",
  "displayName": "Jane Doe",
  "avatarUrl": "https://cdn.example.com/avatars/usr_xyz.webp",
  "balance": { "points": 490 },
  "stats": {
    "totalScans": 22,
    "itemsClaimed": 14,
    "rewardsRedeemed": 3,
    "uniqueMerchants": 6,
    "memberSince": "2024-11-01T00:00:00Z"
  },
  "topMerchants": [
    { "merchantId": "mrc_cafe_01", "merchantName": "Copper Kettle Cafe", "scanCount": 8 }
  ]
}
```

---

## Merchants (Phase 2 — contract defined now)

These endpoints are not yet implemented. They are specified here so that agents
building the data model and authorization middleware account for them.

### GET /v1/merchants/me

Authenticated merchant views own dashboard. Requires `merchant` scope JWT.

**Response**
```json
{
  "merchantId": "mrc_cafe_01",
  "name": "Copper Kettle Cafe",
  "logoUrl": "https://cdn.example.com/merchants/mrc_cafe_01.webp",
  "plan": "starter",
  "analytics": {
    "totalScans": 342,
    "uniqueUsers": 189,
    "scansThisMonth": 47,
    "topItems": [
      { "id": "sku_abc123", "name": "Obsidian Blade", "claimCount": 22 }
    ],
    "repeatVisitRate": 0.34
  },
  "campaigns": [
    {
      "id": "cmp_spring25",
      "name": "Spring Drop Campaign",
      "status": "active",
      "startsAt": "2025-03-01T00:00:00Z",
      "endsAt": "2025-04-01T00:00:00Z",
      "totalCodes": 500,
      "claimedCodes": 187
    }
  ]
}
```

### GET /v1/merchants/me/scans

Paginated scan history for the merchant's codes. Useful for reconciliation
and billing verification.

**Response**
```json
{
  "scans": [
    {
      "scanId": "scn_001",
      "userId": "usr_xyz_hash",
      "itemId": "sku_abc123",
      "campaignId": "cmp_spring25",
      "scannedAt": "2025-03-02T14:22:00Z",
      "claimStatus": "claimed"
    }
  ],
  "pagination": { "page": 1, "limit": 50, "total": 342, "hasNext": true }
}
```

Note: `userId` is a one-way hash, not the raw ID. Merchants see engagement
patterns but cannot identify individual users.

---

## Partner Game API (Phase 3 — contract defined now)

Third-party games read a user's Loot inventory via OAuth2 consent flow.
The user grants a specific game access to their item data. The game receives
a scoped token that can only read items tagged with `usableIn` matching that
game's `partnerId`.

### GET /v1/partner/inventory

Requires `partner` scope token + user consent token.

**Request**
```
GET {API_BASE_URL}/v1/partner/inventory
Authorization: Bearer <partner_token>
x-user-consent: <user_consent_token>
```

**Response**
```json
{
  "userId": "usr_xyz_hash",
  "partnerId": "partner_game_xyz",
  "items": [
    {
      "id": "sku_abc123",
      "name": "Obsidian Blade",
      "rarity": "rare",
      "category": "weapon",
      "attributes": {
        "damage": 120,
        "durability": 80
      },
      "quantity": 1
    }
  ]
}
```

The partner only sees items where `usableIn` includes their `partnerId`.
Attributes, rarity, and category are provided so the partner game can map
Loot items to its own internal systems without a second lookup.

### POST /v1/partner/consume

A partner game reports that an item was consumed (used, destroyed, crafted)
within its system. This decrements quantity in the user's Loot inventory.

**Request**
```json
{
  "itemId": "sku_abc123",
  "quantity": 1,
  "reason": "crafted_into:sku_flame_blade"
}
```

**Response**
```json
{
  "status": "consumed",
  "itemId": "sku_abc123",
  "remainingQuantity": 0
}
```

---

## Domain Events

The system emits events to an internal event bus (EventBridge). These are not
exposed to consumers directly but are documented here because they define the
integration surface for merchants, partner games, analytics, and future
webhooks.

| Event name              | Emitted when                            | Key payload fields                                      |
|-------------------------|-----------------------------------------|---------------------------------------------------------|
| `scan.completed`        | Any successful scan claim               | `userId`, `itemId`, `merchantId`, `campaignId`, `type`  |
| `scan.rejected`         | Scan fails validation                   | `reason`, `merchantId`, `qrId`                          |
| `item.claimed`          | Item added to user inventory            | `userId`, `itemId`, `rarity`, `merchantId`, `source`    |
| `item.consumed`         | Partner game consumes an item           | `userId`, `itemId`, `partnerId`, `reason`               |
| `reward.claimed`        | Points or reward granted                | `userId`, `rewardId`, `value`, `merchantId`             |
| `market.purchase`       | User buys from marketplace              | `userId`, `listingId`, `itemId`, `cost`                 |
| `redeem.completed`      | User redeems points for real-world good | `userId`, `optionId`, `cost`, `deliveryMethod`          |
| `merchant.code_created` | New QR batch generated for merchant     | `merchantId`, `campaignId`, `codeCount`                 |
| `economy.supply_change` | Item supply crosses a threshold         | `itemId`, `totalSupply`, `threshold`, `direction`       |

Events use a consistent envelope:
```json
{
  "eventId": "evt_uuid",
  "eventName": "scan.completed",
  "version": 1,
  "timestamp": "2025-03-02T14:22:01Z",
  "payload": { }
}
```

Phase 2 introduces merchant webhooks: merchants subscribe to events filtered by
their `merchantId`. Phase 3 introduces partner webhooks for `item.consumed` and
`item.claimed` events scoped to their `partnerId`.

---

## Shared Enums

```
rarity:          common | uncommon | rare | epic | legendary
category:        weapon | armor | consumable | cosmetic | badge | material | currency
rewardType:      points | giftcard | discount | digital_item
deliveryMethod:  email | in_app
currency:        pts | shards | dust
scan.status:     claimed | already_claimed | invalid_signature | expired
                 | not_found | registered | merchant_inactive | geofence_rejected
source.type:     merchant_scan | event_checkin | campaign_bonus | marketplace
                 | crafting | partner_grant | card_transaction
seller.type:     platform | player
merchant.plan:   starter | growth | enterprise
tag:             craftable | tradeable | consumable | soulbound | limited_edition
```

### Enum notes

**`category: material | currency`** — Added to support crafting and multi-currency
economies. `material` covers crafting components (ore, fabric, etc.). `currency`
covers non-points resources like `shards` and `dust` that serve as secondary
economies or crafting inputs.

**`currency: shards | dust`** — Reserved for Phase 2 multi-currency. `pts` remains
the primary redeemable-for-real-value currency. `shards` and `dust` are gameplay
resources with no direct cash redemption (avoids money transmitter licensing
triggers). They function as crafting inputs and can be earned via merchant scans
at different rates than points.

**`source.type: card_transaction`** — Reserved for Phase 3 credit card integration.
When a Loot card purchase triggers a reward, the source type distinguishes it
from QR scans for analytics, billing, and economic balancing.

**`tag: soulbound`** — Items marked soulbound cannot be traded or listed on the
marketplace. Used for achievement badges, event check-in proofs, and
anti-exploitation on high-value promotional drops.

---

## Economy Constraints

These are server-enforced rules. Agents building client UI should reflect them
but must not trust client-side enforcement alone.

| Rule                         | Enforcement                                                |
|------------------------------|------------------------------------------------------------|
| One claim per code per user  | Server checks `(userId, qrId)` uniqueness                  |
| Daily scan cap               | 50 scans per user per calendar day (UTC). Configurable per campaign. |
| Item supply caps             | Each item SKU has a `maxSupply`. Server rejects claims when reached. Emits `economy.supply_change`. |
| Trade cooldown               | Newly claimed items cannot be listed on marketplace for 24h (anti-flip). |
| Points expiry                | Points earned from merchant scans expire after 12 months of account inactivity. |
| Soulbound enforcement        | Items tagged `soulbound` are excluded from all trade and transfer endpoints. |

---

## CameraScreen Integration Point

The hand-off from camera to API lives in `CameraScreen.js` at the `handleCapture`
callback. After `takePictureAsync`, the image is passed to a QR decoder (e.g.
`expo-barcode-scanner` or a vision API call), the decoded string is JSON-parsed,
and the resulting object is posted to `POST /v1/scan`. The `TODO` comment in
`CameraScreen.js` marks the exact insertion point.

The client should:
1. Parse the QR JSON and validate shape locally (all required fields present, `t` is a known discriminator). Reject malformed payloads before hitting the network.
2. Attach `scannedAt` (device clock, ISO 8601 UTC) and `location` (if permission granted).
3. POST to `/v1/scan` with the `x-idempotency-key` header set to a UUID generated at scan time. This makes retries safe.
4. Handle response by `type` discriminator to route to the correct success screen.
5. On 429, respect `Retry-After`. On 500, retry once with the same idempotency key.

---

## Roadmap Extensions (not yet in contract)

These are planned capabilities. They are listed here so agents avoid
architectural decisions that would make them hard to add later.

| Extension                  | Phase | Impact on this contract                                     |
|----------------------------|-------|-------------------------------------------------------------|
| Player-to-player trading   | 2     | New `POST /v1/market/list` endpoint, `seller.type: player`, escrow flow |
| Crafting system            | 2     | New `POST /v1/craft` endpoint consuming `material` items per recipe definitions |
| Multi-currency economy     | 2     | `shards`, `dust` currencies in balance objects, marketplace prices in multiple currencies |
| Merchant webhooks          | 2     | Webhook subscription management endpoints under `/v1/merchants/me/webhooks` |
| Card-linked transactions   | 3     | New `source.type: card_transaction`, transaction webhook from banking partner (Marqeta/Lithic) |
| Partner game SDK           | 3     | OAuth consent flow, `/v1/partner/*` endpoints go live       |
| Leaderboards               | 3     | New `GET /v1/leaderboard` with seasonal and all-time views  |
| Guild / team system        | 3     | Shared inventories, group crafting bonuses                   |
| WAF + bot protection       | 2     | Attaches to the existing CloudFront distribution; no contract change |
| Provisioned concurrency    | 2     | `scan` function only, if p99 cold start becomes user-visible |
| Merchant dashboard SPA     | 2     | New S3 origin behavior under the same distribution           |
| Valkey / Redis cache tier  | 3     | Enables leaderboards, shared counters, pub/sub — see [Caching Strategy](#caching-strategy) L4 |
