# ForgeFlow Infra

ForgeFlow Infra contains Docker Compose orchestration for the separate ForgeFlow backend and frontend repositories. It supports local prod-like development and production VPS deployment for `flowforge.iandev.my.id`.

## Current Branches

Use the existing branches below for infra work:

```bash
git switch master
git switch chore/docker-infra
git switch chore/docker-runtime-config
git switch ci/github-actions-infra
```

No separate production-only branch is required for the current infra flow.

## Repository Layout

The expected local and VPS layout is:

```text
ForgeFlow/
  backend/
  frontend/
  infra/
```

The Compose files use relative build contexts:

- `../backend` for the Laravel API image.
- `../frontend` for the React frontend image.
- `./docker/backend-nginx/default.conf` for the internal backend Nginx proxy.

## Services

Local `compose.yml` services:

- `postgres`: PostgreSQL database.
- `backend-app`: Laravel PHP-FPM runtime.
- `backend-init`: one-shot migration and demo seed job.
- `backend-queue`: workflow execution queue worker.
- `backend-scheduler`: scheduled workflow trigger loop.
- `backend-nginx`: internal Nginx proxy for Laravel public files and `/up` health.
- `frontend`: browser-facing gateway on `http://localhost:8080`.

Production `compose.prod.yml` adds:

- `caddy`: public HTTPS reverse proxy on ports `80` and `443`.
- `backend-migrate`: one-shot production migration job without demo seeding.
- Persistent named volumes for PostgreSQL, backend storage, and Caddy state.

## Local Docker Setup

Create the local environment file and start the stack:

```bash
cp .env.example .env
docker compose up -d --build
docker compose ps
```

On Windows PowerShell, use:

```powershell
copy .env.example .env
docker compose up -d --build
docker compose ps
```

Open:

```text
http://localhost:8080
```

Demo login after `backend-init` seeds the database:

```text
Email: admin@flowforge.test
Password: password
```

Local routing:

```text
http://localhost:8080/        -> frontend
http://localhost:8080/api/*   -> backend API
http://localhost:8080/up      -> backend health
```

## Local Operations

View logs:

```bash
docker compose logs -f backend-app backend-queue backend-scheduler frontend
```

Smoke test:

```bash
curl http://localhost:8080/
curl http://localhost:8080/up
curl -X POST http://localhost:8080/api/v1/auth/login -H "Content-Type: application/json" -d '{"email":"admin@flowforge.test","password":"password"}'
```

Reset local data:

```bash
docker compose down -v
docker compose up -d --build
```

Validate local Compose config:

```bash
docker compose --env-file .env.example config --quiet
```

## Environment Files

Committed examples:

- `.env.example` for local Docker.
- `.env.production.example` for production Docker.

Ignored runtime files:

- `.env`
- `.env.production`

Keep real secrets only in ignored runtime env files. Vite variables are public browser build variables; backend secrets such as `OPENROUTER_API_KEY`, `GEMINI_API_KEY`, `APP_KEY`, and database passwords must stay in backend/infra runtime env files.

## Production VPS Deployment

Production domain:

```text
https://flowforge.iandev.my.id
```

Production routing:

```text
https://flowforge.iandev.my.id/        -> frontend
https://flowforge.iandev.my.id/api/*   -> backend API
https://flowforge.iandev.my.id/up      -> backend health
```

Expected VPS path:

```text
/home/ian/apps/ForgeFlow/
  backend/
  frontend/
  infra/
```

Before starting Caddy, point DNS `flowforge.iandev.my.id` to the VPS public IP.

Start production:

```bash
cp .env.production.example .env.production
nano .env.production
docker compose -f compose.prod.yml --env-file .env.production up -d --build
docker compose -f compose.prod.yml --env-file .env.production ps
```

Required production edits in `.env.production`:

- `POSTGRES_PASSWORD`
- `DB_PASSWORD`
- `APP_KEY`
- `OPENROUTER_API_KEY` when `AI_PROVIDER=openrouter`
- the matching provider key when using another provider, for example `GEMINI_API_KEY`, `OPENAI_API_KEY`, or `ANTHROPIC_API_KEY`
- `APP_URL=https://flowforge.iandev.my.id`
- `VITE_API_BASE_URL=https://flowforge.iandev.my.id/api/v1`

Create the first production admin after migrations finish:

```bash
docker compose -f compose.prod.yml --env-file .env.production exec backend-app php artisan flowforge:create-admin \
  --name="ForgeFlow Admin" \
  --email="admin@flowforge.iandev.my.id"
```

Production logs:

```bash
docker compose -f compose.prod.yml --env-file .env.production logs -f caddy backend-app backend-queue backend-scheduler
```

Production smoke test:

```bash
curl -I https://flowforge.iandev.my.id
curl -I https://flowforge.iandev.my.id/up
curl -X POST https://flowforge.iandev.my.id/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@flowforge.iandev.my.id","password":"YOUR_PASSWORD"}'
```

Production notes:

- Only Caddy exposes host ports `80` and `443`.
- `postgres`, `backend-nginx`, `backend-app`, `backend-queue`, `backend-scheduler`, and `frontend` stay on the internal Docker network.
- `backend-migrate` runs `php artisan migrate --force` and does not seed demo data.
- `backend-storage`, `postgres-data`, `caddy-data`, and `caddy-config` are persistent named volumes.

## CI

The current CI branch is `ci/github-actions-infra`.

The infra GitHub Actions workflow:

- Checks out infra into `infra/`.
- Checks out backend and frontend sibling repositories.
- Prepares `.env` from `.env.example` and `.env.production` from `.env.production.example` for Compose `env_file` resolution.
- Validates local Compose with `docker compose --env-file .env.example config --quiet`.
- Validates production Compose with `docker compose -f compose.prod.yml --env-file .env.production.example config --quiet`.
- Builds the core `backend-app` and `frontend` services.
- Uploads resolved Compose config artifacts.

## Trade-offs

- Local Docker exposes one browser-facing port (`8080`) and routes API traffic through the same origin to avoid browser localhost confusion.
- Production uses Caddy for automatic HTTPS and keeps backend services internal.
- The production migration service does not seed demo users, so the first admin must be created explicitly.
- Compose is intentionally split into local and production files to keep local seeding and production HTTPS concerns separate.
