# Agora — Requirements

> **Version:** 4.0 | **Status:** AUTHORITATIVE
> **Tagline:** *Find your space. Remember the taste.*

---

## 1. Project Overview

Agora is a personal hospitality location manager. An admin (one or more authenticated users) maintains a curated log of visited places (Periplus) and a wish-list of places to visit (Pothos). A map view (Pinax) displays all places geographically. A public read-only view of Periplus, Pothos, and Pinax is accessible to anyone without authentication. The admin dashboard (Tholos) is restricted to authenticated admin users.

### Access Modes

The application supports exactly two modes of access:

**Admin access** — authenticated and authorised users who maintain all application data. Admin access is protected by SAP BTP Identity Authentication Service (IAS) using SAML 2.0 / OIDC federation. All write operations (create, update, delete, promote) and the Tholos dashboard are exclusively available to admin users. Admin users interact via SAP Fiori Elements applications bound to `ZAGR_ADMIN_SRV`.

**Anonymous access** — unauthenticated, read-only access for any user. Anonymous users can browse Periplus (visited locations), Pothos (wish-list), and Pinax (map). No authentication is required and no write operations are exposed. Anonymous users interact via a SAP CAP Node.js web application that acts as a proxy to `ZAGR_PUBLIC_SRV` on the ABAP backend — the CAP app forwards OData calls to the ABAP service and serves the responses through a React-based SPA.

**Scope boundary:** Single-tenant, SAP BTP ABAP Environment (managed cloud). No self-hosted infrastructure. No public self-registration. No native apps. No multi-tenancy. Admin UI layer is SAP Fiori Elements — no custom React SPA for admin. Anonymous UI layer is SAP CAP Node.js application with React SPA acting as a proxy to the ABAP public OData service.

### Module Name Map

| Internal Name  | Display Name       | Description |
|---|---|---|
| **Periplus**   | Location Log       | Visited location detail pages — metadata, visit timeline, ratings |
| **Pothos**     | Wish-List          | Locations the admin plans to visit; promotable to Periplus entries |
| **Pinax**      | Map                | Geographic map of all places, colour-coded by type and status |
| **Tholos**     | Dashboard          | Analytics strip + sorted location lists (admin only) |

---

## 2. Functional Requirements

Requirement IDs are stable.

### 2.1 Periplus (Location Log)

- **R-001** Admin can create, read, update, and delete visited locations.
- **R-002** Each location stores: name, type (from extensible type list — see R-010), address (via BAS — see R-018), optional website URL, status, coordinates (latitude/longitude — see R-013 and R-014), three dimension ratings, and a computed overall score. Opening hours are managed as a structured child entity (see R-009). Address fields (street address, postal code, city, country) are stored in the SAP Business Address Service (BAS) and linked to the location record via a Business Partner reference (`BpNumber`). The `ZAGR_LOCATION` HANA table does not store address fields directly.
- **R-003** Location type is drawn from an extensible customizing table (see R-010). The default set is: `Restaurant` | `Coffeehouse` | `Bar` | `Bakery`.
- **R-004** Each location has a visit timeline: one or more Visit records, each with a calendar date (`abap.dats`, date-only — no time component) and an optional text note.
- **R-005** The date of first visit is derived from the earliest Visit record for that location (not a stored field).
- **R-006** Each location has three dimension ratings (1–5): **Price**, **Ambience**, **Quality**. Ratings are optional.
- **R-007** Overall score is computed automatically as `(price_rating + ambience_rating + quality_rating) / 3.0`, stored as `numeric(3,2)`. Returns null if any dimension is null or unset.
- **R-008** Overall score is recalculated and persisted to the database on every update operation that modifies any rating field. This is handled in the `MODIFY` phase of the RAP behavior implementation class `ZAGR_BP_LOCATION`.
- **R-009** Opening hours are modelled as a **child entity** of Location (`ZAGR_OPENING_HOURS`), with one record per day of the week. The data model per record is:

  | Field | Type | Description |
  |---|---|---|
  | `DayOfWeek` | `abap.int1` (1–7, Mon–Sun) | Key field — one record per day per location |
  | `OpenTime1` | `abap.tims`, optional | Start of first opening period |
  | `CloseTime1` | `abap.tims`, optional | End of first opening period |
  | `OpenTime2` | `abap.tims`, optional | Start of second opening period (e.g. dinner after a lunch break) |
  | `CloseTime2` | `abap.tims`, optional | End of second opening period |
  | `ClosedToday` | `abap.char(1)` boolean flag | Indicates the location is closed on this day |

  A day record with `ClosedToday = true` may omit all time fields; time fields are meaningless when `ClosedToday` is set. Both time pairs are optional independently — a location that is open continuously (no lunch break) only sets the first pair. Days with no record at all are treated as unknown/not configured.

  The UI presents opening hours as a structured inline table on the location detail view with five columns: **Day | Opens | Closes | Opens (2) | Closes (2)**. The fourth and fifth columns are only shown if at least one day in the set has a second opening period populated. A day with `ClosedToday = true` displays a "Closed" marker spanning the time columns instead of time values. Opening hours records are managed within the Location BO as a composition association and are part of the Location draft (admin service only).

- **R-018** **Business Partner lifecycle management:** Every `ZAGR_LOCATION` record is linked to a shadow Business Partner (BP) of type Organization in the SAP Business Address Service (BAS). The BP is created when a Location draft is **first activated** (not during draft save). The BP stores the location's address (street, postal code, city, country) as its default address (usage `XXDEFAULT`). On Location UPDATE, if any address field (`Address`, `ZipCode`, `City`, `Country`) changes, the behavior implementation updates the corresponding BAS address in the `AFTER SAVE` phase. On Location DELETE, the behavior implementation calls BAS to delete the shadow BP; if BAS deletion fails, the error is logged and the Location record is deleted regardless. The `ZAGR_LOCATION.BpNumber` field (type `bu_partner`) stores the 10-digit BP number and is set server-side — it must not be supplied in request bodies.

### 2.2 Pothos (Wish-List)

- **R-020** Admin can add wish-list entries with a minimum of name + address. "Address" here means at minimum a `City` and `Country` — `Street` (`Address` field) and `ZipCode` are optional. Address fields are provided by the admin on Location creation, stored in BAS via the shadow BP (see R-018), and exposed back in the OData entity as flat address properties.
- **R-021** All other location fields (website, ratings) are optional at creation time for wish-list entries.
- **R-022** Wish-list entries are displayed with a visually distinct treatment from visited entries across all views. Specifically: in Pinax, wish-list places use a **light/pastel** colour variant of the location-type colour; visited places use a **bold/saturated** colour variant.
- **R-023** A wish-list entry can be **promoted** to a Periplus entry. Promotion is implemented as a RAP bound action `promote` on the Location BO. The action transitions `Status` from `WISHLIST` to `VISITED`, preserves all existing fields, and returns the updated entity. No new root entity is created.
- **R-024** A location (whether visited or wish-list) can be marked as **CLOSED** to indicate the place no longer exists or is permanently shut. The `CLOSED` status is a third value in the status domain alongside `VISITED` and `WISHLIST`. Closed locations are visibly marked with a distinct treatment (e.g. strikethrough or badge) across all list and detail views. Closed locations are **excluded from Pinax** — they do not appear as map pins regardless of whether coordinates are set.

### 2.3 Pinax (Map)

- **R-025** Pinax displays all locations (both visited and wish-list) as pins on an interactive geographic map.
- **R-026** Each location type is assigned a distinct colour. Visited places render as a **bold/saturated** version of that colour; wish-list places render as a **light/pastel** version of the same colour. A map legend explains the colour scheme.
- **R-027** Selecting a pin opens a summary card for that location (name, type, status, address, overall score for visited places).
- **R-028** The Pinax view is available to both admin and anonymous users (read-only in both cases).
- **R-029** A location without resolved coordinates, or with status `CLOSED`, is excluded from the map view. Locations missing coordinates are surfaced in the admin UI with a warning indicator (see R-015).

### 2.4 Tholos (Dashboard)

- **R-030** Tholos is the landing page for authenticated admin users.
- **R-031** An **analytics strip** displays five stats, all computed server-side:
  1. Total visited locations
  2. Total wish-list entries
  3. Total visit events (sum of all Visit records)
  4. Top visited location (location with the most Visit records)
  5. Highest-ranked location (highest `OverallScore`; null scores excluded)
- **R-032** Below the strip: two sorted lists — visited locations by most recent `VisitedAt` (desc), wish-list by `CreatedAt` (desc).

### 2.5 Location Type Customizing

- **R-010** Location types are stored in a dedicated customizing table (`ZAGR_LOC_TYPE_T` / `ZAGR_LOC_TYPE_T_T` for the language-dependent text table). The table is transport-managed (client-dependent customizing).
- **R-011** The default seed values are: `Restaurant`, `Coffeehouse`, `Bar`, `Bakery`. These can be extended by an admin without code changes. Display labels must be translatable — the customizing table uses a language-dependent text table so that labels are stored per language and resolved at runtime based on the logon language.
- **R-012** Each location type record stores: a technical key (max 20 chars, upper case), a language-dependent display label, and a **base colour** selected via a colour picker in the admin UI (stored as a hex code). The two Pinax colour variants — bold/saturated for visited places and light/pastel for wish-list places — are **calculated automatically** from the base colour (e.g. by adjusting HSL saturation/lightness) and are not stored separately.

### 2.6 Geocoding

- **R-013** Every location record holds a latitude and longitude field (`ZAGR_LOCATION.Latitude` / `ZAGR_LOCATION.Longitude`, both `abap.dec(10,7)`). These are the canonical coordinates used by Pinax.
- **R-014** **Auto-geocoding (preferred path):** When a location is saved (create or update) and coordinates are not already set (or the address has changed), the system attempts to resolve coordinates automatically. The geocoding call uses address fields fetched from BAS — `StreetName`, `PostalCode`, `CityName`, `Country` — read via `I_BusinessPartnerAddress` using `BpNumber` as the join key. Geocoding failure is non-blocking: the record is saved without coordinates and a warning state is set (see R-013 / R-029).
- **R-015** **Manual override:** An admin can manually enter or correct latitude/longitude directly on the location form. A manually set coordinate is not overwritten by subsequent auto-geocoding runs unless the admin explicitly triggers a re-geocode action.
- **R-016** The geocoding backend is configurable: the default integration target is **Nominatim** (OpenStreetMap). The connection to the geocoding service is configured via a **SAP BTP Communication Arrangement** — a communication system and communication user pair are defined in the BTP ABAP environment, and the RAP behavior class resolves the endpoint and credentials from the communication arrangement at runtime. This allows the geocoding provider to be swapped without code changes.
- **R-017** A **re-geocode** bound action is available on the Location BO in the admin service, allowing an admin to trigger coordinate resolution on demand for any individual location.

### 2.7 User Management

- **R-040** Two roles: **Admin** (full CRUD) and **Public** (unauthenticated read-only). Role assignment is managed via BTP role collections mapped to IAS user groups.
- **R-041** Multiple admin accounts are supported. Admin users are IAS identities assigned to the `Agora_Admin` role collection.
- **R-042** Admin users are provisioned via the BTP cockpit or IAS admin console. No public self-registration. No seeded superuser script.

### 2.8 Anonymous Read-Only Access

- **R-050** Anonymous users can browse **Periplus** (visited location list and detail pages) via a SAP CAP Node.js web application that proxies OData requests to `ZAGR_PUBLIC_SRV`.
- **R-051** Anonymous users can browse **Pothos** (wish-list) via the same CAP web application. The wish-list view is read-only; no promote or edit actions are exposed to anonymous users.
- **R-052** Anonymous users can access **Pinax** (map) via the same CAP web application. The map is read-only; no create, edit, or promote actions are available to anonymous users.
- **R-053** Anonymous OData read access does not expose `AdminUserId` on Visit entities. The public CDS projection view `ZAGR_C_VISIT_PUB` omits that field.
- **R-054** Tholos and all write operations require the `Agora_Admin` role collection and are only accessible via the admin Fiori Elements interface.
- **R-055** The anonymous CAP application provides a React-based single-page application (SPA) with responsive design, optimised for public consumption without authentication requirements. The CAP application acts as a proxy: it forwards OData requests to `ZAGR_PUBLIC_SRV` on the ABAP backend and serves the responses to the React SPA. The CAP app does not define its own data layer or CDS service entities.

---

## 3. Non-Functional Requirements

### Performance

- **R-060** Initial Fiori Elements page load < 3 s. (BTP managed environment; no self-hosted infrastructure.)
- **R-061** Simple OData read responses (single entity, dashboard stats) < 500 ms.

### Scale

- **R-062** Design target: ~500 locations, ~2,000 visit records. No horizontal scaling requirement — managed BTP handles this.
- **R-063** No pagination required in v1. All location list OData requests return the full set (`$skip=0`, no enforced server-side page size).

### SAP Platform

- **R-064** Transport management — all RAP objects, CDS views, behavior definitions, and service bindings must be organised under the transportable ABAP package **`ZAGORA`**. Changes to production require a passing ATC (ABAP Test Cockpit) check queue.
- **R-065** All custom ABAP development objects follow the customer namespace prefix **`ZAGR_`**. No SAP standard namespace objects are modified.
- **R-066** Authorization — all CRUD operations and Tholos are protected by ABAP authorization object `ZAGR_LOC` checked in the RAP behavior implementation. Public read access is enforced via a CDS access control (DCL) granting open read on the public service binding.
- **R-067** ETag optimistic locking — all root and child BO entities must implement ETag-based optimistic locking via the RAP `etag master` annotation on `LastChangedAt` to prevent lost-update conflicts.
- **R-068** Two OData V4 service bindings are provided: `ZAGR_ADMIN_SRV` for the admin Fiori app (role-gated) and `ZAGR_PUBLIC_SRV` for the anonymous CAP application (anonymous read).
- **R-069** ATC checks — all custom ABAP objects must pass ATC with no errors and no priority-1 warnings before transport to production.
- **R-070** CAP Application — the anonymous frontend is a SAP CAP Node.js application with React SPA that **proxies OData requests** to `ZAGR_PUBLIC_SRV`. The CAP app serves the React build and forwards read-only OData requests from the browser to the ABAP backend — it does not define its own CDS entities or own data layer. The CAP app is deployed to BTP Cloud Foundry as the primary target, with local development support.

### Out of Scope (v1)

Native iOS/Android apps, image/file storage (no MinIO or DMS), social media integration (Instagram/Facebook), email notifications, public comments or ratings, multi-tenancy, import from Google Maps/Yelp, offline/PWA, password reset UI (managed by IAS).

---

## Appendix: Requirement ID Mapping (v2.0 → v4.0)

| v2.0 ID | v3.1 ID | v4.0 ID | Disposition |
|---|---|---|---|
| R-001 | R-001 | R-001 | Kept |
| R-002 | R-002 | R-002 | Updated — address fields moved to BAS (`BpNumber` replaces flat fields in `ZAGR_LOCATION`) |
| R-003 | R-003 | R-003 | Updated — type list now extensible via customizing |
| R-004 | R-004 | R-004 | Kept |
| R-005 | R-005 | R-005 | Kept |
| R-006 | R-006 | R-006 | Kept |
| R-007 | R-007 | R-007 | Kept |
| R-008 | R-008 | R-008 | Kept |
| — | R-009 | R-009 | New (v3.1) — structured opening hours as child entity |
| — | — | R-018 | **New (v4.0)** — BP lifecycle management (BAS shadow Business Partner) |
| R-020 | R-020 | R-020 | Updated — "address" clarified as BAS-backed fields; `City` + `Country` minimum |
| R-021 | R-021 | R-021 | Kept |
| R-022 | R-022 | R-022 | Updated — colour variant rule added |
| R-023 | R-023 | R-023 | Kept |
| — | R-024 | R-024 | New (v3.1) — CLOSED status; visual marking; Pinax exclusion |
| — | R-025–R-029 | R-025–R-029 | New (v3.1) — Pinax map module |
| R-030 | R-030 | R-030 | Kept; renamed to Tholos |
| R-031 | R-031 | R-031 | Kept |
| R-032 | R-032 | R-032 | Kept |
| — | R-010–R-012 | R-010–R-012 | New (v3.1) — location type customizing (extensible, translatable, colour picker) |
| — | R-013–R-017 | R-013–R-017 | R-014 updated (v4.0) — geocoding now reads address from BAS (`I_BusinessPartnerAddress`) |
| R-040 | R-040 | R-040 | Kept |
| R-041 | R-041 | R-041 | Kept |
| R-042 | R-042 | R-042 | Kept |
| R-043 | — | — | Removed — application relies on ABAP/BTP user management |
| R-044 | — | — | Removed — application relies on ABAP/BTP user management |
| R-050 | R-050 | R-050 | Kept; renamed Codex → Periplus |
| R-054 | R-051 | R-051 | Renamed; Forum → Pothos |
| — | R-052 | R-052 | New (v3.1) — anonymous Pinax access |
| R-051 | R-053 | R-053 | Kept |
| R-052 | R-054 | R-054 | Kept; renamed Archive → Tholos |
| R-053 | R-055 | R-055 | Kept |
| R-060 | R-060 | R-060 | Kept |
| R-061 | R-061 | R-061 | Kept |
| R-062 | R-062 | R-062 | Kept |
| R-063 | R-063 | R-063 | Kept |
| R-064 | R-064 | R-064 | Updated — explicit package `ZAGORA` |
| R-065 | R-065 | R-065 | Updated — namespace prefix changed to `ZAGR_` |
| R-066 | R-066 | R-066 | Kept; object renamed to `ZAGR_LOC` |
| R-067 | R-067 | R-067 | Kept |
| R-068 | R-068 | R-068 | Kept; service names updated to `ZAGR_` prefix |
| R-069 | R-069 | R-069 | Kept |
| R-070 | R-070 | R-070 | Kept; service name updated to `ZAGR_` prefix |
