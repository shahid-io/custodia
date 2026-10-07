# Custodia backlog

Custodia is a pharmaceutical supply-chain traceability service. It tracks serialized
drug units from commissioning at the manufacturer, through packaging and shipping
between trading partners, to the pharmacy, and keeps a tamper-evident record of every
custody change. It is shaped by three regulatory frameworks:

- **DSCSA** (Drug Supply Chain Security Act): unit-level serialization, interoperable
  tracing between authorized trading partners, suspect product quarantine.
- **GS1 EPCIS 2.0**: the event standard used to exchange supply-chain events
  (commission, pack, ship, receive) between companies.
- **21 CFR Part 11**: FDA rules for electronic records and signatures (secure,
  time-stamped, unalterable audit trails).

## How to use this backlog

1. Work the epics roughly in order. Each epic maps to one course module.
2. Inside an epic, respect the `Depends on` field. Stories without dependencies can
   be done in any order.
3. For each story: read it, write a short plan, implement it, then tick the
   acceptance criteria. A story is done only when it meets the Definition of Done below.
4. The **Technical notes** point you at the files, patterns and library features
   to use. They deliberately stop short of giving you the code.

| Epic | File | Course module | Stories |
|---|---|---|---|
| E0 Foundation | [E0-foundation.md](E0-foundation.md) | Setup | CUS-001 to CUS-003 |
| E1 Product identity and custody core | [E1-architecture.md](E1-architecture.md) | 1. Application Architecture | CUS-101 to CUS-112 |
| E2 Records, genealogy and integrity | [E2-persistence.md](E2-persistence.md) | 2. Persistence and Migrations | CUS-201 to CUS-208 |
| E3 Scale of data | [E3-performance.md](E3-performance.md) | 3. Database Performance Tuning | CUS-301 to CUS-308 |
| E4 Access, compliance and contention | [E4-security.md](E4-security.md) | 4. Security and Traffic Management | CUS-401 to CUS-408 |
| E5 Network and distribution | [E5-scalability.md](E5-scalability.md) | 5. Scalability and Distributed Systems | CUS-501 to CUS-507 |

Story points use a Fibonacci scale (1, 2, 3, 5, 8, 13). They show relative size, not hours.

## Definition of Done (applies to every story)

- [ ] Code follows the layer rules: routers hold no business logic, services never
      import `fastapi`, repositories own all query construction, domain exceptions
      are mapped to HTTP only at the transport boundary.
- [ ] Unit tests for domain and service logic (no database, fake repositories).
- [ ] Integration or e2e test for anything that touches the database or HTTP.
- [ ] Schema changes ship as an Alembic migration with a working `downgrade()` (from E2 on).
- [ ] New endpoints are visible and correctly documented in `/docs` (OpenAPI).
- [ ] No secrets in code; configuration comes from `app/core/config.py`.

## Domain glossary

| Term | Meaning |
|---|---|
| **GTIN** | Global Trade Item Number. A GS1 product identifier (we normalize to 14 digits, GTIN-14). The last digit is a mod-10 check digit. |
| **NDC** | National Drug Code. The FDA's 10-digit drug identifier in one of three segment layouts (4-4-2, 5-3-2, 5-4-1). Often normalized to 11 digits (5-4-2) for billing. In the US, the NDC is embedded inside the GTIN. |
| **Serial number** | Unique per unit within a GTIN. Up to 20 alphanumeric characters (GS1 AI 21). |
| **SGTIN** | Serialized GTIN: GTIN + serial number. Identifies one physical unit or case. |
| **Product identifier** | DSCSA term for GTIN/NDC + serial + lot + expiry, printed as a 2D DataMatrix barcode. |
| **Lot / batch** | A production run. Recalls are usually issued per lot. |
| **Aggregation** | The packaging parent-child hierarchy: pallet contains cases, case contains inner packs, inner pack contains units. Scanning a parent stands in for all its children. |
| **Commissioning** | Assigning a serial number to a physical unit and declaring it to exist (EPCIS ObjectEvent, action ADD, bizStep `commissioning`). |
| **EPCIS event** | A record of what happened, when, where and why to which identifiers. Types include ObjectEvent, AggregationEvent, TransactionEvent and TransformationEvent. |
| **bizStep / disposition** | EPCIS vocabulary: the business step (`shipping`, `receiving`) and the resulting state of the goods (`in_transit`, `active`, `recalled`). |
| **Trading partner** | Manufacturer, repackager, wholesale distributor, dispenser (pharmacy/hospital) or 3PL. DSCSA allows transactions only between *authorized* trading partners (valid license or registration). |
| **Suspect product** | Product a partner has reason to believe may be counterfeit, diverted, stolen or otherwise unfit. It must be quarantined and investigated. |
| **Illegitimate product** | Suspect product confirmed as illegitimate. FDA and affected partners must be notified within 24 hours. |
| **API batch** | Active Pharmaceutical Ingredient batch. One input to a finished drug lot (genealogy). |
| **DEA schedule** | Controlled substance class, II (highest abuse potential among legal drugs) to V. Handling requires DEA registration covering that schedule. |
| **Cold chain** | Temperature-controlled transport, e.g. 2 to 8 C for refrigerated biologics. A reading outside range is an *excursion*. |
| **Audit trail (Part 11)** | Secure, computer-generated, time-stamped record of who did what and when. Later entries must never obscure earlier ones. |
