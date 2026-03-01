## Project Overview

**PloshtadkaBG** — a sports venue booking platform (Bulgaria). Monorepo of independent git repos managed via Docker Compose.

## Architecture

### Services

| Service | Language | Port | Path prefix |
|---|---|---|---|
| `ploshtadka-users-ms` | Python/FastAPI | 8000 | `/auth`, `/users`, `/scopes` |
| `ploshtadka-venues-ms` | Python/FastAPI | 8001 | `/venues` |
| `ploshtadka-bookings-ms` | Python/FastAPI | 8002 | `/bookings` |
| `ploshtadka-payments-ms` | Python/FastAPI | 8003 | `/payments` |
| `ploshtadka-notifications-ms` | Python/FastAPI | 8004 | `/notifications` |
| `ploshtadka-frontend` | React/TanStack | — | `/` |
| `ploshtadka-admin-panel` | React/React Router | — | `/admin` |
| Traefik | — | 80/8080 | API gateway + dashboard |
| PostgreSQL 16 | — | 5432 | shared DB in dev |
| pgAdmin | — | — | `/pgadmin` |

### Auth Flow (critical)

1. Client POSTs credentials to `/auth` → `users-ms` issues a JWT
2. **Traefik** `forwardAuth` middleware calls `http://users-ms:8000/auth/verify` for every protected request
3. On success Traefik injects three headers downstream:
   - `X-User-Id` (UUID)
   - `X-Username` (string)
   - `X-User-Scopes` (space-separated)
4. Downstream services read these headers in `app/deps.py::get_current_user()` — **they never validate the JWT themselves**. Do not add JWT validation inside venues-ms or bookings-ms; only users-ms does JWT validation.

Public routes (no auth): `GET /auth/*`, `POST /users`, `OPTIONS *`.

### Backend services — shared patterns

All four backend services follow the same layout (with the exception that `users-ms` does JWT validation itself and has no `conftest.py`/factories):

- **Framework**: FastAPI + Uvicorn (port declared in Dockerfile)
- **ORM**: Tortoise ORM with asyncpg; migrations via **Aerich** 
- **Package manager**: `uv` — always use `uv add` / `uv run`, never bare `pip`
- **Shared library**: `ms-core` (HexChap/MSCore on GitHub) provides:
  - `AbstractModel` — base Tortoise model
  - `CRUD[Model, Schema]` — base CRUD class
  - `setup_app(app, db_url, routers_path, models)` — wires Tortoise + auto-discovers all router files in `routers_path`; no manual router registration needed
- **DB**: SQLite in-memory for tests; PostgreSQL via `DB_URL` env var in prod

```
app/
  settings.py    # env vars
  models.py      # Tortoise models
  schemas.py     # Pydantic schemas (mirror enums from models)
  crud.py        # extends ms_core.CRUD
  deps.py        # get_current_user(), pre-built scope deps
  scopes.py      # StrEnum scopes (resource:action / admin:resource:action)
  routers/       # auto-discovered; one file per resource
tests/
  conftest.py    # fixtures: owner_client, admin_client, anon_app, client_factory
  factories.py   # test data builders
  test_*.py
```

#### users-ms key models
- `User`: `id` (UUID PK), `username`, `full_name`, `email`, `hashed_password`, `is_active`, `scopes` (JSON list)

#### venues-ms key models
- `Venue`: id, name, description, sport_types (JSON), status (enum), owner_id (UUID FK to users-ms), address, city, lat/lng, price_per_hour, currency, capacity, indoor/parking/amenities flags, working_hours (JSON dict by weekday), rating, images ↔ VenueImage, unavailabilities ↔ VenueUnavailability
- `VenueImage`: id, venue FK, url, is_thumbnail, order
- `VenueUnavailability`: id, venue FK, start_datetime, end_datetime, reason

#### bookings-ms key models
- `Booking`: id, venue_id (UUID), venue_owner_id (UUID, denormalized), user_id (UUID), start_datetime, end_datetime, status, price_per_hour, total_price, currency, notes, updated_at
- `BookingStatus`: `PENDING` → `CONFIRMED` / `CANCELLED`; `CONFIRMED` → `COMPLETED` / `CANCELLED` / `NO_SHOW`; terminal states are `COMPLETED`, `CANCELLED`, `NO_SHOW`

### Frontend services

| | ploshtadka-frontend | ploshtadka-admin-panel |
|---|---|---|
| Router | TanStack Router/Start | React Router DOM v7 |
| Data fetching | TanStack Query | TanStack Query |
| Forms | TanStack Form | React Hook Form + Zod |
| State | — | Zustand |
| Styling | Tailwind 4 + shadcn/ui | Tailwind 4 + shadcn/ui |
| i18n | intlayer | — |
| Charts | — | Recharts |
| DnD | — | dnd-kit |
| Runtime | Vite + bun | Vite + bun |
| Tests | vitest + @testing-library/react | — |

### Testing conventions (backend)

- **Mock the CRUD layer**, not the database (`unittest.mock.AsyncMock`)
- Use `owner_client` / `admin_client` fixtures for happy-path tests
- Use `anon_app` for 401/403 assertions
- Use `client_factory(make_user(scopes=[...]))` for custom scope combos
- Build data with `tests/factories.py`, not inline dicts
- Coverage minimum: **80%**

### Networking

All services communicate on the `backend` Docker network. No service is directly reachable from outside — only Traefik is exposed. Use service names (e.g. `http://users-ms:8000`) for inter-service calls.

### Docker Compose (`ploshtadka-compose/`)

`ploshtadka-compose/` contains the compose files that wire all services together. The service subdirectories inside it (`ploshtadka-admin-panel/`, `ploshtadka-users-ms/`, `ploshtadka-venues-ms/`) are the build contexts referenced by `build:` in compose — they mirror the repos in the monorepo root.

| File | Purpose |
|---|---|
| `docker-compose.yml` | Production: built static assets served by bun, all Traefik labels |
| `docker-compose.dev.yml` | Dev override: volume-mounts source, runs `bun install && bun run dev` (install is required — Docker anonymous volumes don't inherit image node_modules when a parent bind mount is present) |

```bash
# Production stack
docker compose -f docker-compose.yml up

# Dev stack (live-reload for frontend services)
docker compose -f docker-compose.yml -f docker-compose.dev.yml up
```

### Kubernetes (`ploshtadka-k8s/`)

Helm umbrella chart. Redis is a chart dependency; Traefik + CloudNativePG are installed separately via `scripts/prerequisites.sh` (they install CRDs that must exist before `helm install`).

| File | Purpose |
|---|---|
| `values.yaml` | Shared defaults (image repos, service ports, log levels) |
| `values.local.yaml` | Minikube: `imagePullPolicy: Never`, domain `localhost` |
| `values.staging.yaml` | Staging overrides |
| `values.prod.yaml` | Prod: HA postgres, HPA enabled, TLS |
| `sealed-secrets/` | Encrypted secrets — safe to commit |
| `scripts/seal.sh` | Interactive sealed-secret creation/rotation |

```bash
# Local (Minikube)
helm dependency update
helm install ploshtadka . -f values.yaml -f values.local.yaml

# Upgrade (prod — only changed services)
helm upgrade ploshtadka . -f values.yaml -f values.prod.yaml \
  --set "users-ms.image.tag=<sha>"

helm rollback ploshtadka   # instant rollback
```

**Key secrets:** `ploshtadka-db-app` (created by CloudNativePG — URI must use `asyncpg://` scheme, not `postgresql://`), `users-ms-secrets` (SECRET_KEY, GOOGLE_CLIENT_ID), `payments-ms-secrets` (STRIPE_SECRET_KEY, STRIPE_WEBHOOK_SECRET).

See `ploshtadka-k8s/README.md` for full prod setup (k3s, DNS, CI image push).

---

## Standard Workflow
Before making changes:
1. Investigate the affected service and its dependencies
2. Investigate thoroughly before touching code
3. Write tests FIRST (TDD) for backend logic
4. Implement in small diffs (<200 lines per step)
5. Run lint + tests after every step
6. Generate a PR description before finishing

## Shared Conventions
- Conventional commits: feat/fix/chore/docs/refactor/test
- ~80% test coverage

## Important Constraints
- Never modify the shared/ package without asking me first
- All inter-service communication uses the internal Docker network
- Secrets always go in .env files, never hardcoded
- Each service uses Aerich for migration

## Caching infrastructure
- Redis (`redis:7-alpine`) is on the `backend` Docker network, `REDIS_URL: redis://redis:6379/0`
- users-ms: caches `/auth/verify` in Redis keyed by SHA-256 of the JWT (`app/cache.py`), 5-min TTL; invalidated on user update/scope change
- bookings-ms: caches `/bookings/slots` keyed by `slots:{venue_id}` (`app/cache.py`), 60s TTL; invalidated on booking create/status update
- venues-ms: `Cache-Control: public` headers on `GET /venues/` (max-age=30) and `GET /venues/{id}` (max-age=60)

## Logging
- All backend services use loguru via `app/logging.py` + `setup_logging()` called in `main.py`
- `LOG_LEVEL` env var controls verbosity (default `INFO`; set `DEBUG` to see cache hit/miss logs)
- Compose: users-ms and bookings-ms default to `DEBUG`; venues-ms defaults to `INFO`
