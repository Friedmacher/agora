# Agora — Requirements

> **Version:** 2.0 | **Status:** AUTHORITATIVE
> **Tagline:** *Find your space. Remember the taste.*

---

## 1. Project Overview

Agora is a personal hospitality location manager. An admin (one or more authenticated users) maintains a curated log of visited places (The Codex) and a wish-list of places to visit (The Forum). A public read-only view is accessible to anyone without authentication.

**Scope boundary:** Single-tenant, SAP BTP ABAP Environment (managed cloud). No self-hosted infrastructure. No public self-registration. No native apps. No multi-tenancy. UI layer is SAP Fiori Elements — no custom React SPA.

### Module Name Map

| Internal Name | Display Name | Description |
|---|---|---|
| **The Codex** | Location Log | Visited location detail pages — metadata, visit timeline, ratings |
| **The Forum** | Wish-List | Locations the admin plans to visit; promotable to Codex entries |
| **The Archive** | Dashboard | Analytics strip + sorted location lists |
| — | Admin User Management | Manage recognized admin principals |

---

## 2. Functional Requirements

Requirement IDs are stable. `[x]` = frozen / decided.

### 2.1 The Codex (Location Log)

- `[x]` **R-001** Admin can create, read, update, and delete visited locations.
- `[x]` **R-002** Each location stores: name, type (enum), address, city, country, optional website URL, status, three dimension ratings, and a computed overall score.
- `[x]` **R-003** Location type is one of: `Restaurant` | `Coffeehouse` | `Bar` | `Bakery`.
- `[x]` **R-004** Each location has a visit timeline: one or more Visit records, each with a date and an optional text note.
- `[x]` **R-005** The date of first visit is derived from the earliest Visit record for that location (not a stored field).
- `[x]` **R-006** Each location has three dimension ratings (1–5): **Price**, **Ambience**, **Quality**. Ratings are optional.
- `[x]` **R-007** Overall score is computed automatically as `(price_rating + ambience_rating + quality_rating) / 3.0`, stored as `numeric(3,2)`. Returns null if any dimension is null or unset.
- `[x]` **R-008** Overall score is recalculated and persisted to the database on every update operation that modifies any rating field. This is handled in the `MODIFY` phase of the RAP behavior implementation class `ZAGORA_BP_LOCATION`.

### 2.2 The Forum (Wish-List)

- `[x]` **R-020** Admin can add wish-list entries with a minimum of name + address.
- `[x]` **R-021** All other location fields (website, ratings) are optional at creation time for wish-list entries.
- `[x]` **R-022** Wish-list entries are displayed with a visually distinct treatment from visited entries across all views.
- `[x]` **R-023** A wish-list entry can be **promoted** to a Codex entry. Promotion is implemented as a RAP bound action `promote` on the Location BO. The action transitions `Status` from `WISHLIST` to `VISITED`, preserves all existing fields, and returns the updated entity. No new root entity is created.

### 2.3 The Archive (Dashboard)

- `[x]` **R-030** The Archive is the landing page for authenticated admin users.
- `[x]` **R-031** An **analytics strip** displays five stats, all computed server-side:
  1. Total visited locations
  2. Total wish-list entries
  3. Total visit events (sum of all Visit records)
  4. Top visited location (location with the most Visit records)
  5. Highest-ranked location (highest `OverallScore`; null scores excluded)
- `[x]` **R-032** Below the strip: two sorted lists — visited locations by most recent `VisitedAt` (desc), wish-list by `CreatedAt` (desc).

### 2.4 User Management

- `[x]` **R-040** Two roles: **Admin** (full CRUD, user management) and **Public** (unauthenticated read-only). Role assignment is managed via BTP role collections mapped to IAS user groups.
- `[x]` **R-041** Multiple admin accounts are supported. Admin users are IAS identities assigned to the `Agora_Admin` role collection.
- `[x]` **R-042** Admin users are provisioned by an existing admin via the BTP cockpit or IAS admin console. No public self-registration. No seeded superuser script.
- `[x]` **R-043** An admin cannot delete their own account. The delete action must return an error if the acting user's IAS principal matches the target user's `IasPrincipal` field.
- `[x]` **R-044** Each admin account stores a display name and an IAS principal name (email/username) as the external identity.

### 2.5 Public Read-Only Access

- `[x]` **R-050** The public (unauthenticated) can browse The Codex (location list and detail pages).
- `[x]` **R-051** Public OData read access does not expose `AdminUserId` on Visit entities. The public CDS projection view `ZAGORA_C_VISIT_PUB` omits that field.
- `[x]` **R-052** The Archive dashboard and all write operations require the `Agora_Admin` role collection.

---

## 3. Non-Functional Requirements

### Performance

- `[x]` **R-060** Initial Fiori Elements page load < 3 s. (BTP managed environment; no self-hosted infrastructure.)
- `[x]` **R-061** Simple OData read responses (single entity, dashboard stats) < 500 ms.

### Scale

- `[x]` **R-062** Design target: ~500 locations, ~2,000 visit records. No horizontal scaling requirement — managed BTP handles this.
- `[x]` **R-063** No pagination required in v1. All location list OData requests return the full set (`$skip=0`, no enforced server-side page size).

### SAP Platform

- `[x]` **R-064** Transport management — all RAP objects, CDS views, behavior definitions, and service bindings must be organized under a transportable ABAP package. Changes to production require a passing ATC (ABAP Test Cockpit) check queue.
- `[x]` **R-065** All custom ABAP development objects follow the customer namespace prefix `ZAGORA_`. No SAP standard namespace objects are modified.
- `[x]` **R-066** Authorization — all CRUD operations and The Archive are protected by ABAP authorization object `ZAGORA_LOC` checked in the RAP behavior implementation. Public read access is enforced via a CDS access control (DCL) granting open read on the public service binding.
- `[x]` **R-067** ETag optimistic locking — all root and child BO entities must implement ETag-based optimistic locking via the RAP `etag master` annotation on `LastChangedAt` to prevent lost-update conflicts.
- `[x]` **R-068** Two OData V4 service bindings are provided: `ZAGORA_ADMIN_SRV` for the admin Fiori app (role-gated) and `ZAGORA_PUBLIC_SRV` for the public read-only Fiori app (anonymous read).
- `[x]` **R-069** ATC checks — all custom ABAP objects must pass ATC with no errors and no priority-1 warnings before transport to production.

### Out of Scope (v1)

Native iOS/Android apps, image/file storage (no MinIO or DMS), social media integration (Instagram/Facebook), interactive map, geocoding, email notifications, public comments or ratings, multi-tenancy, import from Google Maps/Yelp, offline/PWA, password reset UI (managed by IAS).

---

## Appendix: Requirement ID Mapping (v1.0 → v2.0)

| v1.0 ID | v2.0 ID | Disposition |
|---|---|---|
| R-001 | R-001 | Kept |
| R-002 | R-002 | Rewritten — removed geo-coordinates and image gallery |
| R-003 | R-003 | Kept |
| R-004 | — | Dropped — Nominatim geocoding removed |
| R-005 | — | Dropped — manual lat/lng override removed |
| R-006 | R-004 | Renamed |
| R-007 | R-005 | Renamed |
| R-008 | — | Dropped — image upload removed |
| R-009 | — | Dropped — image reorder removed |
| R-010 | R-006 | Renamed |
| R-011 | R-007 | Renamed; trigger restated for RAP |
| R-012 | R-008 | Renamed; trigger restated for RAP |
| R-020 | R-020 | Kept |
| R-021 | R-021 | Kept; image reference removed |
| R-022 | R-022 | Kept |
| R-023 | R-023 | Rewritten — promotion as RAP bound action |
| R-030 | R-030 | Kept |
| R-031 | R-031 | Kept |
| R-032 | R-032 | Kept |
| R-040 | — | Dropped — Atlas removed |
| R-041 | — | Dropped |
| R-042 | — | Dropped |
| R-043 | — | Dropped |
| R-044 | — | Dropped |
| R-045 | — | Dropped |
| R-050 | — | Dropped — social sharing removed |
| R-051 | — | Dropped |
| R-052 | — | Dropped |
| R-053 | — | Dropped |
| R-054 | — | Dropped |
| R-055 | — | Dropped |
| R-056 | — | Dropped |
| R-060 | R-040 | Renamed; rewritten for BTP IAM |
| R-061 | R-041 | Renamed |
| R-062 | R-042 | Renamed; seeded superuser removed |
| R-063 | R-043 | Renamed |
| R-064 | — | Dropped — JWT replaced by BTP IAS |
| R-065 | — | Dropped — social tokens removed |
| R-070 | R-050 | Renamed; Atlas reference removed |
| R-071 | R-051 | Renamed; restated for CDS projection |
| R-072 | R-052 | Renamed |
| R-080 | R-060 | Renamed; SPA shell reference removed |
| R-081 | — | Dropped — map render removed |
| R-082 | R-061 | Renamed; threshold adjusted for BTP |
| R-083 | R-062 | Renamed |
| R-084 | R-063 | Renamed |
| R-085 | — | Dropped — bcrypt removed |
| R-086 | — | Dropped — AES-256-GCM social tokens removed |
| R-087 | — | Dropped — JWT expiry removed |
| R-088 | — | Dropped — refresh token rotation removed |
| — | R-064 | New — transport management |
| — | R-065 | New — namespace prefix |
| — | R-066 | New — ABAP authorization objects |
| — | R-067 | New — ETag optimistic locking |
| — | R-068 | New — two service bindings |
| — | R-069 | New — ATC checks |
