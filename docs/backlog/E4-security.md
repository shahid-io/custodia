# E4 Access, compliance and contention

**Course module:** 4. Security and Traffic Management (OAuth2 and JWT, RBAC,
rate limiting, idempotency, optimistic locking, deadlock avoidance)

Custodia handles controlled substances and regulated records. This epic decides who can
do what, protects the API from abusive or duplicated traffic, and keeps inventory
correct when many buyers compete for the same stock.

---

## CUS-401 OAuth2 authentication with signed JWTs

**Points:** 5 · **Depends on:** CUS-110 · **Concepts:** OAuth2 flows, JWT signing, stateless auth

**User story**
As a security officer, I want every API call authenticated with a short-lived signed
token, so that only known users and partner systems can read or change custody records.

**Acceptance criteria**
- [ ] Users belong to a partner; passwords are hashed with Argon2 (or bcrypt).
- [ ] `POST /api/v1/auth/token` supports the password flow for users and the client-credentials flow for partner integrations.
- [ ] Access tokens are RS256-signed, expire in 15 minutes and carry `sub`, `partner_id`, `roles` and `jti`.
- [ ] Refresh tokens are rotated on use; a reused refresh token revokes the whole token family.
- [ ] `get_current_partner` from CUS-110 now derives the partner from the token; the `X-Partner-ID` header is removed.

**Technical notes**
- `OAuth2PasswordBearer` for the docs UI; verify with `PyJWT` (or `joserfc`) using the public key.
- Keep key loading in `app/core/security.py`; never log tokens.
- Explain why RS256 suits a multi-service setup better than HS256.

---

## CUS-402 Role-based access control

**Points:** 3 · **Depends on:** CUS-401 · **Concepts:** dependency-based authorization

**User story**
As a compliance officer, I want actions restricted by role, so that, for example, a
warehouse operator can ship but cannot release a quarantined lot.

**Acceptance criteria**
- [ ] Roles: `WAREHOUSE_OPERATOR`, `QA_MANAGER`, `COMPLIANCE_OFFICER`, `MASTER_DATA_MANAGER`, `PARTNER_INTEGRATION`, `AUDITOR`.
- [ ] A `require_roles(*roles)` dependency guards each endpoint; denial returns `403` with the error envelope.
- [ ] A permission matrix (endpoint x role) is documented and covered by a parametrized test.
- [ ] Users can only act on records of their own partner unless they are `AUDITOR` (read-only, cross-partner).

**Technical notes**
- Use `Security(...)` with scopes or a plain dependency factory; compare both.
- Ownership checks belong in the service (it is a business rule); role checks belong at the transport boundary.

---

## CUS-403 Attribute-based access for controlled substances

**Points:** 8 · **Depends on:** CUS-402, CUS-104 · **Concepts:** ABAC policies, pure policy functions

**User story**
As a DEA compliance officer, I want access to Schedule II to V products decided by the
user's and partner's registrations, not just their role, so that no one handles a
controlled substance their registration does not cover.

**Acceptance criteria**
- [ ] Policy inputs: product schedule, partner DEA registration (schedules, expiry, registered state), destination state registration, user role, action.
- [ ] Shipping a Schedule II product requires the receiver's DEA registration to cover CII and a reference to the ordering record (DEA Form 222 or CSOS order number).
- [ ] Viewing serial-level data for controlled products is limited to roles with a business need.
- [ ] Every allow and deny decision for controlled products is written to the audit trail with the rule that decided it.
- [ ] The policy is a pure function with exhaustive unit tests (table-driven).

**Technical notes**
- Shape: `def evaluate(subject, resource, action, context) -> Decision(allowed, rule_id, reason)` in a new `app/domains/compliance/` package.
- Keep policy rules as data (a list of rule objects) so adding a state-specific rule does not change control flow.

---

## CUS-404 Electronic signatures for critical actions

**Points:** 5 · **Depends on:** CUS-401, CUS-204 · **Concepts:** Part 11 e-signatures, step-up auth

**User story**
As a QA manager, I want to sign lot release and illegitimate-product determinations
with an electronic signature, so that the record meets 21 CFR Part 11 signature
requirements.

**Acceptance criteria**
- [ ] Signing requires re-entering both user ID and password at the time of signing, even with a valid session.
- [ ] The signature records printed name, date and time (UTC) and meaning (`APPROVED`, `REVIEWED`, `RESPONSIBLE`).
- [ ] The signature is linked to the exact record version it signs (store the record's hash), so it cannot be moved to another record.
- [ ] CUS-109 release and illegitimate confirmation require a signature.
- [ ] Signatures appear in the audit trail and in the lot's API response.

**Technical notes**
- A signing endpoint returns a short-lived, single-use signing token tied to the action and record; the action endpoint consumes it.
- Keep signature manifestation (what is shown to a human) separate from signature storage.

---

## CUS-405 Distributed rate limiting

**Points:** 5 · **Depends on:** CUS-401, CUS-002 · **Concepts:** sliding window, Redis atomicity

**User story**
As a platform engineer, I want each partner limited to a fair request rate across all
API instances, so that one misbehaving integration cannot degrade service for others.

**Acceptance criteria**
- [ ] Sliding-window limit per partner (and a stricter one per IP for `/auth/token`).
- [ ] Limits are configurable per partner tier.
- [ ] Over-limit requests get `429` with `Retry-After` and `X-RateLimit-Remaining` headers.
- [ ] The limit holds when running two app instances against the same Redis.
- [ ] If Redis is unavailable, the limiter fails open and logs a warning (justify this choice for a pharma system, or argue for failing closed).

**Technical notes**
- Sorted set per key: remove entries older than the window, add the current timestamp, count, set TTL. Run it as one Lua script (`redis.asyncio` `register_script`) so it is atomic.
- Implement as a dependency or middleware; compare where each fits.

---

## CUS-406 Idempotency keys for event submission

**Points:** 5 · **Depends on:** CUS-107, CUS-002 · **Concepts:** exactly-once effects over at-least-once delivery

**User story**
As a partner integration, I want to safely retry a shipment or EPCIS submission after a
timeout, so that a network glitch never records the same shipment twice.

**Acceptance criteria**
- [ ] `POST` endpoints for shipments, receipts, commissioning and EPCIS accept an `Idempotency-Key` header (required for partner integrations).
- [ ] First request: key claimed atomically, request processed, response stored with a 24-hour TTL.
- [ ] Repeat with same key and same body: the stored response is replayed with an `Idempotent-Replayed: true` header.
- [ ] Repeat with same key but different body: `422`.
- [ ] Repeat while the first is still processing: `409`.

**Technical notes**
- Claim with `SET idem:{partner}:{key} <state> NX EX 86400`; store a hash of the canonical body to detect mismatches.
- Think about the window where the DB commit succeeds but storing the response fails. Write down what your design does in that case.

---

## CUS-407 Optimistic locking on inventory allocation

**Points:** 8 · **Depends on:** CUS-203 · **Concepts:** version columns, retry policies

**User story**
As a wholesaler, I want concurrent fulfillment orders from competing hospital networks
to allocate from the same inventory pool without overselling, so that two hospitals are
never promised the same last cases.

**Acceptance criteria**
- [ ] A new `inventory` domain: pools per product, lot and warehouse with `on_hand`, `allocated` and a `version` column.
- [ ] `POST /api/v1/inventory/allocations` allocates a quantity for an order; `available = on_hand - allocated` can never go negative.
- [ ] Concurrent allocations against one pool: losers get a stale-data conflict, retried up to 3 times with jittered backoff, then `409`.
- [ ] A concurrency test fires 50 parallel allocations for a pool with stock for 10 and proves exactly 10 succeed.
- [ ] A write-up compares this with the pessimistic `FOR UPDATE` approach used in CUS-107, and when each fits.

**Technical notes**
- SQLAlchemy: `__mapper_args__ = {"version_id_col": version}`; a stale update raises `StaleDataError`.
- Retry belongs in the service (or a decorator on it), never in the router, and each retry needs a fresh UoW.

---

## CUS-408 Deadlock-safe multi-container shipments

**Points:** 5 · **Depends on:** CUS-107 · **Concepts:** lock ordering, lock timeouts, SKIP LOCKED

**User story**
As a shipping clerk, I want simultaneous shipments that share containers to fail fast
and clearly, so that warehouse operations never stall on database deadlocks.

**Acceptance criteria**
- [ ] A test reproduces a deadlock by shipping containers `[A, B]` and `[B, A]` concurrently under the original CUS-107 code.
- [ ] The fix acquires row locks in a deterministic order (sorted IDs) and the test passes reliably.
- [ ] `lock_timeout` is set for shipping transactions; timeouts return `409` with a "container busy" message.
- [ ] A pick-list worker endpoint uses `FOR UPDATE SKIP LOCKED` to hand different pending orders to different workers.

**Technical notes**
- `SET LOCAL lock_timeout = '2s'` inside the transaction.
- Postgres reports deadlocks as `DeadlockDetectedError` (asyncpg); map it in the exception handler too, as a safety net.
