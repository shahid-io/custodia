# E2 Records, genealogy and integrity

**Course module:** 2. Persistence and Migrations (async SQLAlchemy, connection pooling,
Alembic, Unit of Work, pagination, Testcontainers)

Regulators care less about speed and more about whether records are complete, atomic
and unalterable. This epic hardens the data layer around those guarantees.

---

## CUS-201 Async engine and pool tuning

**Points:** 3 · **Depends on:** CUS-002 · **Concepts:** asyncpg, pool sizing, connection limits

**User story**
As a platform engineer, I want database connection pooling configured from settings,
so that a burst of scanner traffic from hundreds of warehouses cannot exhaust
PostgreSQL connections.

**Acceptance criteria**
- [ ] `pool_size`, `max_overflow`, `pool_timeout`, `pool_recycle` and `pool_pre_ping` are configurable via settings.
- [ ] A short write-up (in the PR or `docs/decisions/`) calculates: workers x (pool_size + max_overflow) must stay below Postgres `max_connections` minus a reserve.
- [ ] A load test (e.g. 200 concurrent requests with `httpx` or `locust`) with a deliberately tiny pool shows `pool_timeout` errors surfacing as `503`, not hangs.
- [ ] Statement timeout is set at the connection level so a runaway query cannot hold a connection forever.

**Technical notes**
- `create_async_engine(url, pool_size=..., max_overflow=..., pool_timeout=..., pool_pre_ping=True)`.
- asyncpg server settings: `connect_args={"server_settings": {"statement_timeout": "5000"}}`.
- Map SQLAlchemy's pool `TimeoutError` to `503` in the exception handler from CUS-103.

---

## CUS-202 Alembic migrations with async support

**Points:** 3 · **Depends on:** CUS-201 · **Concepts:** schema versioning, reversible migrations

**User story**
As a release manager, I want every schema change versioned and reversible, so that
validated environments can be upgraded and rolled back in a controlled way.

**Acceptance criteria**
- [ ] `alembic init` with the async template; `env.py` reads the URL from `Settings`, not `alembic.ini`.
- [ ] A shared `MetaData(naming_convention=...)` so constraints and indexes get deterministic names.
- [ ] Initial migration creates all tables built so far (products, partners, licenses, lots, units, events, shipments).
- [ ] `alembic upgrade head`, `alembic downgrade base`, `alembic upgrade head` all succeed on an empty database.
- [ ] Autogenerate is used, but every migration is reviewed and hand-edited where needed.

**Technical notes**
- `alembic init -t async migrations`. Import every domain's `models.py` in `env.py` so autogenerate sees them.
- Write down which changes autogenerate misses (e.g. enum value changes, server defaults) so you know to check them.

---

## CUS-203 Unit of Work across repositories

**Points:** 5 · **Depends on:** CUS-107, CUS-202 · **Concepts:** UoW pattern, transaction boundaries, atomic rollback

**User story**
As a compliance officer, I want a shipment to either fully succeed or leave no trace,
so that we never record units as in transit without the matching shipping event.

**Acceptance criteria**
- [ ] A `UnitOfWork` exposes the repositories (`products`, `serialization`, `custody`, `lots`, `partners`) sharing one session.
- [ ] Used as `async with uow:`: commits on clean exit, rolls back on any exception.
- [ ] Repositories never call `commit()`; services never touch the session directly.
- [ ] A test forces a failure after unit status updates but before the event insert, and proves nothing was persisted.
- [ ] CUS-105, CUS-107, CUS-108 and CUS-109 are refactored onto the UoW.

**Technical notes**
- Put the UoW in `app/core/uow.py` (or `app/domains/uow.py`). Inject a UoW factory through a dependency.
- Decide explicitly: does the service commit (`await uow.commit()`) or does the context manager? Compare with the lesson's "check your understanding" answer.
- An `InMemoryUnitOfWork` with fake repositories makes service unit tests fast.

---

## CUS-204 Hash-chained audit trail (21 CFR Part 11)

**Points:** 8 · **Depends on:** CUS-203 · **Concepts:** append-only tables, cryptographic hashing, DB-level enforcement

**User story**
As a QA auditor, I want every custody and status change recorded in an append-only,
cryptographically chained audit trail, so that any retroactive alteration of a record
is detectable during an FDA inspection.

**Why it matters**
Part 11 requires secure, computer-generated, time-stamped audit trails that record who
did what and when, where changes never obscure earlier entries. A hash chain makes
tampering evident: changing any old row breaks every hash after it.

**Acceptance criteria**
- [ ] Table `audit_log`: `seq` (bigserial), `occurred_at` (server time, UTC), `actor_user_id`, `actor_partner_id`, `action`, `entity_type`, `entity_id`, `before` and `after` (JSONB), `reason`, `prev_hash`, `hash`.
- [ ] `hash = sha256(prev_hash + canonical_json(record))`; the first row uses a fixed genesis value.
- [ ] Every state-changing service call writes its audit rows in the same UoW transaction.
- [ ] The database rejects `UPDATE` and `DELETE` on `audit_log` (trigger raising an exception, plus revoking privileges from the app role).
- [ ] Concurrent writers cannot fork the chain (two rows with the same `prev_hash`).

**Technical notes**
- Canonical JSON: `json.dumps(obj, sort_keys=True, separators=(",", ":"), default=str)`. Decimals and datetimes must serialize identically every time.
- Chain serialization: take a transaction-scoped advisory lock (`pg_advisory_xact_lock`) before reading the last hash, or keep the tip in a single-row table locked `FOR UPDATE`. Measure the throughput cost.
- The trigger and `REVOKE` go in an Alembic migration with `op.execute(...)`, including the downgrade.

---

## CUS-205 Audit trail integrity verification

**Points:** 3 · **Depends on:** CUS-204 · **Concepts:** streaming large reads, CLI reuse of services

**User story**
As a QA auditor, I want to verify the integrity of the audit trail on demand, so that I
can prove to an inspector that no record has been altered.

**Acceptance criteria**
- [ ] `GET /api/v1/audit/verify?from_seq=&to_seq=` recomputes the chain and returns `{valid, checked, first_broken_seq}`.
- [ ] The same service runs from a CLI command (`python -m app.cli verify-audit`) with no HTTP involved.
- [ ] Verification streams rows in batches; memory stays flat for a million rows.
- [ ] A test tampers with a row (bypassing the trigger as superuser in the test) and the check reports the exact `seq`.

**Technical notes**
- `await session.stream(select(...).execution_options(yield_per=1000))`.
- The CLI path proves the lesson's point: services with zero HTTP awareness are reusable.

---

## CUS-206 Lot genealogy and recall impact

**Points:** 8 · **Depends on:** CUS-202 · **Concepts:** self-referencing relations, recursive CTEs

**User story**
As a QA manager, I want to trace a finished drug lot back to its API batches and raw
material suppliers, and forward from a raw material to every finished lot that used it,
so that I can scope a recall within minutes instead of days.

**Acceptance criteria**
- [ ] Lots have a `kind`: `RAW_MATERIAL`, `API_BATCH`, `INTERMEDIATE`, `FINISHED`. Raw materials record a supplier partner.
- [ ] `POST /api/v1/lots/{id}/inputs` links input lots with quantity and unit of measure; cycles are rejected.
- [ ] `GET /api/v1/lots/{id}/genealogy?direction=upstream|downstream&max_depth=` returns the tree with depth per node.
- [ ] Downstream trace of a raw material lists affected finished lots and how many serialized units of each are still `ACTIVE`, grouped by current owner.
- [ ] Each trace runs as a single SQL query.

**Technical notes**
- Edge table `lot_inputs(output_lot_id, input_lot_id, quantity, uom)`.
- SQLAlchemy recursive CTE: `select(...).cte(recursive=True)` then `.union_all(...)`. Track the path in an array column to detect cycles and cap depth.
- Reuse this CTE technique to replace the N+1 loop in CUS-106 `contents`.

---

## CUS-207 Cursor pagination for custody history

**Points:** 5 · **Depends on:** CUS-202 · **Concepts:** keyset pagination, opaque cursors

**User story**
As an investigator, I want to page through the full custody history of a unit, lot or
partner, so that I can review millions of events without the API slowing down on later
pages.

**Acceptance criteria**
- [ ] `GET /api/v1/custody/events?sgtin=|lot_id=|partner_id=&limit=&cursor=` returns `{items, next_cursor}`.
- [ ] Ordering is `(event_time DESC, id DESC)`; the cursor encodes the last row's pair.
- [ ] The cursor is opaque (base64 JSON) and tampering returns `400`, not a server error.
- [ ] Response time for page 1 and page 1,000 is roughly the same on a large dataset (verify after CUS-301).
- [ ] An index supports each filter plus the sort order.

**Technical notes**
- Row value comparison: `where(tuple_(Event.event_time, Event.id) < (t, id))`.
- Write a small generic helper in `app/core/pagination.py` so other list endpoints reuse it.
- Contrast with `OFFSET`: explain why page 1,000 with offset reads and discards 999 pages of rows.

---

## CUS-208 Integration tests with Testcontainers

**Points:** 5 · **Depends on:** CUS-202, CUS-003 · **Concepts:** hermetic tests, transactional fixtures

**User story**
As a developer, I want integration tests to run against a disposable real PostgreSQL,
so that tests are deterministic and exercise real SQL (locks, triggers, CTEs).

**Acceptance criteria**
- [ ] A session-scoped fixture starts Postgres via Testcontainers and runs `alembic upgrade head`.
- [ ] Each test runs inside a transaction (or savepoint) that is rolled back afterwards.
- [ ] Integration tests exist for: the audit trigger rejecting updates (CUS-204), the genealogy CTE (CUS-206) and keyset pagination (CUS-207).
- [ ] An e2e test drives commission, aggregate, ship and receive through HTTP against the container.
- [ ] `pytest tests/integration` works on a clean machine with only Docker installed.

**Technical notes**
- `testcontainers[postgres]`; convert its sync URL to `postgresql+asyncpg://`.
- Bind sessions to a connection with an outer transaction and use `join_transaction_mode="create_savepoint"` so code that commits does not escape the rollback.
- Override `get_session` / the UoW factory in `app.dependency_overrides` for e2e tests.
