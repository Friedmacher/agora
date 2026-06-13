# Agora — Architecture

> **Version:** 1.0 | **Status:** AUTHORITATIVE

---

## 1. System Overview

### Component Diagram

```
Browser (React SPA)
        │ HTTPS (443)
        ▼
┌─────────────────────┐
│      Nginx          │  Serves static SPA build on /
│  (reverse proxy)    │  Proxies /api/* to Express API
└────────┬────────────┘
         │ HTTP (3000, internal)
         ▼
┌─────────────────────┐
│    Express API      │──── PostgreSQL (5432, internal)
│  (Node/TypeScript)  │──── MinIO       (9000, internal)
│                     │──── Nominatim   (HTTPS, external, rate-limited)
│                     │──── Meta Graph  (HTTPS, external)
└─────────────────────┘

PostgreSQL and MinIO are not publicly exposed (Docker internal network only).
```

### Service Boundaries

- The Express API is the **sole** service that communicates with PostgreSQL and MinIO. The React SPA never holds DB credentials or storage credentials.
- Social tokens exist only in PostgreSQL (encrypted at rest) and briefly in Express process memory during a share operation.
- The React SPA is a static build artifact served by Nginx. It has no runtime process.

---

## 2. Request Data Flows

**1. Admin creates a location**
`POST /api/locations` → Express validates JWT → writes location to DB (status=wishlist or visited) → returns 201 with location → background: queues Nominatim geocoding request → on geocoding success, `PATCH locations SET lat/lng` → WebSocket or polling update to UI (or admin triggers manually).

**2. Image upload**
`POST /api/locations/:id/images` (multipart) → Express validates JWT + file constraints → streams to MinIO at `{location_id}/{uuid}.{ext}` → inserts `images` record with `storage_key` → generates presigned URL (1h TTL) → returns `{id, presigned_url, display_order}`.

**3. Public map load**
`GET /api/dashboard/map` (unauthenticated) → DB query `SELECT id, name, type, status, lat, lng, overall_score, first image storage_key FROM locations WHERE lat IS NOT NULL` → for each location with an image, generate MinIO presigned URL (24h TTL) → return JSON array → React renders Leaflet map with 8 pin icon variants.

**4. Social share**
`POST /api/social/share` → Express validates JWT → fetches encrypted token from DB → decrypts in memory with `SOCIAL_TOKEN_ENCRYPTION_KEY` → if IG and token within 10 days of expiry: refresh token first, re-encrypt, store → generate MinIO presigned URL for selected image (10m TTL minimum) → call Meta Graph API → return `{post_id, url}` → plaintext token discarded from memory.

---

## 3. Security

### 3.1 JWT Strategy

- **Access token:** 15-minute expiry, HS256, payload `{sub: user_id, email, role: "admin"}`.
- **Refresh token:** 7-day expiry, HS256. On issuance: raw token is returned to client and its SHA-256 hash is stored in `refresh_tokens` with an expiry. On use: old token row is marked `revoked_at = now()` and a new pair is issued (rotation). Compromised tokens can be revoked server-side.
- All protected routes validate the access token via `authMiddleware` before the route handler executes.

### 3.2 Password Handling

- bcrypt with cost factor ≥ 12.
- No password reset UI in v1. Recovery: a CLI seed script (`npm run seed:reset-password`) available on the server for emergency access restoration.

### 3.3 Social Token Encryption

- Algorithm: **AES-256-GCM**.
- Key source: `SOCIAL_TOKEN_ENCRYPTION_KEY` env var (32-byte hex). Never stored in DB or source control.
- Storage format in DB (single text column): `{iv_hex}:{auth_tag_hex}:{ciphertext_hex}`.
- Decryption occurs in Express memory only at the moment of a share or token refresh call. Plaintext never leaves the process.

### 3.4 Transport and Headers

- Nginx terminates TLS (HTTPS on port 443). HTTP redirects to HTTPS.
- Express uses `helmet` middleware for standard security headers (CSP, HSTS, X-Frame-Options).
- CORS: Express configured to accept requests only from `CORS_ORIGIN` (nginx public URL in production; `http://localhost:5173` in development).

---

## 4. Implementation Notes

### 4.1 Nominatim Rate Limiting

The public `nominatim.openstreetmap.org` instance enforces a **1 request/second** hard limit per [usage policy](https://operations.osmfoundation.org/policies/nominatim/). Violations result in IP bans.

- Maintain an in-process async FIFO queue in `api/src/services/geocoding.ts`.
- Enforce a **1,100 ms minimum delay** between outgoing requests (10% buffer).
- Geocoding is triggered **after** a successful location save, not before. The save returns `201` immediately; lat/lng start as null.
- Each request must include a `User-Agent` header: `Agora/1.0 (self-hosted; contact: <admin-email>)`.
- If geocoding fails or times out (10 s timeout), lat/lng remain null. The location detail page and edit form show a visual indicator and a "Re-geocode" button that re-queues the request.
- The map endpoint (`GET /dashboard/map`) silently excludes locations with null coordinates.

### 4.2 Instagram Graph API Constraints

- Requires an **Instagram Business or Creator account** linked to a **Facebook Page**. Personal Instagram accounts cannot use the publishing API.
- Token flow: authorization code → short-lived user token (1h) → exchange for long-lived token (60 days) → store encrypted.
- **Token refresh strategy:** On each share call, check `expires_at`. If within 10 days, call the token refresh endpoint first, re-encrypt the new token, update the DB, then proceed with the share.
- The image sent to the Instagram API must be **publicly accessible via URL** at the time of the API call. Use MinIO presigned URLs with a **minimum 10-minute TTL** to ensure the URL is valid during Meta's asynchronous media processing.
- v1 supports **single-image posts** only. Carousel posts are out of scope.
- Surface a clear error in the admin UI if: (a) no Instagram account is connected, (b) the token is expired, or (c) the location has no images.

### 4.3 Overall Score Computation

`overall_score = (price_rating + ambience_rating + quality_rating) / 3.0`

- Computed in **application code** (not a PostgreSQL `GENERATED ALWAYS AS` column) because all three inputs are nullable and a generated column would require verbose `CASE` expressions to handle nullability correctly.
- Returns `null` if any dimension is `null` — including for wish-list entries (which typically have no ratings).
- Recalculated and stored on every `PATCH /locations/:id` that includes any of the three rating fields.
- The Archive "highest ranked" stat excludes locations where `overall_score IS NULL`.

### 4.4 MinIO Presigned URL Strategy

- `images.storage_key` stores the MinIO object path (e.g. `3f8a1c.../a9d2b4....jpg`). This is the only image identifier persisted in the DB.
- Presigned URLs are generated by the API layer **on every response** that includes image data. They are never stored.
- TTL policy:
  - Gallery view (Codex detail page): **1-hour TTL**
  - Map thumbnail (Atlas): **24-hour TTL** (reduces overhead on map load)
  - Social share image: **10-minute minimum TTL**
- When deleting a location or image: delete the MinIO object first, then the DB record. Failure to delete from MinIO is logged but does not block the DB deletion (orphan cleanup can be a maintenance task).

### 4.5 Visit Attribution in Public vs. Admin Contexts

- `visits.admin_user_id` records which admin logged each visit.
- **Public API responses** (`GET /locations/:id/visits`) omit `admin_user_id` — only `id`, `visited_at`, `note`, `created_at` are returned.
- **Admin API responses** may include `admin_user_id` and `display_name` (joined from `admin_users`) to show attribution in the admin UI.

---

## 5. Infrastructure (Docker Compose)

### Service Topology

| Service | Base Image | Exposed Port | Internal Port | Volumes | Depends On |
|---|---|---|---|---|---|
| `nginx` | `nginx:alpine` | 80, 443 | — | `./web/dist:/usr/share/nginx/html:ro`, SSL certs | `api` |
| `api` | Custom Dockerfile | — | 3000 | — | `db`, `minio` |
| `db` | `postgres:16-alpine` | — | 5432 | `postgres_data:/var/lib/postgresql/data` | — |
| `minio` | `minio/minio:latest` | 9001 (console, optional) | 9000 | `minio_data:/data` | — |

The React SPA is **not** a runtime service. It is built via `npm run build` and the `dist/` output is bind-mounted into the `nginx` container. The `api` container hosts only the Node/Express backend.

### Environment Variables

**`api` service**

| Variable | Description |
|---|---|
| `DATABASE_URL` | PostgreSQL connection string |
| `JWT_SECRET` | HS256 signing key for access tokens |
| `JWT_REFRESH_SECRET` | HS256 signing key for refresh tokens |
| `MINIO_ENDPOINT` | MinIO host (e.g. `minio:9000`) |
| `MINIO_ACCESS_KEY` | MinIO access key |
| `MINIO_SECRET_KEY` | MinIO secret key |
| `MINIO_BUCKET` | Bucket name (e.g. `agora-images`) |
| `MINIO_USE_SSL` | `false` for internal Docker network |
| `SOCIAL_TOKEN_ENCRYPTION_KEY` | 32-byte hex string (AES-256 key) |
| `NOMINATIM_BASE_URL` | Default: `https://nominatim.openstreetmap.org` |
| `META_APP_ID` | Meta (Facebook/Instagram) App ID |
| `META_APP_SECRET` | Meta App Secret |
| `META_OAUTH_REDIRECT_URI` | Publicly reachable callback URL |
| `CORS_ORIGIN` | Allowed origin (nginx public URL) |
| `NODE_ENV` | `production` or `development` |

**`db` service**

| Variable | Description |
|---|---|
| `POSTGRES_USER` | DB superuser |
| `POSTGRES_PASSWORD` | DB password |
| `POSTGRES_DB` | Database name |

**`minio` service**

| Variable | Description |
|---|---|
| `MINIO_ROOT_USER` | MinIO root user |
| `MINIO_ROOT_PASSWORD` | MinIO root password |

---

## 6. Repository Structure

Monorepo. All services in one repository, deployed via a single `docker-compose.yml`.

```
agora/
├── api/                        Node/Express backend (TypeScript)
│   ├── src/
│   │   ├── routes/             One file per resource group
│   │   │   ├── auth.ts
│   │   │   ├── locations.ts
│   │   │   ├── visits.ts
│   │   │   ├── images.ts
│   │   │   ├── dashboard.ts
│   │   │   ├── social.ts
│   │   │   └── adminUsers.ts
│   │   ├── middleware/
│   │   │   ├── authMiddleware.ts
│   │   │   └── errorHandler.ts
│   │   ├── services/
│   │   │   ├── geocoding.ts    Nominatim queue + rate limiter
│   │   │   ├── minio.ts        MinIO client, presigned URL helpers
│   │   │   ├── crypto.ts       AES-256-GCM encrypt/decrypt for social tokens
│   │   │   └── social/
│   │   │       ├── instagram.ts
│   │   │       └── facebook.ts
│   │   ├── db/
│   │   │   ├── pool.ts         pg Pool singleton
│   │   │   ├── migrations/     Numbered SQL migration files
│   │   │   └── seed.ts         Superuser seed script
│   │   └── index.ts            Express app entry point
│   ├── Dockerfile
│   └── package.json
├── web/                        React SPA (Vite + TypeScript)
│   ├── src/
│   │   ├── modules/
│   │   │   ├── codex/          Location log pages and components
│   │   │   ├── forum/          Wish-list pages and components
│   │   │   ├── archive/        Dashboard
│   │   │   ├── atlas/          Map view (Leaflet)
│   │   │   └── social/         Share UI + OAuth connection pages
│   │   ├── shared/
│   │   │   ├── components/     Reusable UI components
│   │   │   ├── hooks/          Custom React hooks
│   │   │   └── lib/            API client, types, utilities
│   │   └── App.tsx             Router + providers
│   └── vite.config.ts
├── nginx/
│   └── nginx.conf
├── docker-compose.yml
├── .env.example
└── README.md
```
