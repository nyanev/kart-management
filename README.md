# kart-management

Orchestration for the kart-management stack. The application code lives in two
separate repositories; this one holds the compose files, environment templates
and deployment notes that tie them together.

```
kart-management/
├── docker-compose.yml        development
├── docker-compose.prod.yml   production (the VPS runs this file)
├── .env.example              key names for both env files
├── backend/                  clone of kart-management-backend   (gitignored)
└── frontend/                 clone of kart-management-frontend  (gitignored)
```

The two application repos are plain clones rather than submodules: the deploy
workflow does `git pull origin main` inside `backend/`, which submodule pinning
would fight for no benefit at this scale.

## Getting started

```bash
git clone git@github.com:nyanev/kart-management.git
cd kart-management
git clone git@github.com:nyanev/kart-management-backend.git backend
git clone git@github.com:nyanev/kart-management-frontend.git frontend

docker compose up -d --build
```

- Frontend — http://localhost:3000
- API — http://localhost:8001
- Postgres — host port 5433

Ports 5432 and 8000 are deliberately avoided; they are commonly already taken.

### Tests

```bash
docker compose exec backend python manage.py test     # 59 tests
docker compose exec frontend pnpm vitest run
docker compose exec frontend pnpm type-check
```

Telemetry jobs run **inline** when `TELEMETRY_REDIS_URL` is empty, which is why
the test suite needs no broker. Development compose sets it, so dev uses the
real worker.

## Services

| Service | Purpose |
|---|---|
| `db` | Postgres 15 |
| `backend` | Django + DRF API |
| `frontend` | Vite/React |
| `telemetry-redis` | job broker for telemetry ingest (no persistence) |
| `telemetry-worker` | RQ worker; parses `.xrk`/`.vbo` out-of-process |

## Storage and backups

Telemetry uses three tiers with different durability:

| Tier | Where | Durability |
|---|---|---|
| Metadata, laps, channel inventory | Postgres (`postgres_data`) | **back up** |
| Original uploads, content-addressed | `telemetry_blobs` | **back up — source of truth** |
| Derived Arrow files, one per channel | `telemetry_cache` | disposable, rebuilds from blobs |

Sample data never goes in Postgres — a session is ~1.5M samples, held as Arrow
IPC files and memory-mapped on demand.

**A database backup alone is not a restore.** Without the blobs you get rows
describing sessions whose bytes are gone: the library lists them as ready while
every chart request fails, and reparse cannot help.

Blobs are shared between sessions with identical content, so deleting a session
is a soft delete and never unlinks the file. There is **no garbage collection
yet** — telemetry storage grows monotonically. A refcount-aware sweep is needed
before this runs for a long time.

## Deploying

Deployment is triggered by pushing a `deploy-be*` tag to the **backend** repo,
which SSHes to the VPS and runs `docker compose -f docker-compose.prod.yml`.

Two things to know before the first telemetry deploy:

1. **This file must be copied to the VPS.** The server's copy is what actually
   runs; it is not pulled from this repo. Update
   `$DEPLOY_PATH/docker-compose.prod.yml` before tagging.
2. **The deploy script uses `--no-deps backend`**, so it will not start
   `telemetry-redis` or `telemetry-worker`. Bring those up once by hand, or
   amend the workflow in the backend repo:

   ```bash
   docker compose -f docker-compose.prod.yml up -d telemetry-redis telemetry-worker
   ```

Without a worker the API still functions — ingest falls back to running inline
inside a gunicorn worker — but a large upload can then exceed gunicorn's 30 s
timeout and leave the session stuck in `parsing`. Nothing clears that state
automatically, because the reaper runs on worker boot; recover with:

```bash
docker compose -f docker-compose.prod.yml exec backend python manage.py telemetry_reap
```

## Notes

- `libxrk` publishes manylinux wheels only. The backend image must stay
  glibc-based (`python:3.11-slim`); on Alpine pip silently falls back to a
  source build.
- `TELEMETRY_ANTHROPIC_API_KEY` is optional. Empty means the AI analysis
  endpoints return 503 and never create an analysis row.
