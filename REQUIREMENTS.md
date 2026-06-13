# Agora — Requirements

> **Version:** 1.0 | **Status:** AUTHORITATIVE
> **Tagline:** *Find your space. Remember the taste.*

---

## 1. Project Overview

Agora is a personal hospitality location manager. An admin (one or more authenticated users) maintains a curated log of visited places (The Codex) and a wish-list of places to visit (The Forum). A public read-only view is accessible to anyone without authentication. Social sharing to Instagram and Facebook is included.

**Scope boundary:** Single-tenant, self-hosted web application. No public self-registration. No native apps. No multi-tenancy.

### Module Name Map

| Internal Name | Display Name | Description |
|---|---|---|
| **The Codex** | Location Log | Visited location detail pages — metadata, visit timeline, image gallery, ratings |
| **The Forum** | Wish-List | Locations the admin plans to visit; promotable to Codex entries |
| **The Archive** | Dashboard | Analytics strip + sorted location lists |
| **The Atlas** | Map View | Interactive Leaflet map with distinct pin icons per type and status |
| — | Social Sharing | Direct posting to Instagram and Facebook from within the admin UI |

---

## 2. Functional Requirements

Requirement IDs are stable. `[x]` = frozen / decided.

### 2.1 The Codex (Location Log)

- `[x]` **R-001** Admin can create, read, update, and delete visited locations.
- `[x]` **R-002** Each location stores: name, type (enum), address, city, country, optional website URL, geo-coordinates, status, three dimension ratings, computed overall score, and a gallery of images.
- `[x]` **R-003** Location type is one of: `Restaurant` | `Coffeehouse` | `Bar` | `Bakery`.
- `[x]` **R-004** Geo-coordinates (lat/lng) are auto-derived from address via Nominatim on location save. If geocoding fails or is pending, lat/lng remain null and the UI shows a visual indicator with a manual "Re-geocode" button.
- `[x]` **R-005** Admin can manually override lat/lng coordinates on the edit form.
- `[x]` **R-006** Each location has a visit timeline: one or more Visit records, each with a date and an optional text note.
- `[x]` **R-007** The date of first visit is derived from the earliest Visit record for that location (not a stored field).
- `[x]` **R-008** Admin can upload multiple images per location. Accepted formats: JPEG, PNG, WEBP. Max size: 10 MB per image.
- `[x]` **R-009** Images have a `display_order` field; admin can reorder images in the gallery.
- `[x]` **R-010** Each location has three dimension ratings (1–5 stars): **Price**, **Ambience**, **Quality**. Ratings are optional.
- `[x]` **R-011** Overall score is computed automatically as `(price_rating + ambience_rating + quality_rating) / 3.0`, stored as `numeric(3,2)`. Returns null if any dimension is null.
- `[x]` **R-012** Overall score is recalculated and persisted to the DB on every PATCH that modifies any rating field.

### 2.2 The Forum (Wish-List)

- `[x]` **R-020** Admin can add wish-list entries with a minimum of name + address.
- `[x]` **R-021** All other location fields (website, ratings, images) are optional at creation time for wish-list entries.
- `[x]` **R-022** Wish-list entries are displayed with a visually distinct treatment from visited entries across all views.
- `[x]` **R-023** A wish-list entry can be **promoted** to a Codex entry. Promotion changes `status` from `wishlist` to `visited`, preserves all existing fields, and opens the full edit form in place. No new record is created.

### 2.3 The Archive (Dashboard)

- `[x]` **R-030** The Archive is the landing page for authenticated admin users.
- `[x]` **R-031** An **analytics strip** displays five stats, all computed server-side:
  1. Total visited locations
  2. Total wish-list entries
  3. Total visit events (sum of all Visit records)
  4. Top visited location (location with the most Visit records)
  5. Highest-ranked location (highest `overall_score`; null scores excluded)
- `[x]` **R-032** Below the strip: two sorted lists — visited locations by most recent `visited_at` (desc), wish-list by `created_at` (desc).

### 2.4 The Atlas (Map View)

- `[x]` **R-040** The Atlas displays all locations on an interactive map using Leaflet + OpenStreetMap tiles (no API key required).
- `[x]` **R-041** Locations without geo-coordinates (null lat/lng) are not rendered on the map.
- `[x]` **R-042** Map uses **8 distinct pin icon variants**: 4 types × 2 statuses (visited / wishlist).
- `[x]` **R-043** Clicking a pin opens a summary card showing: name, type, overall_score (if set), thumbnail (first image if available), and a link to the full Codex detail page.
- `[x]` **R-044** All locations are loaded in a single API call on map mount (no tile-based lazy loading; personal-scale data volume).
- `[x]` **R-045** The Atlas is accessible to unauthenticated public users.

### 2.5 Social Sharing

- `[x]` **R-050** Admin can connect their Instagram Business/Creator account and Facebook Page via a full OAuth 2.0 redirect flow (Meta Graph API).
- `[x]` **R-051** Tokens are long-lived (60-day) and refreshed automatically when within 10 days of expiry.
- `[x]` **R-052** Admin can disconnect a social account; this clears the stored encrypted token.
- `[x]` **R-053** A share can be triggered from two points: when a new location is added to The Codex, and when a new visit is logged for an existing location.
- `[x]` **R-054** Share content: auto-generated post text (location name, type, overall_score, optional custom caption override) + one image selected from the gallery.
- `[x]` **R-055** Instagram requires an image; if the location has no images, the share attempt is rejected with a clear error message. Facebook image is optional.
- `[x]` **R-056** Social sharing is only available to authenticated admin users.

### 2.6 User Management

- `[x]` **R-060** Two roles: **Admin** (full CRUD, social sharing, user management) and **Public** (unauthenticated read-only).
- `[x]` **R-061** Multiple admin accounts are supported. There is no single owner concept.
- `[x]` **R-062** Admins are created by an existing admin or via a seeded superuser during initial setup. No public self-registration.
- `[x]` **R-063** An admin cannot delete their own account.
- `[x]` **R-064** Session auth uses JWT (see `ARCHITECTURE.md` Section 3.1).
- `[x]` **R-065** Each admin has their own social OAuth tokens stored separately (per-admin social credentials).

### 2.7 Public Read-Only Access

- `[x]` **R-070** The public (unauthenticated) can browse The Codex (location list and detail pages) and The Atlas.
- `[x]` **R-071** Public views do not expose admin identity on visits, admin user data, or social tokens.
- `[x]` **R-072** The Archive dashboard and all write operations require authentication.

---

## 3. Non-Functional Requirements

### Performance

- `[x]` **R-080** Initial page load (SPA shell) < 2 s on a 10 Mbps connection.
- `[x]` **R-081** Map render with up to 500 pins < 1 s after API response.
- `[x]` **R-082** Simple read API responses (single location, dashboard stats) < 200 ms.

### Scale

- `[x]` **R-083** Design target: ~500 locations, ~2,000 visit records, ~1,000 images. No horizontal scaling requirement in v1.
- `[x]` **R-084** No pagination required. All location list endpoints return the full set.

### Security

- `[x]` **R-085** Passwords hashed with bcrypt, cost factor ≥ 12.
- `[x]` **R-086** Social tokens encrypted with AES-256-GCM before DB storage.
- `[x]` **R-087** JWT access tokens expire in 15 minutes; refresh tokens expire in 7 days.
- `[x]` **R-088** Refresh tokens are stored hashed in the DB and revoked on use (rotation).

### Out of Scope (v1)

Native iOS/Android apps, public comments or ratings, import from Google Maps/Yelp, offline/PWA, email notifications, X/Twitter and WhatsApp sharing, password reset UI flow (use seeded CLI script).
