# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Agora is a personal hospitality location manager — a **hybrid software project** consisting of:
1. **SAP BTP ABAP Environment backend** (SAP Fiori Elements + SAP HANA) for authenticated admin users
2. **SAP CAP Node.js frontend** for anonymous read-only access

This is not an Obsidian vault task. The vault-level CLAUDE.md (one level up) does not apply here.

## Authoritative References

Read these before making any substantive changes. Together they are the single source of truth.

| File | Contents |
|---|---|
| [`REQUIREMENTS.md`](REQUIREMENTS.md) | Project overview, functional requirements (R-001–R-070), non-functional requirements, out-of-scope, R-ID mapping appendix |
| [`DATA_MODEL.md`](DATA_MODEL.md) | RAP BO hierarchy, HANA table schemas (ABAP types), ABAP domain definitions |
| [`API_SPEC.md`](API_SPEC.md) | OData V4 service bindings, entity sets, operations, custom actions, query options |
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | Component diagram, request flows, security, RAP implementation notes, CDS naming conventions, Fiori floorplan map, repo structure, terminology glossary |

## Development Workflow

### Primary tools

- **ABAP Development Tools (ADT)** in Eclipse — create and edit all ABAP objects, run ATC checks, manage transport requests.
- **abapGit** — sync between the ABAP system and this Git repository (`src/` directory contains abapGit-serialized XML objects).
- **BTP Cockpit** — manage role collection assignments, IAS trust configuration, service instances.
- **IAS Admin Console** — provision admin users, manage user groups and attribute mappings.
- **SAP CAP Development** — Node.js application development for the anonymous frontend (`cap-frontend/` directory).

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

**Backend**: SAP BTP ABAP Environment (managed cloud) + SAP HANA (managed) + SAP Fiori Elements (OData V4 / CDS annotations).

**Frontend**: SAP CAP Node.js application with React SPA for anonymous access, consuming ABAP OData services.

**ABAP package:** `ZAGORA`

**Namespace prefix:** `ZAGR_` (all custom ABAP objects)

**Module names (use these consistently):**
- **Periplus** — visited location log
- **Pothos** — wish-list
- **Pinax** — geographic map of all places (visited + wish-list)
- **Tholos** — admin dashboard (landing page for authenticated users)

## Architecture

### Request path

**Admin users**: Browser (Fiori Elements) → OData V4 service binding (`ZAGR_ADMIN_SRV`) → RAP Business Objects (`ZAGR_BP_LOCATION`, `ZAGR_BP_ADMIN_USER`) → SAP HANA.

**Anonymous users**: Browser → React SPA (served by CAP Node.js app) → CAP API endpoints → OData V4 service binding (`ZAGR_PUBLIC_SRV`) → RAP Business Objects → SAP HANA.

All routing is handled by the BTP ABAP Environment's built-in OData gateway — no Nginx, no custom middleware.

### Auth

- Admin access: BTP IAS (SAML 2.0 / OIDC). No application-layer tokens or password hashing.
- Role enforcement: ABAP authorization object `ZAGR_LOC` checked in RAP behavior implementations. Admin role collection: `Agora_Admin` (assigned in BTP cockpit).
- Anonymous access: no authentication required via `ZAGR_PUBLIC_SRV`; enforced by CDS access control (DCL). CAP application provides the web interface.

### Key cross-cutting concerns

| Concern | Where | Detail |
|---|---|---|
| `OverallScore` | `ZAGR_BP_LOCATION` (MODIFY method) | Application-computed `(price + ambience + quality) / 3`; **not** a HANA generated column. Recalculated and persisted on every MODIFY that changes any rating. Returns null if any rating is unset (initial value). |
| `AdminUserId` on Visit | `ZAGR_BP_LOCATION` (Create Visit) | Set server-side via `cl_abap_context_info=>get_user_technical_name()`. Never in request body. **Structurally absent** from `ZAGR_C_VISIT_PUB`. |
| ETag locking | All root + child BOs | `LastChangedAt` is the ETag master. `If-Match` required on all modifying requests; HTTP 412 on mismatch — RAP-enforced. |
| Draft | `Location` BO only | RAP managed draft (`with draft`). Draft tables `ZAGR_LOC_D` / `ZAGR_VIS_D` / `ZAGR_OPH_D` auto-generated. Admin service only. |

## Domain Entities

`ZAGR_LOCATION`, `ZAGR_VISIT`, `ZAGR_OPENING_HOURS`, `ZAGR_ADMIN_USER` — all `sysuuid_x16` PKs for root/child BOs; `ZAGR_OPENING_HOURS` uses `(LocationId, DayOfWeek)` as key.

Key constraints not obvious from field names:
- `Status`: domain `ZAGR_D_STATUS` — `VISITED | WISHLIST | CLOSED` (default `WISHLIST`)
- `ZipCode`: `abap.char(10)`, optional
- `LocType`: foreign key to customizing table `ZAGR_LOC_TYPE_T` / `ZAGR_LOC_TYPE_T_T` (language-dependent text); default values `RESTAURANT | COFFEEHOUSE | BAR | BAKERY`; extensible by admin without code changes
- `Country`: SAP `LAND1` domain (ISO 3166-1 alpha-2, 2-char country code)
- Rating fields (`PriceRating`, `AmbienceRating`, `QualityRating`): `abap.int1`, nullable, CHECK 1–5 in behavior definition
- `VisitedAt` on `ZAGR_VISIT`: `abap.dats` (calendar date, no time component)
- `Latitude` / `Longitude` on `ZAGR_LOCATION`: `abap.dec(10,7)`, nullable; resolved by auto-geocoding on save via Communication Arrangement, overridable manually; used by Pinax
- `AdminUserId` on `ZAGR_VISIT`: not projected in public CDS view
- `DayOfWeek` on `ZAGR_OPENING_HOURS`: `abap.int1` 1–7 (Mon–Sun); `OpenTime1/CloseTime1/OpenTime2/CloseTime2`: `abap.tims`, all optional; `ClosedToday`: boolean flag
- Closed locations are excluded from Pinax map regardless of coordinate state

## Repository Structure

```
<repo-root>/
├── REQUIREMENTS.md
├── DATA_MODEL.md
├── API_SPEC.md
├── ARCHITECTURE.md
├── CLAUDE.md
├── src/                abapGit-serialized ABAP objects (populated after first ADT push)
│   ├── ZAGR_DATA/      HANA tables, domains, data elements
│   └── ZAGR_SERVICES/  CDS views, behavior definitions, service objects
└── cap-frontend/       SAP CAP Node.js + React SPA (anonymous access)
    ├── package.json
    ├── mta.yaml        BTP Cloud Foundry deployment descriptor
    ├── srv/            CAP service layer (proxy to ZAGR_PUBLIC_SRV)
    └── app/            React SPA (HTML/CSS/JS)
```

See `ARCHITECTURE.md` Section 6 for the full abapGit object listing.

## Implementation Status

All specification documents are finalized. ABAP implementation is performed directly in ADT — `src/` is populated by abapGit after objects are created in the ABAP development system. There are no template files to copy from this repository.
