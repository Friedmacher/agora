# Agora — Architecture

> **Version:** 2.0 | **Status:** AUTHORITATIVE

---

## 1. System Overview

### Component Diagram

```
Browser (SAP Fiori Elements app — admin or public)
        │ HTTPS
        ▼
┌──────────────────────────────────────────────────┐
│             SAP BTP ABAP Environment              │
│                                                  │
│  ┌───────────────────────────────────────────┐   │
│  │           OData V4 Service Bindings       │   │
│  │  ZAGORA_ADMIN_SRV  │  ZAGORA_PUBLIC_SRV   │   │
│  └─────────────────┬─────────────────────────┘   │
│                    │                             │
│  ┌─────────────────▼─────────────────────────┐   │
│  │           RAP Business Objects            │   │
│  │  ZAGORA_BP_LOCATION (root + visits)       │   │
│  │  ZAGORA_BP_ADMIN_USER                     │   │
│  └─────────────────┬─────────────────────────┘   │
│                    │                             │
│  ┌─────────────────▼─────────────────────────┐   │
│  │              SAP HANA (managed)           │   │
│  │  ZAGORA_LOCATION · ZAGORA_VISIT           │   │
│  │  ZAGORA_ADMIN_USER                        │   │
│  └───────────────────────────────────────────┘   │
│                                                  │
│  BTP Identity Authentication Service (IAS)       │
│  ←──── All admin authentication and sessions     │
└──────────────────────────────────────────────────┘
```

### Service Boundaries

- The RAP framework is the **sole** layer that reads and writes SAP HANA. There is no separate API process or middleware tier.
- SAP Fiori Elements apps are metadata-driven and generated from CDS annotations. No custom SAPUI5 controller code is required for standard List Report / Object Page flows.
- SAP BTP IAS manages all authentication and session tokens. No application code handles passwords, tokens, or session state.
- There is no file storage layer. Image handling is out of scope for v2.

---

## 2. Request Data Flows

**1. Admin creates a location**

Fiori Elements List Report (Create button) → OData POST to `ZAGORA_ADMIN_SRV/.../Location` → RAP framework validates ETag prerequisites → routes to `MODIFY` method of `ZAGORA_BP_LOCATION` → behavior implementation validates required fields, checks authorization object `ZAGORA_LOC` activity `02` (Create) → inserts row into `ZAGORA_LOCATION` → OData response `201 Created` with the new entity including RAP-managed `LocationUuid`, `CreatedAt`, `LastChangedAt` (ETag value).

**2. Admin updates ratings (overall_score recomputation)**

Fiori Elements Object Page (Save after editing ratings) → OData PATCH on `Location` entity → `If-Match` ETag header required → RAP `MODIFY` method detects that `PriceRating`, `AmbienceRating`, or `QualityRating` is in the changed field set → behavior implementation checks each rating for the initial value (unset) → if all three are set: `OverallScore = (price + ambience + quality) / 3`; if any is unset: `OverallScore = null` → both the changed ratings and the new `OverallScore` are persisted in the same RAP SAVE sequence. No separate or asynchronous step.

**3. Public browses The Codex**

Browser loads the public Fiori Elements List Report app → app calls `GET .../Location` on `ZAGORA_PUBLIC_SRV` (anonymous binding, no authentication required) → RAP framework evaluates CDS access control (DCL object grants unrestricted read on the public service) → returns `Location` collection → user navigates to a detail page → `GET .../Location(guid)?$expand=_Visit` → `Visit` entities returned from `ZAGORA_C_VISIT_PUB` projection, which structurally excludes `AdminUserId`.

**4. Wishlist promotion**

Admin triggers the "Promote to Codex" action button on the Location Object Page → Fiori Elements sends OData POST to `.../Location(guid)/com.sap.zagora.promote` → RAP framework routes to the bound action method in `ZAGORA_BP_LOCATION` → action validates `Status = 'WISHLIST'` (raises exception if already `VISITED`) → sets `Status = 'VISITED'` and updates `LastChangedAt` → returns the updated `Location` entity → Fiori Elements refreshes the Object Page.

**5. Admin views The Archive**

Admin navigates to The Archive Fiori app → app calls `GET .../GetDashboardStats()` on `ZAGORA_ADMIN_SRV` → RAP function implementation executes five HANA aggregation queries (COUNT for totals, MAX visit count per location, MAX `OverallScore` excluding nulls) → returns a single structured result → Fiori Elements renders the analytics strip and two sorted location lists.

---

## 3. Security

### 3.1 Authentication — BTP Identity Authentication Service (IAS)

All admin users authenticate via SAP BTP IAS using SAML 2.0 or OIDC federation between IAS and the BTP ABAP system. The Fiori Elements app is registered as an application in IAS. Token lifetimes, session management, and refresh are governed by IAS configuration — no application code handles these concerns. Public read access requires no authentication.

### 3.2 Authorization — ABAP Authorization Objects and BTP Role Collections

One custom ABAP authorization object is defined:

| Object | Field | Allowed Activities |
|---|---|---|
| `ZAGORA_LOC` | `ACTVT` | `01` Read, `02` Create, `03` Change/Update, `06` Delete |

A PFCG role `ZAGORA_ADMIN_ROLE` grants `ZAGORA_LOC` with the full activity set (`01`, `02`, `03`, `06`). This role is mapped to the BTP role collection `Agora_Admin`. Admin users are assigned `Agora_Admin` in the BTP cockpit.

Authorization checks are performed in the RAP behavior implementation methods before any MODIFY or DELETE operation executes.

Public read access: The public service binding `ZAGORA_PUBLIC_SRV` uses a CDS access control (DCL object) that grants unrestricted read without any authorization object check. Write operations are structurally absent from the public service definition.

### 3.3 Transport Management

All development objects (HANA tables, ABAP dictionary types and domains, CDS views, behavior definitions, service definitions, service bindings, PFCG roles) belong to transportable ABAP package `ZAGORA`. Changes must pass the ABAP Test Cockpit (ATC) with no errors and no priority-1 warnings before the transport request is released for import to quality/production. Standard CTS (Change and Transport System) is used, integrated with the BTP ABAP environment's three-system transport chain (Dev → QA → Production).

### 3.4 ETag and Optimistic Locking

All root and child BO entities expose `LastChangedAt` as the `etag master` field (declared via the `@Semantics.systemDateTime.lastChangedAt` annotation and `etag master LastChangedAt` in the behavior definition). Clients must supply the current ETag value in the `If-Match` header on all PATCH, PUT, DELETE, and bound-action requests. The RAP framework enforces this automatically and returns HTTP `412 Precondition Failed` on an ETag mismatch — no application code is required for this check.

---

## 4. Implementation Notes

### 4.1 Overall Score Computation

`overall_score = (price_rating + ambience_rating + quality_rating) / 3.0`

- Computed in the `MODIFY` method of behavior implementation class `ZAGORA_BP_LOCATION`, triggered when `PriceRating`, `AmbienceRating`, or `QualityRating` is in the changed field set.
- **ABAP initial-value handling:** In ABAP, the initial value for `abap.int1` is `0`, which is indistinguishable from a zero rating without an explicit "is set" flag. The HANA column for each rating field must allow NULL. The behavior implementation sets the field to null when the user clears a rating, and tests for null/initial before computing the score. If any rating is null/initial, `OverallScore` is explicitly set to null (`abap.dec(3,2)` with NULL in HANA).
- Extract the computation into a private helper method in the behavior implementation class and cover it with ABAP Unit tests.
- The Archive stat `HighestRanked` excludes locations where `OverallScore IS NULL`.

### 4.2 AdminUserId Attribution

On Visit Create, the behavior implementation sets `AdminUserId` using:
```abap
cl_abap_context_info=>get_user_technical_name( )
```
or the equivalent call for resolving the BTP IAS principal in the ABAP system context. This field is set server-side and must **not** be included in the Create request body — if supplied, it is ignored.

`AdminUserId` is defined on the interface CDS view `ZAGORA_I_VISIT`. It is structurally absent from the public consumption view `ZAGORA_C_VISIT_PUB`. The admin consumption view `ZAGORA_C_VISIT` may expose it, optionally joined with `DisplayName` from `ZAGORA_ADMIN_USER` for attribution display.

### 4.3 CDS Naming Conventions

All custom development objects use the `ZAGORA_` namespace prefix. Sub-packages: `ZAGORA_DATA` (HANA tables, dictionary types, domains), `ZAGORA_SERVICES` (CDS views, behavior definitions, service objects).

| Object Type | Pattern | Example |
|---|---|---|
| HANA table (transparent) | `ZAGORA_<ENTITY>` | `ZAGORA_LOCATION` |
| Basic interface CDS view | `ZAGORA_I_<ENTITY>` | `ZAGORA_I_LOCATION` |
| Consumption CDS view (admin) | `ZAGORA_C_<ENTITY>` | `ZAGORA_C_LOCATION` |
| Consumption CDS view (public) | `ZAGORA_C_<ENTITY>_PUB` | `ZAGORA_C_LOCATION_PUB` |
| Behavior definition | `ZAGORA_I_<ENTITY>` | (same name as interface CDS view) |
| Behavior implementation class | `ZAGORA_BP_<ENTITY>` | `ZAGORA_BP_LOCATION` |
| Service definition (admin) | `ZAGORA_<ENTITY>_SD` | `ZAGORA_LOCATION_SD` |
| Service definition (public) | `ZAGORA_<ENTITY>_PUBLIC_SD` | `ZAGORA_LOCATION_PUBLIC_SD` |
| Service binding (admin) | `ZAGORA_ADMIN_SRV` | |
| Service binding (public) | `ZAGORA_PUBLIC_SRV` | |
| Authorization object | `ZAGORA_LOC` | |
| PFCG role | `ZAGORA_ADMIN_ROLE` | |
| ABAP domain | `ZAGORA_D_<NAME>` | `ZAGORA_D_LOC_TYPE` |
| RAP draft table (auto-generated) | `ZAGORA_<ENTITY>_D` | `ZAGORA_LOC_D`, `ZAGORA_VIS_D` |

### 4.4 Fiori Elements Floorplan Assignments

All UI behavior is driven by CDS annotations (`@UI.lineItem`, `@UI.fieldGroup`, `@UI.facet`, `@UI.selectionField`, `@UI.identification`, etc.) on the consumption CDS views. No custom SAPUI5 controller code is required for standard floorplans.

| Module | Fiori Elements Floorplan | Notes |
|---|---|---|
| The Codex — location list | List Report Object Page (LROP) | Filter bar: `Status`, `LocType`; table: `Name`, `City`, `LocType`, `Status`, `OverallScore` |
| The Codex — location detail | Object Page (within LROP) | Sections: Basic Info, Ratings, Visit Timeline (table facet for `_Visit`) |
| The Forum | Same LROP pre-filtered `Status = 'WISHLIST'`, or separate Fiori app with a fixed filter | Object Page header: "Promote to Codex" bound action button |
| The Archive | Overview Page (OVP) or Analytical List Page (ALP) | KPI card or header area for the analytics strip (R-031); table sections for the two sorted lists (R-032) |
| Admin User Management | Simple List Report | Create and Delete actions inline; no navigation to Object Page required |
| Public Codex | Read-only List Report | Served from `ZAGORA_PUBLIC_SRV`; no Create/Edit/Delete buttons; separate Fiori app deployment |

### 4.5 Draft Handling

RAP managed draft is enabled for the `Location` BO (root + compositions). This supports the standard Fiori Object Page UX where a user can start editing, navigate away, and resume their unsaved changes.

- Declared with `with draft` on the root entity in the behavior definition.
- Draft HANA tables (`ZAGORA_LOC_D`, `ZAGORA_VIS_D`) are auto-generated by the RAP framework — do not create these manually.
- Draft is available only on the admin service binding (`ZAGORA_ADMIN_SRV`). The public service binding does not expose draft operations.
- The `AdminUser` BO does not use draft (simple roster management does not benefit from it).

---

## 5. Infrastructure and Deployment

### Deployment Model

SAP BTP ABAP Environment is a fully managed PaaS. There is no infrastructure to provision, no containers to build, and no environment variables in the Docker Compose sense. Deployment is performed via:

- **ABAP Development Tools (ADT)** in Eclipse — primary development and transport workflow.
- **abapGit** — links the ABAP package `ZAGORA` to a Git repository. Each ABAP object is serialized as XML on push and deserialized on pull.

### Transport Chain

Standard ABAP three-system landscape:

```
Development system → Quality / Test system → Production system
```

Transport requests are created and released in ADT (Transport Organizer). Import into quality/production is triggered via the BTP cockpit or the CTS import queue.

### Configuration

There are no environment variables. Runtime configuration is managed via:

- **ABAP system parameters** (managed by SAP BTP).
- **IAS application configuration** (OIDC/SAML trust setup, user groups, attribute mappings) — configured in the IAS admin console.
- **BTP role collection assignments** — configured in the BTP cockpit; maps `Agora_Admin` role collection to IAS user groups or individual users.

---

## 6. Repository Structure

The monorepo file layout (api/, web/, nginx/) from v1.0 does not apply. All application logic lives in the BTP ABAP system, managed via abapGit.

```
<git-repository-root>/
├── REQUIREMENTS.md
├── DATA_MODEL.md
├── API_SPEC.md
├── ARCHITECTURE.md
└── src/                         abapGit-serialized ABAP objects
    ├── ZAGORA_DATA/             Sub-package: dictionary objects
    │   ├── ZAGORA_LOCATION.tabl.xml
    │   ├── ZAGORA_VISIT.tabl.xml
    │   ├── ZAGORA_ADMIN_USER.tabl.xml
    │   ├── ZAGORA_D_LOC_TYPE.doma.xml
    │   └── ZAGORA_D_STATUS.doma.xml
    └── ZAGORA_SERVICES/         Sub-package: CDS views, behavior, service objects
        ├── ZAGORA_I_LOCATION.ddls.xml
        ├── ZAGORA_I_VISIT.ddls.xml
        ├── ZAGORA_C_LOCATION.ddls.xml
        ├── ZAGORA_C_LOCATION_PUB.ddls.xml
        ├── ZAGORA_C_VISIT.ddls.xml
        ├── ZAGORA_C_VISIT_PUB.ddls.xml
        ├── ZAGORA_C_ADMIN_USER.ddls.xml
        ├── ZAGORA_C_DASHBOARD_STATS.ddls.xml
        ├── ZAGORA_I_LOCATION.bdef.xml    (behavior definition)
        ├── ZAGORA_BP_LOCATION.clas.xml   (behavior impl class)
        ├── ZAGORA_BP_ADMIN_USER.clas.xml
        ├── ZAGORA_LOCATION_SD.srvd.xml   (service definition — admin)
        ├── ZAGORA_LOCATION_PUBLIC_SD.srvd.xml
        ├── ZAGORA_ADMIN_SRV.srvb.xml     (service binding — admin)
        └── ZAGORA_PUBLIC_SRV.srvb.xml
```

---

## 7. Terminology Glossary

| v1.0 Term | v2.0 Equivalent |
|---|---|
| PostgreSQL table | HANA transparent table (ABAP dictionary table) |
| UUID PK / `gen_random_uuid()` | `sysuuid_x16`; RAP-managed key generation (`%key` auto-fill) |
| `PATCH /locations/:id` | OData PATCH on `Location` entity with `If-Match` ETag |
| JWT access token | BTP IAS session token (managed by IAS, not by application code) |
| `authMiddleware` (Express) | ABAP authorization object check in RAP behavior implementation |
| Express route handler | RAP behavior implementation method in `ZAGORA_BP_*` class |
| Docker service | BTP ABAP Environment managed service instance |
| `timestamptz` | `abap.utclong` |
| `smallint CHECK(1–5)` | `abap.int1` with validation rule in behavior definition |
| PostgreSQL `ENUM` type | ABAP domain with fixed values (`ZAGORA_D_*`) |
| npm migration script | ABAP dictionary table activation + CTS transport |
| `.env` file | BTP cockpit / IAS configuration |
| Nginx reverse proxy | SAP BTP ABAP Environment built-in HTTP routing |
