# Honcho for Coolify

A self-contained, public Docker Compose deployment for [Honcho](https://github.com/plastic-labs/honcho), designed to be installed from the Coolify graphical interface.

This repository is intentionally independent from `vypdev/ai01`. It contains everything required to deploy a fresh Honcho instance on a Coolify Docker destination:

- Honcho API
- Honcho Deriver worker
- PostgreSQL 15 with `pgvector`
- Redis with AOF persistence
- Idempotent vector-extension initialization
- Health checks and dependency ordering
- A pinned upstream Honcho source (`v3.0.12`)

No application secrets, database data, Hermes state, or historical memory are stored here.

## Deployment model

```text
Hermes / OpenClaw on the host
        |
        | localhost HTTP, initially port 18081
        v
Honcho API (Coolify-managed Compose application)
        |
        +-- PostgreSQL + pgvector (private Compose network)
        +-- Redis (private Compose network)
        +-- Deriver worker (private Compose network)
```

The Compose file binds the API to loopback by default. It does not publish PostgreSQL or Redis. Keep the API private while `AUTH_USE_AUTH=false`.

## Coolify installation

### Build pack selection

When Coolify asks for the build pack, select:

```text
Docker Compose
```

Do not select Nixpacks, Railpack, Static, or Dockerfile. This deployment contains multiple coordinated services, so the Compose build pack is required.

The repository already contains the complete `docker-compose.yml` manifest. No custom Dockerfile path, Nixpacks configuration, Railpack configuration, or static-site configuration is needed.

The existing Coolify control plane uses host port `8000`. Do not assign Honcho to that port. During parallel validation, Honcho uses:

```text
Host address: 127.0.0.1
Host port:    18081
Container port: 8000
```

The container's port is `8000`; the host port is controlled by `HONCHO_API_PORT`.

### 1. Create the application

In Coolify:

1. Create or select a project.
2. Create a new application and select the **Docker Compose** build pack.
3. Select the target Docker server and destination.
4. Select the public repository:
   `https://github.com/vypdev/honcho-coolify`
5. Use the default branch: `master`.
6. Use repository root as the base directory: `/`.
7. Use `/docker-compose.yml` as the Compose file location (with base directory `/`). If the UI removes the leading slash automatically, `docker-compose.yml` is equivalent.
8. Leave the domain/public proxy configuration empty during initial validation.
9. Do not expose or publish PostgreSQL or Redis.

The repository is deliberately root-installable: no path inside `ai01` is required.

If Coolify displays a separate exposed-port field, use the application/container port `8000`; the Compose file handles the loopback host mapping to `18081`. Do not use Coolify's own host port `8000` for Honcho.

### 2. Configure variables

Add these variables in Coolify's environment-variable UI. Coolify should generate the PostgreSQL password automatically through its magic variable mechanism:

```text
SERVICE_PASSWORD_64_HONCHO_DB=<generated automatically by Coolify>
LLM_OPENAI_API_KEY=
OPENAI_API_KEY=
```

The Compose file maps `SERVICE_PASSWORD_64_HONCHO_DB` to the container variable `HONCHO_DB_PASSWORD`. Do not replace the generated database password with an API key or commit its value. Enter the DeepSeek and OpenAI API keys manually after Coolify has created the variables, and keep their values hidden.

Optional validation settings:

```text
HONCHO_API_PORT=18081
AUTH_USE_AUTH=false
```

The model policy is already encoded in `docker-compose.yml`:

- DeepSeek for derivation, summaries, observations, and dialectic operations.
- OpenAI `text-embedding-3-small` for embeddings.
- `DREAM_ENABLED=false` during the initial validation window.
- Vector reconciliation every 60 seconds.

Do not add `.env` files to this repository. Coolify is the secret store for this deployment.

### 3. Deploy and validate

Deploy from Coolify, then verify from the host:

```bash
curl --fail --silent --show-error \
  --max-time 10 \
  http://127.0.0.1:18081/health
```

Expected response:

```json
{"status":"ok"}
```

Then verify in Coolify or with Docker:

```bash
docker compose ps
docker compose logs --tail=100 api deriver database database-init redis
```

Required state:

- `api`: healthy
- `database`: healthy
- `redis`: healthy
- `deriver`: running
- `database-init`: exited successfully
- no PostgreSQL or Redis host-published ports

The first deployment starts with an empty database. Honcho migrations are executed by the API entrypoint and `pgvector` is initialized before the API is allowed to start.

## Hermes cutover

Keep the existing Honcho deployment running while validating this repository. Do not change Hermes until all of the following pass:

1. API health check.
2. PostgreSQL health and successful migrations.
3. Redis health.
4. Deriver startup without errors.
5. OpenAI embedding request succeeds.
6. DeepSeek derivation produces observations.
7. A real semantic-search request succeeds through the Hermes Honcho integration.
8. A restart preserves PostgreSQL and Redis data.

For a reversible cutover, either:

- keep the new instance on `18081` and change Hermes' Honcho base URL; or
- stop the old instance, change `HONCHO_API_PORT` to `18080`, and redeploy this Compose application.

Do not run both instances on the same host port.

## Security

- This repository must remain free of credentials and data.
- Do not commit `.env`, database dumps, Docker logs, OAuth tokens, or Hermes state.
- Do not expose the API publicly with `AUTH_USE_AUTH=false`.
- If a public or private HTTPS route is required, enable Honcho authentication and configure `AUTH_JWT_SECRET` before assigning a domain.
- Do not reuse Hermes Codex OAuth credentials as an LLM API key.
- PostgreSQL and Redis should remain private to the Compose network.

## Persistence and rollback

Coolify creates project-scoped persistent volumes for the `pgdata` and `redis-data` declarations. Do not enable destructive volume deletion during redeployments.

To roll back the application without deleting data, redeploy the previous Compose revision or stop the Coolify application. Do not remove its volumes until a verified database backup exists.

This repository intentionally does not migrate data from the direct `honcho-ai01` deployment. A fresh Honcho workspace is acceptable during the initial rollout; data migration can be added later if required.

## Updating Honcho

The Compose build context is pinned to the upstream tag:

```text
https://github.com/plastic-labs/honcho.git#v3.0.12
```

Upgrade only through a reviewed pull request. Update the tag, validate the Compose configuration and deployment, then document migration or compatibility notes in the same change.

## Local validation

The same Compose file can be syntax-checked without exposing secrets:

```bash
cp .env.example .env
chmod 600 .env
# Replace the three placeholders locally.
docker compose --env-file .env config --quiet
docker compose --env-file .env config --services
```

Do not run the local stack on a host where port `18081` is already in use. Use a different `HONCHO_API_PORT` for a parallel validation deployment.

## Repository layout

```text
docker-compose.yml  Coolify-compatible deployment manifest
.env.example       Secret names and safe validation defaults
README.md          Installation, validation, security, and rollback guide
SECURITY.md        Security reporting and handling policy
LICENSE            MIT license
.github/workflows/ CI syntax and secret-scan checks
docs/              Operational notes and troubleshooting
```

## License

MIT. See [LICENSE](LICENSE).
