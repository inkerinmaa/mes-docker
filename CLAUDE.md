# Project: mes-docker

## Purpose
Docker Compose orchestration for the MES stack: Redis backplane, .NET backend, Vue/nginx frontend.

## Directory layout
```
mes-docker/
  docker-compose.yml   — service definitions
  .env                 — real credentials (gitignored)
  .env.example         — template for .env
  README.md
  CLAUDE.md
```

## Commands
- Start stack:  `docker compose up -d --build` (from `~/projects/mes-docker/`)
- Stop stack:   `docker compose down`
- Logs:         `docker compose logs -f [service]`
- Rebuild one:  `docker compose up -d --build mes-backend`

## Services

| Service | Image | Port |
|---------|-------|------|
| redis | redis:8.6.2 | internal 6379 |
| mes-backend | built from `../mes-backend` | internal 5000 |
| mes-frontend | built from `../mes-frontend` | host `${HTTP_PORT:-80}` → 80 |

## Environment variables (all in .env)
- `POSTGRES_CONNECTION_STRING` — Npgsql connection string; `host.docker.internal` resolves to the host machine from inside the backend container
- `KEYCLOAK_AUTHORITY` — public Keycloak issuer URL (string-compared against JWT `iss`; no DNS needed)
- `KEYCLOAK_BACKCHANNEL_AUTHORITY` — internal Keycloak URL the backend fetches JWKS from; must be reachable from the container
- `KEYCLOAK_CLIENT_ID` — Keycloak client ID passed to both backend (audience check) and frontend (OIDC config)
- `KEYCLOAK_CLIENT_SECRET` — client secret for the `mes-frontend` confidential client; used by the frontend during authorization code exchange and stored in the backend config
- `HTTP_PORT` — host port for nginx (default 80)

## Keycloak requirements (what must exist before the stack starts)
- Realm with client `mes-frontend` (public, standard flow, correct redirect URIs)
- Two protocol mappers on `mes-frontend`: **Audience** (`mes-frontend`) and **Group Membership** (`groups`, full path OFF)
- Groups: `mes-admins` (→ admin role) and `mes-viewers` (→ viewer role)
- See `keycloak-setup/realm-export.json` for the canonical config and `mes-backend/README.md` for step-by-step instructions

## Networking
- `mes-backend` has `extra_hosts: host.docker.internal:host-gateway` so it can reach host-exposed services (Postgres on 5432, Keycloak on 8080).
- `mes-frontend` (nginx) proxies `/api/` and `/hubs/` to `http://mes-backend:5000` over the internal Docker network. The browser never calls the backend directly.

## Redis backplane
`Redis__ConnectionString=redis:6379` is injected into the backend container. The backend's SignalR configuration checks this at startup and enables the Redis backplane if non-empty. In dev (`dotnet run`), `appsettings.json` leaves it empty → in-memory fallback.

## Runtime frontend config
`docker-entrypoint.sh` in the frontend image runs before nginx starts and writes `/usr/share/nginx/html/config.json` from `$KEYCLOAK_AUTHORITY` and `$KEYCLOAK_CLIENT_ID`. The SPA fetches this file at startup via `loadConfig()` before initialising OIDC. No rebuild is needed to change Keycloak coordinates.

## Data persistence
- `redis_data` named volume stores Redis AOF/RDB. Survives `docker compose down`; destroyed by `docker compose down -v`.
- Postgres data lives outside this compose stack (managed separately).

## Source references
- Backend source: `~/projects/mes-backend/` — see its CLAUDE.md
- Frontend source: `~/projects/mes-frontend/` — see its CLAUDE.md
