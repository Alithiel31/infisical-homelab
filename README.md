*[Version française](README.fr.md)*

# Infisical — self-hosted centralized secrets manager (homelab)

## Overview

Self-hosted [Infisical](https://infisical.com/) instance running on a Raspberry Pi 5 homelab, meant to become the centralized secrets manager for all projects (replacing scattered per-project `.env` files).

- **Status**: deployed and operational, admin account created.
- **Access**: through a private Tailscale tunnel, on a dedicated port — no public exposure.

## Architecture

Two containers, default Docker network (no custom network, no reverse proxy at this stage):

| Service | Image | Host port | Role |
|---|---|---|---|
| `infisical` | `infisical/infisical:latest` | only exposed inside the private tunnel | Main application |
| `infisical-redis` | `redis:7-alpine` | *(none, internal only)* | Internal cache / queue |

- **Database**: a **shared** PostgreSQL instance already running on the host (no dedicated Postgres container), with a dedicated database and application user.
  - Connects through the default Docker bridge gateway (no custom network, so no need for `extra_hosts` / a dedicated network).
  - ⚠️ To check: firewall rule allowing the default Docker bridge subnet to reach the Postgres port, otherwise the connection silently times out.
- **Redis**: no host port exposed, only reachable by `infisical` over the compose's internal network.
- **Volumes**: none declared — expected, since all persistent state (secrets, config) lives in the external Postgres instance; Redis is just a cache, losing it on restart is not an issue.

## Networking / routing

- Currently direct access via Tailscale only, no reverse proxy in front.
- The compose file has a commented-out block ready to enable routing through a reverse proxy (Traefik or similar) later, with a dedicated internal subdomain.

## Environment variables (`.env`, not committed — see `.gitignore` and `.env.example`)

| Variable | Description |
|---|---|
| `DB_PASSWORD` | Password for the Postgres application user |
| `DB_HOST` | Connection address to the shared Postgres instance |
| `ENCRYPTION_KEY` | Infisical encryption key (secrets at rest) — **critical, never lose or leak it** |
| `AUTH_SECRET` | Session/JWT signing secret |
| `INFISICAL_SITE_URL` | Public URL of Infisical (Tailscale address), e.g. `http://caesura.<tailnet>.ts.net:8090` |

See `.env.example` for the template to copy.

## Common operations

- **Start / update**: `docker compose up -d` (the `latest` tag on the image means `docker compose pull && docker compose up -d` can bump the version without warning — worth watching).
- **Logs**: `docker logs -f infisical`
- **Stop**: `docker compose down` (does not delete data, everything lives in the external Postgres instance).

## Known caveats / TODO

1. **Backup**: `ENCRYPTION_KEY` and `AUTH_SECRET` currently have no known backup outside the local `.env` file. Losing that file would make the secrets stored in Postgres unreadable. Should be stored in a second safe location.
2. **`latest` tag**: pin an exact version instead of `latest` to avoid a surprise upgrade in production.
3. **Reverse proxy**: decision still open, depending on how the mutualization effort for other projects progresses.
4. **Migrating existing secrets**: other projects still using local `.env` files haven't been migrated to Infisical yet.

## License

[MIT](LICENSE)
