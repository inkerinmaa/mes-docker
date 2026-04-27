# mes-docker

Docker Compose stack for the MES (Manufacturing Execution System) application.

Runs three containers:
- **redis** — SignalR backplane (Redis 8.6.2)
- **mes-backend** — .NET 10 minimal API
- **mes-frontend** — nginx serving the Vue SPA + reverse-proxying `/api/` and `/hubs/` to the backend

---

## Full deployment on a new server

### Prerequisites

- Docker Engine with Compose v2 (`docker compose`)
- Git
- An existing Keycloak instance (v21+) — see [Keycloak setup](#keycloak-setup) below
- A PostgreSQL instance for the MES database

### 1. Clone repositories

```bash
git clone https://github.com/your-org/mes-backend.git   ~/projects/mes-backend
git clone https://github.com/your-org/mes-frontend.git  ~/projects/mes-frontend
git clone https://github.com/your-org/mes-docker.git    ~/projects/mes-docker
git clone https://github.com/your-org/dwh.git           ~/projects/dwh
```

The repos must live at these exact relative paths — `docker-compose.yml` uses `../mes-backend` and `../mes-frontend` as build contexts.

### 2. Start PostgreSQL and apply schema

If you are using the bundled dev PostgreSQL from the `dwh` repo:

```bash
cd ~/projects/dwh
docker compose up -d
# Wait ~5 s for Postgres to initialise
docker exec -i postgres-db psql -U nik -d mydb < init.sql
```

If you have an existing PostgreSQL server, just apply the schema:

```bash
psql -h <host> -U <user> -d <database> < ~/projects/dwh/init.sql
```

### 3. Keycloak setup

Do this once in your Keycloak admin console.

#### 3a. Create realm

- Name: `mes-realm` (or any name; update `KEYCLOAK_AUTHORITY` to match)

#### 3b. Create client `mes-frontend`

| Setting | Value |
|---------|-------|
| Client type | OpenID Connect |
| Client authentication | **OFF** (public client) |
| Standard flow | ON |
| Direct access grants | OFF |
| Valid redirect URIs | `https://<mes-domain>/*` |
| Valid post-logout redirect URIs | `https://<mes-domain>/*` |
| Web origins | `https://<mes-domain>` |

#### 3c. Add protocol mappers to `mes-frontend`

Go to **Clients → mes-frontend → Client scopes → mes-frontend-dedicated → Add mapper → By configuration**.

**Audience mapper** (so `aud: mes-frontend` appears in tokens):
| Field | Value |
|-------|-------|
| Mapper type | Audience |
| Name | `mes-frontend-audience` |
| Included client audience | `mes-frontend` |
| Add to access token | ON |

**Group membership mapper** (so MES can derive role from Keycloak group):
| Field | Value |
|-------|-------|
| Mapper type | Group Membership |
| Name | `groups` |
| Token claim name | `groups` |
| Full group path | **OFF** |
| Add to ID token | ON |
| Add to access token | ON |
| Add to userinfo | ON |

#### 3d. Create groups

Create two groups in **Groups**:

| Group | MES role |
|-------|----------|
| `mes-admins` | `admin` — can create/cancel/transition orders, change settings |
| `mes-viewers` | `viewer` — read-only + can complete cages |

#### 3e. Create users

For each MES user:
1. **Users → Add user** — username, email, enable
2. **Credentials** tab → set password, disable "Temporary"
3. **Groups** tab → assign to `mes-admins` or `mes-viewers`

### 4. Configure .env

```bash
cd ~/projects/mes-docker
cp .env.example .env
```

Edit `.env`:

```env
# Full Npgsql connection string.
# Use host.docker.internal when Postgres is a Docker container on the same host.
POSTGRES_CONNECTION_STRING=Host=host.docker.internal;Port=5432;Database=mydb;Username=nik;Password=mysecretpassword

# Public Keycloak realm URL.
# Must exactly match the `iss` claim in every JWT (what the browser sees).
KEYCLOAK_AUTHORITY=https://keycloak.example.com/realms/mes-realm

# Internal URL the backend container uses to fetch JWKS.
# If Keycloak is a Docker container on the same host exposed on port 8080:
KEYCLOAK_BACKCHANNEL_AUTHORITY=http://host.docker.internal:8080/realms/mes-realm
# If Keycloak is on a separate host reachable by the backend container:
# KEYCLOAK_BACKCHANNEL_AUTHORITY=https://keycloak.example.com/realms/mes-realm

# Must match the Keycloak client ID
KEYCLOAK_CLIENT_ID=mes-frontend

# Host port for nginx (default 80)
HTTP_PORT=80
```

**Authority vs BackchannelAuthority explained:**

`KEYCLOAK_AUTHORITY` is a string comparison — the backend checks that every JWT's `iss` claim equals this value exactly. No network call; no DNS. Set it to the **public** URL your Keycloak puts in tokens.

`KEYCLOAK_BACKCHANNEL_AUTHORITY` is a real network URL — the backend fetches `{url}/.well-known/openid-configuration` and the JWKS from it at startup. Set it to a URL the **backend container** can actually reach. When Keycloak and the MES backend both run as Docker containers on the same host, use `http://host.docker.internal:8080/...`.

### 5. Start the stack

```bash
cd ~/projects/mes-docker
docker compose up -d --build
```

MES is now available at `http://<server>:${HTTP_PORT}`.

### 6. Verify

1. Open the MES URL in a browser — you should see the login page.
2. Click "Log in with Keycloak" — you should be redirected to Keycloak and back.
3. Check backend logs: `docker compose logs mes-backend | grep "User synced"` — you should see the role.

---

## Day-to-day operations

### Updating after code changes

```bash
cd ~/projects/mes-docker
docker compose up -d --build
```

Rebuilds only the images whose source changed. Redis data is preserved in the `redis_data` named volume.

### Viewing logs

```bash
docker compose logs -f                   # all containers
docker compose logs -f mes-backend      # backend only
docker compose logs -f mes-frontend     # nginx access log
```

### Stopping

```bash
docker compose down          # stop containers, keep volumes
docker compose down -v       # also remove the redis_data volume
```

---

## Configuration reference

| Variable | Description |
|----------|-------------|
| `POSTGRES_CONNECTION_STRING` | Npgsql connection string. Use `host.docker.internal` to reach Postgres on the host. |
| `KEYCLOAK_AUTHORITY` | Public Keycloak realm URL. Must match `iss` in JWTs (browser-facing). |
| `KEYCLOAK_BACKCHANNEL_AUTHORITY` | Internal Keycloak URL the backend fetches JWKS from. Use `host.docker.internal` when Keycloak is on the same host. |
| `KEYCLOAK_CLIENT_ID` | Keycloak client ID (`mes-frontend`). |
| `HTTP_PORT` | Host port for nginx (default 80). |

---

## Networking

The backend container has `extra_hosts: host.docker.internal:host-gateway`, mapping `host.docker.internal` to the host machine's IP. This lets the backend reach Postgres and Keycloak even when they run as separate Docker containers exposed on host ports.

The frontend container does **not** need `host.docker.internal` — it only talks to the backend over the internal Docker network (`http://mes-backend:5000`).

---

## Redis / SignalR backplane

Without Redis, SignalR broadcasts from one backend replica would not reach clients connected to a different replica. With the Redis backplane, any replica's broadcast is published to Redis Pub/Sub and forwarded to all other replicas' WebSocket connections.

`Redis:ConnectionString` is set to `redis:6379` in `docker-compose.yml`. In bare `dotnet run` dev mode, `appsettings.json` leaves it empty → in-memory fallback (single process, fine for dev).
