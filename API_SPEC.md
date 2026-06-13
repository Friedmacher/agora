# Agora — API Specification

> **Version:** 1.0 | **Status:** AUTHORITATIVE

---

Base path: `/api`. All endpoints are served by the Express API behind Nginx.

**Auth notation:** `Public` = no authentication required. `JWT` = valid access token required in `Authorization: Bearer <token>` header.

---

## Auth

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/auth/login` | Public | `{email, password}` → `{access_token, refresh_token, user}` |
| POST | `/auth/refresh` | Public | `{refresh_token}` → `{access_token, refresh_token}` (old token revoked) |
| POST | `/auth/logout` | JWT | Revokes current refresh token |

## Locations

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/locations` | Public | All locations. Query: `?status=visited\|wishlist&type=restaurant\|...` |
| GET | `/locations/:id` | Public | Full location detail (includes images array with presigned URLs, visits array) |
| POST | `/locations` | JWT | Create location. Triggers async Nominatim geocoding if lat/lng absent. |
| PATCH | `/locations/:id` | JWT | Update fields. Recomputes `overall_score` if any rating field changes. |
| DELETE | `/locations/:id` | JWT | Hard delete. Cascades to visits, images (MinIO objects deleted first). |
| POST | `/locations/:id/promote` | JWT | Wishlist → visited transition. Returns updated location. |

## Visits

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/locations/:id/visits` | Public | All visits for location. `admin_user_id` excluded from response. |
| POST | `/locations/:id/visits` | JWT | Log a visit. `{visited_at, note?}`. Records `admin_user_id` from JWT. |
| DELETE | `/locations/:id/visits/:visitId` | JWT | Remove a visit record. |

## Images

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/locations/:id/images` | JWT | Multipart upload (max 10 MB, JPEG/PNG/WEBP). Streams to MinIO, returns image record with presigned URL. |
| PATCH | `/locations/:id/images/:imageId` | JWT | Update `display_order`. |
| DELETE | `/locations/:id/images/:imageId` | JWT | Deletes object from MinIO and removes DB record. |

## Dashboard

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/dashboard/stats` | JWT | Analytics strip: `{total_visited, total_wishlist, total_visits, top_visited: {id, name, visit_count}, highest_ranked: {id, name, overall_score}}` |
| GET | `/dashboard/map` | Public | Lightweight location payload for Atlas: `[{id, name, type, status, lat, lng, overall_score, thumbnail_key}]`. Excludes null-coordinate entries. |

## Social

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/social/status` | JWT | `{instagram: {connected: bool, expires_at?}, facebook: {connected: bool, expires_at?}}` |
| GET | `/social/oauth/:platform/start` | JWT | Redirects admin browser to Meta OAuth authorization URL. `platform`: `instagram` or `facebook`. |
| GET | `/social/oauth/:platform/callback` | Public* | OAuth callback. Exchanges code for long-lived token, encrypts, stores. Redirects to admin UI. *Must validate `state` param against session. |
| POST | `/social/share` | JWT | `{location_id, platform, image_id, caption?}` → `{post_id, url}`. Validates image exists for Instagram. |
| DELETE | `/social/tokens/:platform` | JWT | Disconnect social account; clears encrypted token. |

## Admin Users

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/admin/users` | JWT | List all admin users (no password hashes in response). |
| POST | `/admin/users` | JWT | Create new admin. `{email, password, display_name}`. |
| DELETE | `/admin/users/:id` | JWT | Delete admin. Returns 403 if `id` matches the requesting user. |

---

## Protected vs. Public Routes Summary

| Scope | Routes |
|---|---|
| **Public** | `GET /locations`, `GET /locations/:id`, `GET /locations/:id/visits`, `GET /dashboard/map`, `GET /social/oauth/:platform/callback` |
| **JWT required** | All other routes (POST, PATCH, DELETE + dashboard stats + social + admin users) |
