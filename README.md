# mes-docker

Docker Compose stack for the MES (Manufacturing Execution System) application.

Runs three containers:
- **redis** — SignalR backplane (Redis 8.6.2)
- **mes-backend** — .NET 10 minimal API
- **mes-frontend** — nginx serving the Vue SPA + reverse-proxying `/api/` and `/hubs/` to the backend

## Prerequisites

- Docker Engine with Compose v2 (`docker compose`)
- A running Keycloak instance (the `mes-realm` realm with the `mes-frontend` client configured)
- A running PostgreSQL instance with the MES schema already applied (`~/projects/dwh/init.sql`)

## Quick start

```bash
cd ~/projects/mes-docker
cp .env.example .env        # then edit .env with real values
docker compose up -d --build
```

The frontend is then reachable at `http://localhost:${HTTP_PORT}` (default 8090).

## Configuration (.env)

Copy `.env.example` to `.env` and fill in the values. **Never commit `.env`** — it contains credentials.

| Variable | Description |
|----------|-------------|
| `POSTGRES_CONNECTION_STRING` | Full Npgsql connection string. Use `host.docker.internal` to reach Postgres on the host. |
| `KEYCLOAK_AUTHORITY` | Public Keycloak realm URL. Must match the `iss` claim in every JWT (browser-facing). |
| `KEYCLOAK_BACKCHANNEL_AUTHORITY` | Internal Keycloak URL used by the backend to fetch JWKS. Use `host.docker.internal` when Keycloak runs as a Docker container on host port 8080. |
| `KEYCLOAK_CLIENT_ID` | Keycloak client ID (`mes-frontend`). |
| `HTTP_PORT` | Host port for the nginx container (default 80). |

### Authority vs BackchannelAuthority

`KEYCLOAK_AUTHORITY` is a **string** — the backend compares it against the `iss` claim in each JWT. No DNS resolution happens for this value. Set it to the exact issuer URL your Keycloak puts in tokens (e.g. `https://keycloak.test.local/realms/mes-realm`).

`KEYCLOAK_BACKCHANNEL_AUTHORITY` is a **network URL** — the backend fetches `{url}/.well-known/openid-configuration` and the JWKS endpoint from it. Set it to a URL the backend container can actually reach (e.g. `http://host.docker.internal:8080/realms/mes-realm` when Keycloak is exposed on the host's port 8080).

## Networking

The backend container has `extra_hosts: host.docker.internal:host-gateway`, which maps `host.docker.internal` to the host machine's IP. This lets the backend reach Postgres and Keycloak even when they are Docker containers exposed on the host.

The frontend container does **not** need `host.docker.internal` — it only talks to the backend container via the internal Docker network (`http://mes-backend:5000`).

## Redis / SignalR backplane

Redis is used as a SignalR backplane. Without it, SignalR broadcasts from one backend replica would not reach clients connected to a different replica (each replica only knows its own WebSocket connections).

With the Redis backplane:
1. Any backend replica calls `hub.Clients.All.SendAsync(...)`.
2. The SignalR Redis extension publishes the message to a Redis Pub/Sub channel.
3. Every other replica is subscribed to that channel and forwards the message to its own local WebSocket connections.

This makes horizontal scaling of `mes-backend` transparent to the frontend.

`Redis:ConnectionString` is set to `redis:6379` in `docker-compose.yml`. In the dev environment (bare `dotnet run`), `appsettings.json` leaves this empty, so SignalR falls back to in-memory mode.

## Updating after code changes

```bash
docker compose up -d --build
```

Compose rebuilds only the images whose source changed and restarts the affected containers. Redis data is preserved in the `redis_data` named volume.

## Logs

```bash
docker compose logs -f                    # all containers
docker compose logs -f mes-backend       # backend only
docker compose logs -f mes-frontend      # nginx access log
```

## Stopping

```bash
docker compose down          # stop and remove containers, keep volumes
docker compose down -v       # also remove the redis_data volume
```
