# Agora — API Specification

> **Version:** 3.1 | **Status:** AUTHORITATIVE

---

## Overview

Agora's service layer is built with the **ABAP RESTful Application Programming Model (RAP)**. Services are OData V4, generated from CDS consumption views and service definitions — there are no hand-written REST endpoints.

Two types of clients interact with these services:
- **Admin clients**: SAP Fiori Elements applications consuming `ZAGR_ADMIN_SRV` with authentication
- **Anonymous clients**: SAP CAP Node.js application consuming `ZAGR_PUBLIC_SRV` without authentication

All service URLs are accessed via the SAP BTP ABAP Environment's standard OData gateway path pattern.

---

## Service Bindings

Two OData V4 service bindings are provided — one for each access tier.

| Binding | Service Definition | Auth | Client Type | Base URL |
|---|---|---|---|---|
| `ZAGR_ADMIN_SRV` | `ZAGR_LOCATION_SD` | `Agora_Admin` BTP role collection | SAP Fiori Elements | `/sap/opu/odata4/zagr/admin/srvd/zagr/zagr_location_sd/0001/` |
| `ZAGR_PUBLIC_SRV` | `ZAGR_LOCATION_PUBLIC_SD` | Anonymous / open read | SAP CAP Node.js App | `/sap/opu/odata4/zagr/public/srvd/zagr/zagr_location_public_sd/0001/` |

The public binding is restricted to read operations via CDS access control (DCL). Attempts to write via the public binding are rejected with HTTP `405 Method Not Allowed`. The CAP application acts as a proxy and web interface for anonymous users.

---

## Entity Sets

| Entity Set | Source CDS View | Parent Entity Set | Available In |
|---|---|---|---|
| `Location` | `ZAGR_C_LOCATION` (admin) / `ZAGR_C_LOCATION_PUB` (public) | — | Both bindings |
| `Visit` | `ZAGR_C_VISIT` (admin) / `ZAGR_C_VISIT_PUB` (public) | `Location` (nav property `_Visit`) | Both bindings; `AdminUserId` excluded from public |
| `OpeningHours` | `ZAGR_C_OPENING_HOURS` (admin) / `ZAGR_C_OPENING_HOURS_PUB` (public) | `Location` (nav property `_OpeningHours`) | Both bindings |
| `LocType` | `ZAGR_C_LOC_TYPE` | — | Admin binding only — read + write for customizing |
| `AdminUser` | `ZAGR_C_ADMIN_USER` | — | Admin binding only |
| `DashboardStats` | `ZAGR_C_DASHBOARD_STATS` | — | Admin binding only — exposed as an **unbound OData function** (`GetDashboardStats()`), not as an entity set. No CRUD. |

---

## Operations: Location

Base path (admin): `.../Location`
Base path (public): `.../Location`

| Operation | HTTP Method | URL | Auth | Notes |
|---|---|---|---|---|
| Read collection | GET | `.../Location` | Public | Supports `$filter`, `$orderby`, `$expand`, `$select`, `$count` |
| Read entity | GET | `.../Location(LocationUuid=guid'...')` | Public | Supports `$expand=_Visit,_OpeningHours` |
| Create | POST | `.../Location` | Admin | Body: all non-key, non-RAP-managed fields. ETag returned in response. |
| Update (partial) | PATCH | `.../Location(LocationUuid=guid'...')` | Admin | ETag required in `If-Match` header. Preferred over PUT for partial updates. |
| Update (full) | PUT | `.../Location(LocationUuid=guid'...')` | Admin | ETag required in `If-Match` header. |
| Delete | DELETE | `.../Location(LocationUuid=guid'...')` | Admin | Cascades to `Visit` and `OpeningHours` child entities via RAP managed composition. ETag required. |
| Promote (action) | POST | `.../Location(LocationUuid=guid'...')/com.sap.zagr.promote` | Admin | Bound action; transitions `Status` from `WISHLIST` → `VISITED`. See action spec below. |
| Re-geocode (action) | POST | `.../Location(LocationUuid=guid'...')/com.sap.zagr.reGeocode` | Admin | Bound action; triggers coordinate resolution from address fields. See action spec below. |

---

## Operations: Visit

Accessed via navigation from `Location`. No direct root-level access.

| Operation | HTTP Method | URL | Auth | Notes |
|---|---|---|---|---|
| Read collection | GET | `.../Location(LocationUuid=guid'...')/_Visit` | Public | `AdminUserId` excluded from public binding response |
| Read entity | GET | `.../Location(LocationUuid=guid'...')/_Visit(VisitUuid=guid'...')` | Public | |
| Create | POST | `.../Location(LocationUuid=guid'...')/_Visit` | Admin | `AdminUserId` is set by the behavior implementation from the acting user's IAS principal — it must **not** be supplied in the request body |
| Update (partial) | PATCH | `.../Location(LocationUuid=guid'...')/_Visit(VisitUuid=guid'...')` | Admin | ETag required |
| Delete | DELETE | `.../Location(LocationUuid=guid'...')/_Visit(VisitUuid=guid'...')` | Admin | ETag required |

---

## Operations: OpeningHours

Accessed via navigation from `Location`. No direct root-level access. Key is `DayOfWeek` (1–7).

| Operation | HTTP Method | URL | Auth | Notes |
|---|---|---|---|---|
| Read collection | GET | `.../Location(LocationUuid=guid'...')/_OpeningHours` | Public | Returns up to 7 records (one per day) |
| Read entity | GET | `.../Location(LocationUuid=guid'...')/_OpeningHours(DayOfWeek=1)` | Public | |
| Create | POST | `.../Location(LocationUuid=guid'...')/_OpeningHours` | Admin | Body: `DayOfWeek`, `ClosedToday`, and any time fields. `DayOfWeek` must be unique per location — creating a duplicate day returns HTTP 422. |
| Update (partial) | PATCH | `.../Location(LocationUuid=guid'...')/_OpeningHours(DayOfWeek=1)` | Admin | ETag required |
| Delete | DELETE | `.../Location(LocationUuid=guid'...')/_OpeningHours(DayOfWeek=1)` | Admin | ETag required; deletes the day record (treated as "not configured" thereafter) |

---

## Operations: LocType (Customizing)

Admin-only. Supports the extensible location type list (R-010–R-012).

| Operation | HTTP Method | URL | Auth | Notes |
|---|---|---|---|---|
| Read collection | GET | `.../LocType` | Admin | Returns all active location types with language-dependent description (resolved per logon language) |
| Read entity | GET | `.../LocType(LocType='RESTAURANT')` | Admin | |
| Create | POST | `.../LocType` | Admin | Body: `LocType` (technical key), `BaseColor` (hex), `Description` (display label for logon language) |
| Update (partial) | PATCH | `.../LocType(LocType='RESTAURANT')` | Admin | Update `BaseColor` or `Description` |
| Delete | DELETE | `.../LocType(LocType='RESTAURANT')` | Admin | Returns HTTP 422 if any `Location` record references this type |

---

## Operations: AdminUser

| Operation | HTTP Method | URL | Auth | Notes |
|---|---|---|---|---|
| Read collection | GET | `.../AdminUser` | Admin | No sensitive fields in response |
| Read entity | GET | `.../AdminUser(UserUuid=guid'...')` | Admin | |
| Create | POST | `.../AdminUser` | Admin | Creates the application roster record. Separate BTP cockpit step required to assign the `Agora_Admin` role collection to the new user. |
| Delete | DELETE | `.../AdminUser(UserUuid=guid'...')` | Admin | Returns HTTP 400 if the acting user's IAS principal matches the target record's `IasPrincipal` field (self-delete prevention) |

---

## Operations: DashboardStats

Exposed as an OData unbound function.

| Operation | HTTP Method | URL | Auth | Response |
|---|---|---|---|---|
| Read stats | GET | `.../GetDashboardStats()` | Admin | `{TotalVisited, TotalWishlist, TotalVisits, TopVisited: {LocationUuid, Name, VisitCount}, HighestRanked: {LocationUuid, Name, OverallScore}}` |

`TopVisited` returns the location with the most Visit records. `HighestRanked` excludes locations where `OverallScore IS NULL`.

---

## Custom Actions

### `promote`

| Property | Value |
|---|---|
| Type | Bound action on `Location` entity (single-entity binding) |
| HTTP | POST |
| URL pattern | `.../Location(LocationUuid=guid'...')/com.sap.zagr.promote` |
| Input parameters | None |
| Return type | `Location` entity (the updated record) |
| Side effects | Sets `Status = 'VISITED'`; does not modify ratings or Visit records |
| ETag | `If-Match` header required |
| Error — not eligible | HTTP 422 if `Status` is already `VISITED` or `CLOSED` |

### `reGeocode`

| Property | Value |
|---|---|
| Type | Bound action on `Location` entity (single-entity binding) |
| HTTP | POST |
| URL pattern | `.../Location(LocationUuid=guid'...')/com.sap.zagr.reGeocode` |
| Input parameters | None |
| Return type | `Location` entity (the updated record, including resolved or cleared coordinates) |
| Side effects | Calls the geocoding service via Communication Arrangement; on success: writes `Latitude`, `Longitude`, sets `GeoResolved = X`; on failure: clears coordinates, `GeoResolved = space` |
| ETag | `If-Match` header required |
| Error — geocoding failure | HTTP 200 with updated entity returned; geocoding failure is non-blocking. The admin UI displays a warning if coordinates remain unresolved. |

---

## OData Query Options

| Option | Supported On | Notes |
|---|---|---|
| `$filter` | `Location` | Fields: `Status`, `LocType`, `City`, `Country` |
| `$orderby` | `Location` | Fields: `Name`, `CreatedAt`, `OverallScore`, `LastChangedAt` |
| `$expand` | `Location` | Associations: `_Visit`, `_OpeningHours` |
| `$select` | All entity sets | Standard OData field selection |
| `$count` | All collections | Returns total record count |
| `$skip` / `$top` | All collections | Supported but not required by the app (no pagination in v1) |
| `$search` | — | Not supported in v1 |

---

## Authorization Summary

| Operation | Admin Binding (`ZAGR_ADMIN_SRV`) | Public Binding (`ZAGR_PUBLIC_SRV`) | CAP Application Access |
|---|---|---|---|
| Read `Location` collection / entity | Yes | Yes | Yes (proxied via CAP) |
| Write `Location` (create / update / delete) | Yes | No (405) | No (405) |
| Read `Visit` collection / entity | Yes (with `AdminUserId`) | Yes (without `AdminUserId`) | Yes (without `AdminUserId`, proxied) |
| Write `Visit` | Yes | No (405) | No (405) |
| Read `OpeningHours` | Yes | Yes | Yes (proxied) |
| Write `OpeningHours` | Yes | No (405) | No (405) |
| `promote` action | Yes | No (405) | No (405) |
| `reGeocode` action | Yes | No (405) | No (405) |
| Read `LocType` | Yes | No | No |
| Write `LocType` | Yes | No | No |
| Read `AdminUser` | Yes | No | No |
| Write `AdminUser` | Yes | No | No |
| Read `DashboardStats` | Yes | No | No |

---

## CAP Application Integration

The SAP CAP Node.js application with React frontend (`cap-frontend/`) consumes `ZAGR_PUBLIC_SRV` to provide anonymous access. Key integration points:

### Service Proxy Configuration

```javascript
// CAP server-side proxy configuration (srv/odata-proxy.js)
const proxy = {
  target: 'https://<abap-system>.abap.eu10.hana.ondemand.com',
  basePath: '/sap/opu/odata4/zagr/public/srvd/zagr/zagr_location_public_sd/0001',
  auth: false // Anonymous access
};
```

### Filter Translation

The CAP proxy translates its internal URL conventions to standard OData `$filter` expressions. The React SPA does **not** construct OData query strings — it calls CAP-defined endpoints, and the CAP layer translates these to OData before forwarding.

| CAP endpoint | Forwarded OData query |
|---|---|
| `GET /api/locations` | _(no filter — returns all non-closed locations)_ `$filter=Status ne 'CLOSED'` |
| `GET /api/locations?status=VISITED` | `$filter=Status eq 'VISITED'` |
| `GET /api/locations?status=WISHLIST` | `$filter=Status eq 'WISHLIST'` |
| `GET /api/locations/:id` | _(single-entity read, no filter)_ |
| `GET /api/locations/:id/visits` | Navigation to `_Visit` |
| `GET /api/locations/:id/opening-hours` | Navigation to `_OpeningHours` |
| `GET /api/map-locations` | `$filter=Status ne 'CLOSED'&$select=LocationUuid,Name,LocType,Status,Latitude,Longitude,OverallScore` (Pinax optimised payload) |

### React Service Layer

```javascript
// React service calls (app/src/services/locationService.js)
const API_BASE = '/api'; // Proxied through CAP backend

export const locationService = {
  getAllLocations: ()     => fetch(`${API_BASE}/locations`),
  getLocation:    (id)   => fetch(`${API_BASE}/locations/${id}`),
  getPeriplus:    ()     => fetch(`${API_BASE}/locations?status=VISITED`),    // Periplus
  getPothos:      ()     => fetch(`${API_BASE}/locations?status=WISHLIST`),   // Pothos
  getMapLocations:()     => fetch(`${API_BASE}/map-locations`),               // Pinax
  getVisits:      (id)   => fetch(`${API_BASE}/locations/${id}/visits`),
  getOpeningHours:(id)   => fetch(`${API_BASE}/locations/${id}/opening-hours`)
};
```

### Typical Request Flows

1. **Periplus (visited):** React `GET /api/locations?status=VISITED` → CAP proxies to `GET .../Location?$filter=Status eq 'VISITED'&$orderby=LastChangedAt desc`
2. **Pothos (wish-list):** React `GET /api/locations?status=WISHLIST` → CAP proxies to `GET .../Location?$filter=Status eq 'WISHLIST'&$orderby=CreatedAt desc`
3. **Location detail:** React `GET /api/locations/:id` → CAP proxies to `GET .../Location(LocationUuid=guid'...')?$expand=_Visit,_OpeningHours`
4. **Pinax (map):** React `GET /api/map-locations` → CAP proxies to `GET .../Location?$filter=Status ne 'CLOSED'&$select=LocationUuid,Name,LocType,Status,Latitude,Longitude,OverallScore` — returns only locations with coordinates populated (further filtered client-side or via additional `$filter=Latitude ne null`)

### Data Transformation

The CAP application transforms OData responses for React consumption:
- Convert ABAP field names to camelCase for JavaScript conventions
- Resolve `LocType` technical key to display label (from the `LocType` entity set or a cached lookup)
- Calculate the two Pinax colour variants (bold / pastel) from the `BaseColor` hex value using HSL manipulation at render time
- Format dates (`abap.dats` ISO strings to display strings; `abap.tims` to HH:MM display)
- Handle null values gracefully (especially for ratings, overall scores, and coordinates)
- Map `DayOfWeek` integer (1–7) to locale-appropriate day name

### BTP Cloud Foundry Deployment

```yaml
# mta.yaml - Multi-Target Application descriptor
_schema-version: '3.1'
ID: agora-cap-frontend
version: 1.0.0
modules:
  - name: agora-cap-srv
    type: nodejs
    path: srv
    build-parameters:
      builder: npm-ci
    parameters:
      buildpack: nodejs_buildpack
      memory: 256MB
    provides:
      - name: srv-api
        properties:
          srv-url: ${default-url}
```
