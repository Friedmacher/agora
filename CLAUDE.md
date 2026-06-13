# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Agora is a personal hospitality location manager — a **software project** (SAP BTP ABAP Environment + SAP Fiori Elements + SAP HANA), not an Obsidian vault task. The vault-level CLAUDE.md (one level up) does not apply here.

## Authoritative References

Read these before making any substantive changes. Together they are the single source of truth.

| File | Contents |
|---|---|
| [`REQUIREMENTS.md`](REQUIREMENTS.md) | Project overview, functional requirements (R-001–R-069), non-functional requirements, out-of-scope, R-ID mapping appendix |
| [`DATA_MODEL.md`](DATA_MODEL.md) | RAP BO hierarchy, HANA table schemas (ABAP types), ABAP domain definitions |
| [`API_SPEC.md`](API_SPEC.md) | OData V4 service bindings, entity sets, operations, custom actions, query options |
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | Component diagram, request flows, security, RAP implementation notes, CDS naming conventions, Fiori floorplan map, repo structure, terminology glossary |

## Development Workflow

### Primary tools

- **ABAP Development Tools (ADT)** in Eclipse — create and edit all ABAP objects, run ATC checks, manage transport requests.
- **abapGit** — sync between the ABAP system and this Git repository (`src/` directory contains abapGit-serialized XML objects).
- **BTP Cockpit** — manage role collection assignments, IAS trust configuration, service instances.
- **IAS Admin Console** — provision admin users, manage user groups and attribute mappings.

### ATC (ABAP Test Cockpit)

Run ATC checks before releasing any transport request:

1. In ADT: right-click the `ZAGORA` package → *Run As* → *ABAP Test Cockpit*.
2. All checks must pass with no errors and no priority-1 warnings (R-069).
3. ABAP Unit tests in behavior implementation classes run as part of ATC.

### abapGit sync

```
Push (ABAP → Git):  In ADT abapGit view, select the ZAGORA repository → Push
Pull (Git → ABAP):  In ADT abapGit view, select the ZAGORA repository → Pull
```

### Transport

1. All changes go into a transport request (automatically created on first object edit in the `ZAGORA` package).
2. Release the request in the Transport Organizer after ATC passes.
3. Import into QA/Production via the BTP cockpit or the CTS import queue.

## Stack

SAP BTP ABAP Environment (managed cloud) + SAP HANA (managed) + SAP Fiori Elements (OData V4 / CDS annotations).

**Module names (use these consistently):**
- **The Codex** — visited location log
- **The Forum** — wish-list
- **The Archive** — dashboard (admin landing page)

## Architecture

### Request path

Browser → OData V4 service binding (`ZAGORA_ADMIN_SRV` or `ZAGORA_PUBLIC_SRV`) → RAP Business Objects (`ZAGORA_BP_LOCATION`, `ZAGORA_BP_ADMIN_USER`) → SAP HANA. All routing is handled by the BTP ABAP Environment's built-in OData gateway — no Nginx, no custom middleware.

### Auth

- Admin access: BTP IAS (SAML 2.0 / OIDC). No application-layer tokens or password hashing.
- Role enforcement: ABAP authorization object `ZAGORA_LOC` checked in RAP behavior implementations. Admin role collection: `Agora_Admin` (assigned in BTP cockpit).
- Public access: anonymous read via `ZAGORA_PUBLIC_SRV`; enforced by CDS access control (DCL).

### Key cross-cutting concerns

| Concern | Where | Detail |
|---|---|---|
| `OverallScore` | `ZAGORA_BP_LOCATION` (MODIFY method) | Application-computed `(price + ambience + quality) / 3`; **not** a HANA generated column. Recalculated and persisted on every MODIFY that changes any rating. Returns null if any rating is unset (initial value). |
| `AdminUserId` on Visit | `ZAGORA_BP_LOCATION` (Create Visit) | Set server-side via `cl_abap_context_info=>get_user_technical_name()`. Never in request body. **Structurally absent** from `ZAGORA_C_VISIT_PUB`. |
| ETag locking | All root + child BOs | `LastChangedAt` is the ETag master. `If-Match` required on all modifying requests; HTTP 412 on mismatch — RAP-enforced. |
| Draft | `Location` BO only | RAP managed draft (`with draft`). Draft tables `ZAGORA_LOC_D` / `ZAGORA_VIS_D` auto-generated. Admin service only. |

## Domain Entities

`ZAGORA_LOCATION`, `ZAGORA_VISIT`, `ZAGORA_ADMIN_USER` — all `sysuuid_x16` PKs, RAP-managed.

Key constraints not obvious from field names:
- `Status`: domain `ZAGORA_D_STATUS` — `VISITED | WISHLIST` (default `WISHLIST`)
- `LocType`: domain `ZAGORA_D_LOC_TYPE` — `RESTAURANT | COFFEEHOUSE | BAR | BAKERY`
- Rating fields (`PriceRating`, `AmbienceRating`, `QualityRating`): `abap.int1`, nullable, CHECK 1–5 in behavior definition
- `AdminUserId` on `ZAGORA_VISIT`: not projected in public CDS view

## Repository Structure

```
<repo-root>/
├── REQUIREMENTS.md
├── DATA_MODEL.md
├── API_SPEC.md
├── ARCHITECTURE.md
├── CLAUDE.md
└── src/
    ├── ZAGORA_DATA/        HANA tables, domains, data elements (abapGit XML)
    └── ZAGORA_SERVICES/    CDS views, behavior definitions, service objects (abapGit XML)
```

See `ARCHITECTURE.md` Section 6 for the full object listing.

## Implementation Status (as of 2026-06-13)

**Design phase — no ABAP objects created yet.** The `src/` directory structure does not exist yet.

### Starting a build session

1. Read all four reference docs in full before touching anything.
2. Create ABAP objects in this order: HANA tables + domains → CDS interface views → behavior definitions → consumption views → service definitions → service bindings → Fiori apps.
3. Follow the naming conventions in `ARCHITECTURE.md` Section 4.3 exactly.
4. Update this section as modules are completed.

### Completed modules

_(none yet)_
