# ForgeFlow Infra

Docker orchestration for ForgeFlow local prod-like and VPS production deployments.

## Branch

Use these infra branches for Docker orchestration work:

```bash
git switch -c chore/docker-infra
git switch -c chore/docker-production
```

## Services

- `frontend`: public gateway on `http://localhost:8080`.
- `backend-nginx`: internal Laravel nginx proxy.
- `backend-app`: Laravel PHP-FPM runtime.
- `backend-init`: one-shot database migration and seeding.
- `backend-queue`: workflow execution worker.
- `backend-scheduler`: scheduled workflow runner loop.
- `postgres`: local PostgreSQL database.

## Start

```bash
copy .env.example .env
docker compose up -d --build
docker compose ps
```

Open:

```text
http://localhost:8080
```

Demo login:

```text
admin@flowforge.test
password
```

## Logs

```bash
docker compose logs -f backend-app backend-queue backend-scheduler frontend
```

## Smoke Test

```bash
curl http://localhost:8080/
curl http://localhost:8080/up
curl -X POST http://localhost:8080/api/v1/auth/login -H "Content-Type: application/json" -d "{\"email\":\"admin@flowforge.test\",\"password\":\"password\"}"
```

## Reset Local Data

This removes local database volume.

```bash
docker compose down -v
docker compose up -d --build
```

## Notes

- Backend is not exposed directly to the host by default.
- Browser API traffic uses the same origin through `http://localhost:8080/api/*`.
- Keep real secrets in `.env`; only `.env.example` is committed.

## Production VPS

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

Expected VPS layout:

```text
/home/ian/apps/ForgeFlow/
  backend/
  frontend/
  infra/
```

Start production:

```bash
cp .env.production.example .env.production
nano .env.production
docker compose -f compose.prod.yml --env-file .env.production up -d --build
docker compose -f compose.prod.yml --env-file .env.production ps
```

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
- `postgres`, `backend-nginx`, `backend-app`, `backend-queue`, `backend-scheduler`, and `frontend` stay internal.
- `backend-migrate` runs `php artisan migrate --force` only; it does not seed demo data.
- `backend-storage`, `postgres-data`, `caddy-data`, and `caddy-config` are persistent named volumes.
- Point DNS `flowforge.iandev.my.id` to the VPS public IP before starting Caddy.
