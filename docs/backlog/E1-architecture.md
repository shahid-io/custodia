# E1 Product identity and custody core

**Course module:** 1. Application Architecture (layered architecture, Pydantic V2,
dependency injection, async execution, Dockerization)

This epic builds the core custody loop: define products, register partners, commission
serialized units, pack them into cases and pallets, ship and receive them, and
quarantine suspect lots. Every story is an exercise in keeping the four layers clean.

---

## CUS-101 GS1 identifier value objects

**Points:** 3 · **Depends on:** none · **Concepts:** domain value objects, immutability, pure unit tests

**User story**
As a compliance engineer, I want product identifiers (GTIN, NDC, serial, lot, expiry)
to be validated the moment they are created, so that a malformed identifier can never
enter the system or be sent to a trading partner.

**Why it matters**
A wrong check digit or mis-segmented NDC means a pharmacy scan will fail to match, and
the unit cannot be verified under DSCSA. Catching it in the domain layer means every
entry point (API, CLI, EPCIS import) gets the same protection.

**Acceptance criteria**
- [ ] `GTIN` accepts 8, 12, 13 or 14 digits, normalizes to 14 by left-padding zeros, and rejects a wrong check digit.
- [ ] `NDC` accepts hyphenated 10-digit forms (4-4-2, 5-3-2, 5-4-1), keeps the segment layout, and can produce the 11-digit 5-4-2 form.
- [ ] `GTIN.from_ndc(ndc, indicator=0)` builds the US GTIN-14: indicator digit + `03` + 10-digit NDC + check digit. `gtin.to_ndc10()` reverses it.
- [ ] `SerialNumber` allows 1 to 20 characters from the GS1 AI 82 character set.
- [ ] `ExpiryDate.from_gs1("YYMMDD")` handles day `00`, which means the last day of that month.
- [ ] All value objects are immutable and compare by value.
- [ ] Unit tests cover valid cases, each invalid case and at least one real-world GTIN.

**Technical notes**
- File: `app/domains/products/domain.py`. Use `@dataclass(frozen=True, slots=True)` and validate in `__post_init__`, like the lesson's `Money`.
- GS1 check digit: take the data digits right to left, weight them 3, 1, 3, 1 ..., sum, then `check = (10 - total % 10) % 10`.
- Raise a domain error (`InvalidIdentifierError` in `products/exceptions.py`), not `ValueError`, so the API layer can map it consistently.
- No imports from `fastapi`, `pydantic` or `sqlalchemy` in this file.

---

## CUS-102 Register a drug product

**Points:** 5 · **Depends on:** CUS-101, CUS-002 · **Concepts:** Pydantic V2 schemas, `model_validator`, layered write path

**User story**
As a manufacturer's master data manager, I want to register a drug product with its
NDC, GTIN, name, strength, dosage form and DEA schedule, so that serialized units can
be commissioned against it.

**Acceptance criteria**
- [ ] `POST /api/v1/products` returns `201` with the stored product.
- [ ] Request validation rejects: bad GTIN check digit, malformed NDC, and a GTIN whose embedded NDC does not match the submitted NDC (`422`).
- [ ] `dea_schedule` is optional and limited to `CII`, `CIII`, `CIV`, `CV`.
- [ ] Registering a GTIN that already exists returns `409`.
- [ ] Response never exposes ORM objects; it is a response schema built from the domain entity.

**Technical notes**
- `schemas.py`: use `Annotated[str, AfterValidator(...)]` types that delegate to your CUS-101 value objects, so validation logic is not duplicated.
- Cross-field rule (GTIN embeds NDC) goes in `@model_validator(mode="after")`.
- Try `model_config = ConfigDict(strict=True, extra="forbid")` on requests and note what changes.
- `models.py`: SQLAlchemy 2.0 style (`Mapped[...]`, `mapped_column`). Unique constraint on `gtin`.
- `service.py` raises `ProductAlreadyExistsError`; the router does not catch `IntegrityError` itself.

---

## CUS-103 Read products and central error mapping

**Points:** 3 · **Depends on:** CUS-102 · **Concepts:** exception handlers, consistent error envelope

**User story**
As an API consumer, I want to look up a product by GTIN and receive errors in one
consistent shape, so that my integration can handle failures predictably.

**Acceptance criteria**
- [ ] `GET /api/v1/products/{gtin}` returns `200` or `404`.
- [ ] `GET /api/v1/products?dea_schedule=CII` filters products (simple limit/offset is fine here; cursor pagination comes in CUS-207).
- [ ] Every error response, from any domain, uses one envelope, e.g. `{"error": {"code": "PRODUCT_NOT_FOUND", "message": "...", "request_id": "..."}}`.
- [ ] Routers contain no `try/except` for domain errors.

**Technical notes**
- Give each domain exception a `code` and a default HTTP status by mapping in one place: `app/api/exceptions.py`, registered with `app.add_exception_handler(BaseDomainError, handler)`.
- A shared base class (e.g. `app/core/errors.py: DomainError`) lets one handler cover all domains without the domains importing FastAPI.
- Refactor the lesson pattern: compare this with the lesson's per-route `try/except` and note the tradeoff.

---

## CUS-104 Register trading partners and licenses

**Points:** 5 · **Depends on:** CUS-002 · **Concepts:** aggregates, domain rules with dates

**User story**
As a compliance officer, I want to register trading partners with their type, GLN,
licenses and state registrations, so that the system can refuse transactions with
partners who are not authorized.

**Acceptance criteria**
- [ ] `POST /api/v1/partners` creates a partner of type `MANUFACTURER`, `REPACKAGER`, `WHOLESALER`, `DISPENSER` or `3PL`, identified by a 13-digit GLN (validated with the GS1 check digit from CUS-101).
- [ ] `POST /api/v1/partners/{id}/licenses` adds a license (state or federal, number, issued, expires) and optionally a DEA registration with authorized schedules.
- [ ] Domain method `partner.is_authorized(on: date, state: str | None)` returns `False` if no valid license covers that date (and state, when given).
- [ ] `GET /api/v1/partners/{id}` returns the partner with licenses and an `authorized_today` flag.

**Technical notes**
- Treat `Partner` with its licenses as one aggregate: licenses are only added through the partner service.
- Pass "today" into the domain method instead of calling `date.today()` inside it; this keeps it trivially testable.
- Reuse the GS1 check-digit function from CUS-101 rather than copying it.

---

## CUS-105 Commission serialized units

**Points:** 5 · **Depends on:** CUS-101, CUS-102, CUS-104 · **Concepts:** batch commands, domain events

**User story**
As a packaging line at a manufacturer, I want to commission a batch of serialized units
for a product and lot, so that each physical unit has a unique, traceable identity from
the moment it is made.

**Acceptance criteria**
- [ ] `POST /api/v1/serialization/commission` accepts a GTIN, lot number, expiry and a list of serial numbers (up to 10,000 per request).
- [ ] Creates the lot if it does not exist; rejects if the lot exists with a different expiry.
- [ ] Rejects a serial already commissioned for that GTIN (`409`, listing the duplicates).
- [ ] Each unit starts with status `ACTIVE`, owner = the commissioning manufacturer.
- [ ] One custody event is recorded: type `OBJECT`, action `ADD`, bizStep `commissioning`, disposition `active`.
- [ ] Only partners of type `MANUFACTURER` or `REPACKAGER` may commission.

**Technical notes**
- `serialization/domain.py`: `SerializedUnit` with `status`, `owner_partner_id`, `parent_id` (null for now).
- `custody/domain.py`: an `EpcisEvent` entity with `event_type`, `action`, `biz_step`, `disposition`, `epc_list`, `event_time`, `read_point`.
- The service orchestrates three repositories (products, serialization, custody). For now one session per request is enough; atomicity across repositories is formalized in CUS-203.
- Bulk insert: compare `session.add_all` with `insert().values([...])` for 10,000 rows. Note the timings.

---

## CUS-106 Aggregate units into the packaging hierarchy

**Points:** 8 · **Depends on:** CUS-105 · **Concepts:** tree invariants in the domain, recursive structures

**User story**
As a packaging line operator, I want to record that units were packed into an inner
pack, inner packs into a case and cases onto a pallet, so that scanning a single pallet
label accounts for every unit inside it.

**Acceptance criteria**
- [ ] `POST /api/v1/serialization/aggregate` takes a parent SSCC or SGTIN, its packaging level and a list of child identifiers.
- [ ] Levels are ordered `PALLET > CASE > INNER_PACK > UNIT`; a child must be at a strictly lower level than its parent.
- [ ] A child that already has a parent is rejected unless it was first disaggregated.
- [ ] All children must be `ACTIVE` and owned by the requesting partner.
- [ ] `POST /api/v1/serialization/disaggregate` removes children from a parent.
- [ ] `GET /api/v1/serialization/{id}/contents` returns the full tree under a container, with a total unit count.
- [ ] An `AGGREGATION` event (action `ADD` or `DELETE`, bizStep `packing` / `unpacking`) is recorded.

**Technical notes**
- Pallets and logistics units use an SSCC (18 digits, same GS1 check digit). Add an `SSCC` value object next to `GTIN`.
- Put the hierarchy rules in a pure domain function that receives the parent and children; the service only loads and saves.
- For `contents`, start with a loop of queries. Leave yourself a `# TODO N+1` note: CUS-206 (recursive CTE) and CUS-305 (eager loading) will fix it.

---

## CUS-107 Ship a container to a trading partner

**Points:** 8 · **Depends on:** CUS-104, CUS-106 · **Concepts:** the lesson's transfer flow, row locking, invariant checks

**User story**
As a wholesaler's shipping clerk, I want to ship a pallet or case to an authorized
customer, so that custody of every unit inside moves to the buyer with a record that
satisfies DSCSA transaction information requirements.

**Why it matters**
This is the pharma version of the lesson's ledger transfer. The "balance check" becomes
"every unit inside is active, sellable and in my possession", and the "two ledger
entries" become one shipping event that covers the whole hierarchy.

**Acceptance criteria**
- [ ] `POST /api/v1/custody/shipments` takes sender, receiver, a list of top-level containers and a ship date.
- [ ] Rejected with `422` if: receiver is not authorized on the ship date, any unit is not `ACTIVE`, any unit is expired, any unit's lot is quarantined or recalled, or any unit is not owned by the sender. The error lists the offending identifiers.
- [ ] If the product has a DEA schedule, the receiver must hold a DEA registration covering it (`403`; full ABAC arrives in CUS-403).
- [ ] On success, every unit in the hierarchy moves to status `IN_TRANSIT` and a `shipment` record is created.
- [ ] One `OBJECT` event, bizStep `shipping`, disposition `in_transit`, lists the top-level container IDs (children are implied by aggregation).
- [ ] Response returns the shipment ID and a unit count.

**Technical notes**
- Lock the containers and units being shipped with `SELECT ... FOR UPDATE` in the repository (as in the lesson's `get_account_for_update`).
- Collect *all* violations before raising, so the clerk can fix the whole shipment in one go (a `ShipmentRejectedError` carrying a list).
- Service never builds SQL; add semantic repository methods such as `lock_hierarchy(container_ids)`.

---

## CUS-108 Receive a shipment

**Points:** 5 · **Depends on:** CUS-107 · **Concepts:** reconciliation logic, partial outcomes

**User story**
As a pharmacy receiving clerk, I want to scan an incoming shipment and confirm receipt,
so that ownership transfers to my pharmacy and any discrepancy is flagged immediately.

**Acceptance criteria**
- [ ] `POST /api/v1/custody/shipments/{id}/receive` accepts the list of scanned container identifiers.
- [ ] Only the shipment's receiver can receive it, and only once.
- [ ] Matching containers: units become `ACTIVE`, owner = receiver; a `receiving` event is recorded with disposition `in_progress` then `active` (pick one and justify it).
- [ ] Scanned but not shipped, or shipped but not scanned: the shipment is marked `DISCREPANCY`, missing or unexpected identifiers are listed, and those units are not transferred.
- [ ] Response summarizes received, missing and unexpected counts.

**Technical notes**
- Model reconciliation as a pure function: `(expected: set, scanned: set) -> ReceiptOutcome`. It is the most valuable thing to unit test here.
- Think about status codes: is a discrepancy an error (`4xx`) or a successful request with a `DISCREPANCY` outcome (`200`)? Write your reasoning in the PR description.

---

## CUS-109 Quarantine a suspect lot

**Points:** 5 · **Depends on:** CUS-105 · **Concepts:** the lesson's freeze exercise, state machines

**User story**
As a QA manager, I want to quarantine a lot under suspect-product investigation, so that
no unit from that lot can be shipped or dispensed while we investigate.

**Acceptance criteria**
- [ ] `POST /api/v1/lots/{lot_id}/quarantine` with a reason and a case reference sets the lot status to `QUARANTINED`.
- [ ] Rejected with `422` if the lot is already quarantined or recalled; `404` if it does not exist.
- [ ] `POST /api/v1/lots/{lot_id}/release` returns the lot to `RELEASED` with a disposition note (cleared or confirmed illegitimate).
- [ ] When confirmed illegitimate, the lot becomes `ILLEGITIMATE` and the response includes a `notify_by` timestamp 24 hours after determination.
- [ ] CUS-107 already refuses to ship units of a quarantined lot; add a test proving it.

**Technical notes**
- Model allowed transitions as a small state machine in `lots/domain.py` (a dict of `status -> allowed next statuses` is enough).
- Repository: `update_status(lot_id, status)` without committing, exactly as the lesson exercise asks.
- The audit fields for *who* and *why* matter here; they become a full Part 11 audit trail in CUS-204.

---

## CUS-110 Dependency injection wiring

**Points:** 3 · **Depends on:** CUS-102 · **Concepts:** `Depends`, inversion of control, test overrides

**User story**
As a developer, I want sessions, repositories, services and the current partner provided
by FastAPI dependencies, so that routers stay thin and tests can swap any piece.

**Acceptance criteria**
- [ ] `app/api/dependencies.py` provides `get_session`, one provider per service, and `get_current_partner`.
- [ ] `get_session` is a generator dependency that closes the session after the response.
- [ ] Until auth exists (CUS-401), `get_current_partner` reads an `X-Partner-ID` header and loads the partner (`401` if missing).
- [ ] A unit test overrides a service dependency with a fake and exercises a router with no database.
- [ ] Use `Annotated` aliases (e.g. `SessionDep = Annotated[AsyncSession, Depends(get_session)]`) to keep signatures short.

**Technical notes**
- FastAPI caches a dependency once per request: verify that two services in one request share the same session.
- Keep service constructors framework-free (`def __init__(self, repo: ProductRepository)`). Only the provider functions know about FastAPI.

---

## CUS-111 Request tracing middleware and structured logs

**Points:** 3 · **Depends on:** CUS-001 · **Concepts:** ASGI middleware, `contextvars`, async-safe context

**User story**
As an on-call engineer, I want every request to carry a request ID through every log
line and response, so that I can trace one failing scan across the system.

**Acceptance criteria**
- [ ] Incoming `X-Request-ID` is reused if present, otherwise a UUID is generated; it is returned in the response header.
- [ ] Every log line is JSON and includes `request_id`, `method`, `path`, `status` and `duration_ms`.
- [ ] Logs from inside services include the request ID without passing it as a parameter.
- [ ] The error envelope from CUS-103 includes the request ID.

**Technical notes**
- Write it as a pure ASGI middleware (a class with `async def __call__(self, scope, receive, send)`), then compare with `BaseHTTPMiddleware` and note why pure ASGI is preferred for streaming responses.
- Store the ID in a `contextvars.ContextVar`; a logging filter or `structlog` processor reads it.
- `app/core/logging.py` holds the configuration.

---

## CUS-112 Production container image

**Points:** 3 · **Depends on:** CUS-001 · **Concepts:** multi-stage builds, minimal attack surface

**User story**
As a platform engineer, I want a small, secure container image for Custodia, so that
it can be deployed in a regulated environment that scans images for vulnerabilities.

**Acceptance criteria**
- [ ] Multi-stage `Dockerfile`: a builder stage installs dependencies; the runtime stage contains only the virtual environment and app code.
- [ ] Runs as a non-root user; no compilers or package caches in the final image.
- [ ] Final image is under 200 MB (record the actual size).
- [ ] `docker compose` can run the app container against the CUS-002 Postgres and Redis.
- [ ] `.dockerignore` excludes tests, `.env`, `.git` and caches.

**Technical notes**
- Use `python:3.12-slim` (or similar) for both stages; copy `/opt/venv` from builder to runtime.
- Run with `uvicorn` (or `gunicorn` with uvicorn workers) and decide how many workers; write down why.
- Add a `HEALTHCHECK` or rely on orchestrator probes (CUS-506). Pick one and justify it.
