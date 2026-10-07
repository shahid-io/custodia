# E3 Scale of data

**Course module:** 3. Database Performance Tuning (storage and indexing, query
profiling, N+1 elimination, read replicas, partitioning)

A national distributor handles hundreds of millions of serialized units and billions of
events. This epic makes Custodia stay fast at that size. Always measure before and after:
each story's PR should include `EXPLAIN (ANALYZE, BUFFERS)` output.

---

## CUS-301 Realistic data generator

**Points:** 3 · **Depends on:** CUS-202 · **Concepts:** bulk loading, representative data

**User story**
As a developer, I want to load a realistic large dataset locally, so that performance
work is based on real query plans instead of plans for empty tables.

**Acceptance criteria**
- [ ] `python -m app.cli seed --units 5000000` generates products, partners, lots, aggregated units, shipping and receiving events, and audit rows.
- [ ] Data is skewed realistically: a few high-volume products, most units at dispensers, a small percentage quarantined or recalled.
- [ ] Loading uses `COPY` (asyncpg `copy_records_to_table`) and finishes 5 million units in minutes, not hours.
- [ ] `ANALYZE` runs at the end so the planner has fresh statistics.

**Technical notes**
- Generate valid GTINs and serials with your CUS-101 value objects, so the dataset passes your own validation.
- Seed with a fixed random seed so benchmarks are repeatable.

---

## CUS-302 Index strategy for traceability queries

**Points:** 5 · **Depends on:** CUS-301 · **Concepts:** B-tree, composite and partial indexes, page internals

**User story**
As a pharmacy system, I want unit lookups by SGTIN and lot status checks to be
near-instant, so that dispensing is never delayed by a slow traceability check.

**Acceptance criteria**
- [ ] Unique B-tree index on `(gtin, serial)` serves single-unit lookups with an Index Scan.
- [ ] Composite index on units `(lot_id, status)` serves "active units in lot" queries.
- [ ] Partial index on units `WHERE status IN ('QUARANTINED', 'RECALLED')` and a comparison of its size against a full index.
- [ ] Unused or duplicate indexes are identified with `pg_stat_user_indexes` and removed.
- [ ] A short write-up explains index column order and why `(lot_id, status)` differs from `(status, lot_id)` for these queries.

**Technical notes**
- Inspect sizes with `pg_relation_size` and `pg_indexes_size`. Look at a page with the `pageinspect` extension if you want to see real B-tree pages.
- Consider `INCLUDE` columns for index-only scans and check `Heap Fetches` in the plan.

---

## CUS-303 Store and search EPCIS documents in JSONB

**Points:** 5 · **Depends on:** CUS-301 · **Concepts:** JSONB, GIN indexes, operator classes

**User story**
As an integration engineer, I want the original EPCIS 2.0 documents received from
partners stored and searchable, so that we can answer "show every event from partner X
that mentions this SGTIN" during an investigation.

**Acceptance criteria**
- [ ] `POST /api/v1/custody/epcis` accepts an EPCIS 2.0 JSON-LD document, stores it in a JSONB column and extracts events into the normalized tables.
- [ ] Search endpoint filters by `bizStep`, `disposition` and an EPC contained in `epcList`.
- [ ] GIN index with `jsonb_path_ops` makes containment queries (`@>`) use the index; before and after plans are recorded.
- [ ] A comparison of `jsonb_ops` vs `jsonb_path_ops` size and supported operators.

**Technical notes**
- Containment: `payload @> '{"epcList": ["urn:epc:id:sgtin:..."]}'`. In SQLAlchemy, `Model.payload.contains({...})`.
- For one hot field, compare a GIN index against a B-tree expression index on `(payload->>'bizStep')`.
- Keep the raw document unmodified: it is evidence. Normalized rows are derived data.

---

## CUS-304 Query profiling workflow

**Points:** 3 · **Depends on:** CUS-301 · **Concepts:** EXPLAIN ANALYZE, pg_stat_statements

**User story**
As an on-call engineer, I want a repeatable way to find and diagnose the slowest
queries, so that performance regressions are caught with evidence, not guesses.

**Acceptance criteria**
- [ ] `pg_stat_statements` enabled in the local Compose Postgres.
- [ ] A dev-only endpoint or CLI lists the top 10 queries by total time.
- [ ] Slow SQL (over a configurable threshold) is logged with the request ID from CUS-111.
- [ ] Three real slow queries are diagnosed in a write-up: plan before, cause (seq scan, bad row estimate, sort spill), fix, plan after.

**Technical notes**
- Use SQLAlchemy `before_cursor_execute` / `after_cursor_execute` events on the sync engine (`engine.sync_engine`) to time statements.
- Read plans bottom-up; compare `rows=` estimated vs actual to spot stale statistics.

---

## CUS-305 Eliminate N+1 on shipment details

**Points:** 3 · **Depends on:** CUS-108 · **Concepts:** selectinload, joinedload, query counting

**User story**
As a receiving clerk, I want a shipment's detail page (containers, contents, products,
lots) to load quickly, so that receiving a large pallet is not slowed down by the system.

**Acceptance criteria**
- [ ] `GET /api/v1/custody/shipments/{id}` returns containers with nested contents, product names and lot expiry.
- [ ] A test asserts the endpoint issues a fixed number of queries regardless of how many containers are shipped.
- [ ] A write-up explains when you chose `selectinload` vs `joinedload` for each relationship.
- [ ] `lazy="raise"` is set on relationships so accidental lazy loads fail loudly in async code.

**Technical notes**
- Count queries with a `before_cursor_execute` listener inside a fixture.
- `joinedload` suits many-to-one (unit to product); `selectinload` suits one-to-many collections.

---

## CUS-306 Route reads to a replica

**Points:** 5 · **Depends on:** CUS-201 · **Concepts:** read/write splitting, replication lag

**User story**
As a platform engineer, I want read-heavy traffic (lookups, history, genealogy) served
from a read replica, so that writes from packaging lines are not slowed by investigative
queries.

**Acceptance criteria**
- [ ] A second engine for the replica, configured by settings; Compose runs a streaming replica.
- [ ] Read-only dependencies (`ReadSessionDep`) use the replica; the UoW always uses the primary.
- [ ] Reads that must see a just-made write (e.g. right after commissioning) go to the primary. Document how you decide.
- [ ] If the replica is down, reads fall back to the primary and a warning is logged.

**Technical notes**
- Use the `bitnami/postgresql` image or Postgres native streaming replication in Compose.
- Measure lag with `pg_last_xact_replay_timestamp()` on the replica.

---

## CUS-307 Cold-chain telemetry ingestion

**Points:** 8 · **Depends on:** CUS-107, CUS-202 · **Concepts:** declarative partitioning, write throughput

**User story**
As a logistics provider, I want temperature sensors in shipping containers to stream
readings into Custodia, so that every temperature excursion during transit is recorded
against the shipment and the affected units.

**Acceptance criteria**
- [ ] A new `telemetry` domain. `POST /api/v1/telemetry/readings` accepts batches of `{container_id, sensor_id, recorded_at, temperature_c}`.
- [ ] `telemetry_readings` is range-partitioned by `recorded_at` (monthly); optionally sub-partitioned by hash of `container_id`.
- [ ] Ingestion sustains at least 5,000 readings per second locally (record the measured number).
- [ ] Each product has a storage range (e.g. 2 to 8 C); readings outside it create an `excursion` record linked to the shipment, with duration.
- [ ] A query for one container's readings in a time window shows partition pruning in its plan.

**Technical notes**
- Alembic cannot autogenerate partitioned tables; write the DDL with `op.execute`.
- Primary key on a partitioned table must include the partition key: `(container_id, recorded_at, sensor_id)`.
- Excursion detection: compare running per-request in the service vs a periodic SQL job; pick one and justify it.

---

## CUS-308 Partition lifecycle management

**Points:** 3 · **Depends on:** CUS-307 · **Concepts:** partition maintenance, retention

**User story**
As a DBA, I want future partitions created automatically and old ones archived, so that
telemetry ingestion never fails at month rollover and storage stays bounded.

**Acceptance criteria**
- [ ] A maintenance command creates partitions for the next 3 months if missing; it is safe to run repeatedly.
- [ ] A default partition catches out-of-range rows, and an alert is logged if it is not empty.
- [ ] Partitions older than the retention period are detached (not dropped) and can be exported.
- [ ] Retention for custody records is at least 6 years (DSCSA); explain why telemetry may differ.

**Technical notes**
- `ALTER TABLE ... DETACH PARTITION ... CONCURRENTLY` avoids blocking writers (PostgreSQL 14+).
- Compare a hand-written job against the `pg_partman` extension.
