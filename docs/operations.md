# Operations guide

## Initial deployment checklist

- [ ] Create a new Coolify Docker Compose application from this repository.
- [ ] Confirm the intended Docker destination.
- [ ] Configure the three required secret variables in Coolify.
- [ ] Keep `HONCHO_API_PORT=18082` for parallel validation.
- [ ] Do not configure a public domain while authentication is disabled.
- [ ] Deploy and verify all health checks.
- [ ] Verify the `vector` PostgreSQL extension.
- [ ] Verify that the Deriver remains running after the API becomes healthy.
- [ ] Run an end-to-end Hermes semantic-search test before cutover.

## Health checks

```bash
curl --fail --silent --show-error http://127.0.0.1:18082/health
```

Inspect the application in Coolify and confirm:

```text
api           healthy
database      healthy
redis         healthy
deriver       running
database-init exited successfully
```

## Database checks

Run these from the Docker host with the Coolify-generated container name, or use the Coolify terminal:

```bash
psql -U "$HONCHO_DB_USER" -d "$HONCHO_DB_NAME" -c \
  "select extname from pg_extension where extname = 'vector';"
```

The application starts with an empty database. Honcho's API entrypoint runs Alembic migrations on startup.

## Cutover strategy

The recommended low-risk sequence is:

1. Keep the existing Honcho deployment on `127.0.0.1:18080`.
2. Deploy this repository on `127.0.0.1:18082`.
3. Run health, provider, Deriver, persistence, and semantic-search tests.
4. Back up Hermes configuration before changing its Honcho URL.
5. Point Hermes at `http://127.0.0.1:18082`.
6. Restart or reload the Hermes process.
7. Verify a new session, message synchronization, embedding, and semantic search.
8. Keep the old deployment available for rollback.

If the application is stable, the old deployment may later be stopped and the Coolify variable changed to `HONCHO_API_PORT=18080`.

## Rollback

To roll back the application layer:

1. Restore Hermes' previous Honcho base URL.
2. Restart or reload Hermes.
3. Stop the Coolify application from Coolify.
4. Confirm the original Honcho health endpoint.

Do not delete Coolify volumes during a normal rollback. Delete data only after an explicit decision and a verified backup.

## Troubleshooting

### API is unhealthy

Inspect API logs first. Common causes are:

- PostgreSQL is not healthy yet;
- the database password contains characters that are not safe in a connection URI;
- an LLM variable is missing from Coolify;
- an incompatible upstream Honcho migration.

Use a long URL-safe PostgreSQL password, for example one generated from letters, digits, `_`, or `-`.

### Deriver is not running

Confirm that the API is healthy and inspect Deriver logs. It intentionally waits for API, PostgreSQL, and Redis health conditions.

### Embeddings remain pending

Check that:

- `OPENAI_API_KEY` is the OpenAI embeddings key;
- the embedding model is `text-embedding-3-small`;
- the base URL is `https://api.openai.com/v1`;
- the reconciler is running;
- the OpenAI key was not accidentally replaced by the DeepSeek key.

### Public exposure

Do not expose the API while `AUTH_USE_AUTH=false`. If a domain is required, enable JWT authentication, configure a strong `AUTH_JWT_SECRET`, and update the Hermes client configuration for the authenticated endpoint.
