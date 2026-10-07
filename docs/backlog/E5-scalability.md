# E5 Network and distribution

**Course module:** 5. Scalability and Distributed Systems (caching, task queues and
DLQs, sharding, health probes)

Custodia now has to serve the network: thousands of pharmacies checking units at the
counter, and partners who must receive every event reliably. This epic adds caching,
asynchronous delivery and operational readiness.

---

## CUS-501 Dispensing safety check with cache-aside

**Points:** 5 · **Depends on:** CUS-302, CUS-002 · **Concepts:** cache-aside, TTLs, hot-path design

**User story**
As a pharmacist, I want a near-instant answer to "is this unit safe to dispense?" when
I scan it, so that recalled, quarantined, expired or unknown product never reaches a
patient and the counter queue never waits on the system.

**Acceptance criteria**
- [ ] `GET /api/v1/verify/{gtin}/{serial}?lot=&expiry=` returns `{verified, dispensable, reasons[]}`.
- [ ] Checks: the unit exists, lot and expiry match what was commissioned, the unit is not expired, quarantined, recalled or illegitimate, and it is owned by the requesting dispenser.
- [ ] Product master data and lot status come from Redis using cache-aside; unit ownership comes from the database.
- [ ] p99 latency for cache hits under 10 ms locally (record the measurement and method).
- [ ] Cache outage degrades to database reads, not errors.

**Technical notes**
- Key design: `product:{gtin}`, `lot:{gtin}:{lot}:status`. Serialize with `model_dump_json()` from Pydantic.
- Compare a short TTL against explicit invalidation; CUS-502 adds the invalidation.
- Guard against stampedes on a popular product (a short lock key or request coalescing).

---

## CUS-502 Recall publication with cache invalidation

**Points:** 5 · **Depends on:** CUS-501, CUS-206 · **Concepts:** write-through, invalidation correctness

**User story**
As a QA manager, I want issuing a recall to take effect at every pharmacy counter
immediately, so that no recalled unit is dispensed because a cache was stale.

**Acceptance criteria**
- [ ] `POST /api/v1/recalls` with a class (I, II, III), reason and affected lots (or a raw material lot, expanded downstream via CUS-206 genealogy).
- [ ] Lot status changes and cache updates use write-through: the cache entry is updated after the DB commit succeeds.
- [ ] A test proves a verification check right after the recall returns `dispensable: false`.
- [ ] If the cache update fails, the key is deleted (so the next read falls back to the DB) and a retry is scheduled.
- [ ] A daily import of the FDA recall list (from a file is fine) feeds the same flow.

**Technical notes**
- Order matters: think through "update cache then commit" vs "commit then update cache" and the failure window of each.
- Lot statuses are safety-critical; a write-through set plus a TTL as a safety net is a reasonable design. Justify yours.

---

## CUS-503 Partner webhook subscriptions

**Points:** 3 · **Depends on:** CUS-401 · **Concepts:** subscription model, signed payloads

**User story**
As a trading partner, I want to subscribe a webhook URL to the event types I care about,
so that my systems receive shipping notices and recalls without polling.

**Acceptance criteria**
- [ ] `POST /api/v1/partners/me/webhooks` registers a URL, event types (`shipping`, `receiving`, `recall`, `quarantine`) and returns a signing secret once.
- [ ] HTTPS is required outside local development, and URLs resolving to private IP ranges are rejected (SSRF protection).
- [ ] Partners can list, pause, rotate the secret and delete subscriptions.
- [ ] A test endpoint sends a signed `ping` event.

**Technical notes**
- Signature header: HMAC-SHA256 over `timestamp + "." + body`, so receivers can reject replays.
- Store secrets encrypted at rest (e.g. Fernet with a key from settings).

---

## CUS-504 EPCIS event fan-out with retries and a DLQ

**Points:** 8 · **Depends on:** CUS-503, CUS-505 · **Concepts:** task queues, exponential backoff, dead letter queues

**User story**
As a trading partner, I want every custody event that concerns me delivered reliably,
even when my endpoint is temporarily down, so that my DSCSA records stay complete.

**Acceptance criteria**
- [ ] A worker process (arq or Celery, your choice, justified) delivers webhook events.
- [ ] Failures retry with exponential backoff and jitter, up to a configurable limit (e.g. 8 attempts over about 24 hours).
- [ ] `4xx` responses other than `408` and `429` are not retried; they go straight to the DLQ.
- [ ] Exhausted deliveries land in a dead letter queue with the last error; an admin endpoint lists and replays them.
- [ ] Delivery attempts are stored (status code, latency, attempt number) and visible to the partner.
- [ ] Each delivery carries a stable event ID so partners can deduplicate.

**Technical notes**
- arq is asyncio-native and Redis-backed, so it fits this stack well; it has no built-in DLQ, so you design one (a table is easiest to query and replay).
- Workers reuse services through the UoW; they never import FastAPI.

---

## CUS-505 Transactional outbox

**Points:** 5 · **Depends on:** CUS-203 · **Concepts:** dual-write problem, at-least-once publishing

**User story**
As a platform engineer, I want events published only if the transaction that produced
them committed, so that partners are never notified of a shipment that was rolled back,
and never miss one that committed.

**Acceptance criteria**
- [ ] Services write an `outbox` row in the same UoW transaction as the state change.
- [ ] A relay polls unpublished rows with `FOR UPDATE SKIP LOCKED`, enqueues them and marks them published.
- [ ] Running two relays concurrently publishes each row exactly once from the outbox (delivery to partners remains at-least-once).
- [ ] A test kills the relay mid-batch and shows no events are lost.

**Technical notes**
- Outbox columns: `id`, `aggregate_type`, `aggregate_id`, `event_type`, `payload`, `created_at`, `published_at`.
- Explain in your own words why "commit, then enqueue directly" loses events.

---

## CUS-506 Liveness, readiness and graceful shutdown

**Points:** 3 · **Depends on:** CUS-201, CUS-002 · **Concepts:** orchestrator probes, lifespan

**User story**
As a platform engineer, I want health endpoints that tell an orchestrator when to
restart an instance and when to route traffic to it, so that deploys and failures do not
drop pharmacy requests.

**Acceptance criteria**
- [ ] `GET /livez` checks only that the process can serve a request (no dependencies).
- [ ] `GET /readyz` checks Postgres and Redis with short timeouts and returns `503` with details if either fails.
- [ ] On shutdown, readiness flips to failing first, in-flight requests finish, then pools close.
- [ ] Example Kubernetes probe settings are documented with reasons for each timing.

**Technical notes**
- Explain why checking the database in liveness can cause cascading restarts during a DB blip.
- Use `lifespan` for startup and shutdown; uvicorn's `--timeout-graceful-shutdown` controls the drain window.

---

## CUS-507 Sharding design spike

**Points:** 5 · **Depends on:** CUS-302, CUS-307 · **Concepts:** shard keys, routing, cross-shard queries

**User story**
As an architect, I want a documented sharding strategy with a working prototype, so
that we know how Custodia scales past a single primary before we need it.

**Acceptance criteria**
- [ ] A decision record compares shard keys: manufacturer (GTIN company prefix), current owner partner, and hash of SGTIN. It covers hot spots, cross-shard queries (genealogy, recalls) and resharding.
- [ ] A prototype routes unit reads and writes to one of two Postgres instances by the chosen key.
- [ ] One cross-shard query (e.g. recall impact) is implemented as scatter-gather and its cost is measured.
- [ ] The record names the point at which sharding is worth its complexity, and the alternatives to try first (replicas, partitioning, caching).

**Technical notes**
- Keep routing in the repository layer (an engine chosen per key); services stay unaware of shards.
- Look at Citus as the managed alternative and note what it would change.
