# Code Review Exercise: Infra

## Snippet Under Review

```yaml
services:
  backend:
    build: ../backend
    environment:
      APP_KEY: base64:prod-secret-key
      DB_PASSWORD: super-secret-password
      GEMINI_API_KEY: real-provider-key
    ports:
      - "9000:9000"

  postgres:
    image: postgres:latest
    environment:
      POSTGRES_PASSWORD: super-secret-password
    ports:
      - "5432:5432"
```

## Findings

### High: Secrets are committed into Compose configuration

`APP_KEY`, database password, and provider API keys are embedded directly in the Compose file. This leaks secrets through Git history, CI logs, and local copies. Use `.env`, `.env.production`, or managed cloud secrets.

### High: Database is exposed to the host network

Publishing `5432:5432` exposes PostgreSQL outside the internal application network. The database should stay private and accept traffic only from backend services.

### High: Backend PHP-FPM is exposed directly

Publishing `9000:9000` exposes the PHP-FPM runtime instead of routing through Nginx/Caddy. Public traffic should terminate at the frontend gateway or reverse proxy, then route internally.

### Medium: Floating image tag

`postgres:latest` is not reproducible. A future pull can change major/minor behavior unexpectedly. Pin a major-compatible image such as `postgres:16-alpine`.

### Medium: Missing health checks and dependency conditions

The backend can start before PostgreSQL is ready. Add `pg_isready` health checks and `depends_on` conditions for local Compose.

### Medium: No persistent storage

PostgreSQL has no named volume, so data can be lost when the container is recreated. Use a named volume for local/prod-like Compose.

## Suggested Fix

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-forgeflow_db}
      POSTGRES_USER: ${POSTGRES_USER:-forgeflow}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?POSTGRES_PASSWORD is required}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
    networks:
      - forgeflow-internal

  backend-app:
    build:
      context: ../backend
      dockerfile: Dockerfile
    env_file:
      - .env.production
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - forgeflow-internal

volumes:
  postgres-data:
```

## Review Decision

Request changes. The snippet leaks secrets and exposes internal services directly, which is not acceptable for production or prod-like local deployment.
