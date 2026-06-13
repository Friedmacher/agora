# Agora — API Specification

> **Version:** 2.0 | **Status:** AUTHORITATIVE

---

## Overview

Agora's service layer is built with the **ABAP RESTful Application Programming Model (RAP)**. Services are OData V4, generated from CDS consumption views and service definitions — there are no hand-written REST endpoints. Clients interact with standard OData V4 conventions.

All service URLs are accessed via the SAP BTP ABAP Environment's standard OData gateway path pattern.

---

## Service Bindings

Two OData V4 service bindings are provided — one for each access tier.

| Binding | Service Definition | Auth | Base URL |
|---|---|---|---|
| `ZAGORA_ADMIN_SRV` | `ZAGORA_LOCATION_SD` | `Agora_Admin` BTP role collection | `/sap/opu/odata4/zagora/admin/srvd/zagora/zagora_location_sd/0001/` |
| `ZAGORA_PUBLIC_SRV` | `ZAGORA_LOCATION_PUBLIC_SD` | Anonymous / open read | `/sap/opu/odata4/zagora/public/srvd/zagora/zagora_location_public_sd/0001/` |

The public binding is restricted to read operations via CDS access control (DCL). Attempts to write via the public binding are rejected with HTTP `405 Method Not Allowed`.

---

## Entity Sets

| Entity Set | Source CDS View | Parent Entity Set | Available In |
|---|---|---|---|
| `Location` | `ZAGORA_C_LOCATION` (admin) / `ZAGORA_C_LOCATION_PUB` (public) | — | Both bindings |
| `Visit` | `ZAGORA_C_VISIT` (admin) / `ZAGORA_C_VISIT_PUB` (public) | `Location` (nav property `_Visit`) | Both bindings; `AdminUserId` excluded from public |
| `AdminUser` | `ZAGORA_C_ADMIN_USER` | — | Admin binding only |
| `DashboardStats` | `ZAGORA_C_DASHBOARD_STATS` | — | Admin binding only |

---

## Operations: Location

Base path (admin): `.../Location`
Base path (public): `.../Location`

| Operation | HTTP Method | URL | Auth | Notes |
|---|---|---|---|---|
| Read collection | GET | `.../Location` | Public | Supports `$filter`, `$orderby`, `$expand`, `$select`, `$count` |
| Read entity | GET | `.../Location(LocationUuid=guid'...')` | Public | Supports `$expand=_Visit` |
| Create | POST | `.../Location` | Admin | Body: all non-key, non-RAP-managed fields. ETag returned in response. |
| Update (partial) | PATCH | `.../Location(LocationUuid=guid'...')` | Admin | ETag required in `If-Match` header. Preferred over PUT for partial updates. |
| Update (full) | PUT | `.../Location(LocationUuid=guid'...')` | Admin | ETag required in `If-Match` header. |
| Delete | DELETE | `.../Location(LocationUuid=guid'...')` | Admin | Cascades to `Visit` child entities via RAP managed composition. ETag required. |
| Promote (action) | POST | `.../Location(LocationUuid=guid'...')/com.sap.zagora.promote` | Admin | Bound action; transitions `Status` from `WISHLIST` → `VISITED`. See action spec below. |

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

## Operations: AdminUser

| Operation | HTTP Method | URL | Auth | Notes |
|---|---|---|---|---|
| Read collection | GET | `.../AdminUser` | Admin | No password hashes or tokens in response |
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

## Custom Action: `promote`

| Property | Value |
|---|---|
| Type | Bound action on `Location` entity (single-entity binding) |
| HTTP | POST |
| URL pattern | `.../Location(LocationUuid=guid'...')/com.sap.zagora.promote` |
| Input parameters | None |
| Return type | `Location` entity (the updated record) |
| Side effects | Sets `Status = 'VISITED'`; does not modify ratings or Visit records |
| ETag | `If-Match` header required |
| Error — already visited | HTTP 422 with error detail if `Status` is already `VISITED` |

---

## OData Query Options

| Option | Supported On | Notes |
|---|---|---|
| `$filter` | `Location` | Fields: `Status`, `LocType` |
| `$orderby` | `Location` | Fields: `Name`, `CreatedAt`, `OverallScore`, `LastChangedAt` |
| `$expand` | `Location` | Associations: `_Visit` |
| `$select` | All entity sets | Standard OData field selection |
| `$count` | All collections | Returns total record count |
| `$skip` / `$top` | All collections | Supported but not required by the app (no pagination in v1) |
| `$search` | — | Not supported in v1 |

---

## Authorization Summary

| Operation | Admin Binding (`ZAGORA_ADMIN_SRV`) | Public Binding (`ZAGORA_PUBLIC_SRV`) |
|---|---|---|
| Read `Location` collection / entity | Yes | Yes |
| Write `Location` (create / update / delete) | Yes | No (405) |
| Read `Visit` collection / entity | Yes (with `AdminUserId`) | Yes (without `AdminUserId`) |
| Write `Visit` (create / update / delete) | Yes | No (405) |
| `promote` action | Yes | No (405) |
| Read `AdminUser` | Yes | No |
| Write `AdminUser` | Yes | No |
| Read `DashboardStats` | Yes | No |
