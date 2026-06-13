# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Agora is a personal hospitality location manager — a **software project** (React SPA + Node/Express API + PostgreSQL + MinIO), not an Obsidian vault task. The vault-level CLAUDE.md (one level up) does not apply here.

## Authoritative References

Read these before making any substantive changes. Together they are the single source of truth.

| File | Contents |
|---|---|
| [`REQUIREMENTS.md`](REQUIREMENTS.md) | Project overview, functional requirements (R-001–R-088), non-functional requirements, out-of-scope |
| [`DATA_MODEL.md`](DATA_MODEL.md) | Entity-relationship diagram, all table schemas |
| [`API_SPEC.md`](API_SPEC.md) | Full endpoint inventory with auth notation and payloads |
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | Component diagram, request flows, security, implementation notes, infra/env vars, repo structure |

## Commands

### Docker (primary workflow)

```bash
# Start all services (builds API image and starts db, minio, nginx)
docker compose up --build

# Start in detached mode
docker compose up --build -d

# View API logs
docker compose logs -f api

# Run DB migrations
docker compose exec api npm run migrate

# Seed the superuser
docker compose exec api npm run seed

# Reset an admin password (emergency recovery)
docker compose exec api npm run seed:reset-password
```

### API development (without Docker)

```bash
cd agora/api
npm install
npm run dev        # ts-node-dev / nodemon hot reload
npm run build      # tsc → dist/
npm run lint
npm test           # jest
npm test -- --testPathPattern=auth   # run a single test file
```

### Web development (without Docker)

```bash
cd agora/web
npm install
npm run dev        # Vite dev server on :5173 (proxies /api to localhost:3000)
npm run build      # outputs to dist/
npm run lint
npm run type-check
```

## Stack

React (Vite) + Node/Express (TypeScript) + PostgreSQL 16 + MinIO + Nginx — all in Docker Compose.

**Module names (use these consistently):**
- **The Codex** — visited location log
- **The Forum** — wish-list
- **The Archive** — dashboard (admin landing page)
- **The Atlas** — map view (Leaflet + OSM)

## Architecture

### Request path

All traffic enters Nginx on 443. Static SPA assets served directly; `/api/*` proxied to Express on port 3000 (internal Docker network). PostgreSQL and MinIO are not reachable outside the Docker network.

### Auth flow

- Access token: 15-min HS256 JWT, payload `{sub, email, role: "admin"}`, validated by `authMiddleware` on every protected route.
- Refresh token: 7-day HS256 JWT; raw token returned to client, **SHA-256 hash** stored in `refresh_tokens`. On use: old row is revoked and a new pair issued (rotation).

### Key cross-cutting concerns

| Concern | Where | Detail |
|---|---|---|
| `overall_score` | `api/src/routes/locations.ts` | Application-computed `(price + ambience + quality) / 3`; **not** a DB generated column. Must be recalculated and persisted on every rating PATCH. Returns null if any dimension is null. |
| Presigned URLs | `api/src/services/minio.ts` | `images.storage_key` is the only image identifier in the DB. Presigned URLs are generated per-response (1h gallery, 24h map, 10m social) and never stored. |
| Nominatim geocoding | `api/src/services/geocoding.ts` | FIFO in-process queue, **1,100 ms minimum delay** between requests. Geocoding is triggered after `201` is returned; lat/lng start null. `User-Agent` header required. |
| Social token encryption | `api/src/services/crypto.ts` | AES-256-GCM, key from `SOCIAL_TOKEN_ENCRYPTION_KEY`. DB format: `{iv_hex}:{auth_tag_hex}:{ciphertext_hex}`. Decrypted in memory only at share/refresh time. |
| Public vs. admin responses | `api/src/routes/visits.ts` | `admin_user_id` is stored but **omitted** from all public API responses. |

### Social sharing constraints

- Instagram requires a Business/Creator account linked to a Facebook Page (no personal accounts).
- The share image URL must be publicly accessible during Meta's async media processing — use a MinIO presigned URL with **at least 10-minute TTL**.
- On each share call: if token `expires_at` is within 10 days, refresh the token first, re-encrypt, update the DB, then share.

## Domain Entities

`locations`, `visits`, `admin_users`, `images`, `refresh_tokens` — all UUIDs via `gen_random_uuid()`.

Key constraints not obvious from column names:
- `locations.status`: `visited | wishlist` (default `wishlist`)
- `locations.type`: `restaurant | coffeehouse | bar | bakery`
- Rating columns (`price_rating`, `ambience_rating`, `quality_rating`): `smallint`, nullable, CHECK 1–5
- `images.display_order`: lower = shown first
- `visits.admin_user_id`: FK to `admin_users`, not exposed publicly

## Project Layout

```
agora/
├── api/
│   └── src/
│       ├── routes/         One file per resource group
│       ├── middleware/      authMiddleware, errorHandler
│       ├── services/        geocoding, minio, crypto, social/{instagram,facebook}
│       ├── db/              pool.ts (pg Pool singleton), migrations/, seed.ts
│       └── index.ts
├── web/
│   └── src/
│       ├── modules/         codex/, forum/, archive/, atlas/, social/
│       ├── shared/          components/, hooks/, lib/ (API client, types)
│       └── App.tsx
├── nginx/nginx.conf
├── docker-compose.yml
└── .env.example
```

## Implementation Status (as of 2026-06-13)

**Design phase — no code written yet.** The `agora/` directory structure does not exist yet.

### Starting a build session

1. Read all four reference docs in full before touching anything.
2. Scaffold in this order: `docker-compose.yml` → DB migrations → Express API skeleton → React SPA.
3. Follow the folder structure in `ARCHITECTURE.md` Section 6 exactly.
4. Update this section as modules are completed.

### Completed modules

_(none yet)_
