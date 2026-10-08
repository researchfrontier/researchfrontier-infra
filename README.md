# researchfrontier-infra

Local/self-host orchestration and deployment manifests for **ResearchFrontier**.
Start here to run the whole stack. Architecture and decisions live in
[ARCHITECTURE.md](./ARCHITECTURE.md).

## Prerequisites

- Docker + Docker Compose
- The 4 repos checked out as **siblings** in one folder:

```
researchfrontier/
├─ researchfrontier-frontend/
├─ researchfrontier-backend/
├─ researchfrontier-db/
└─ researchfrontier-infra/   ← you are here
```

## Run everything

```bash
cp .env.example .env         # defaults are fine for local
docker compose up --build
```

Then open:

| Service | URL |
|---------|-----|
| Frontend (SPA) | http://localhost:8080 |
| API docs (Swagger) | http://localhost:8000/docs |
| API health | http://localhost:8000/health |
| Postgres | localhost:5432 (postgres/postgres) |

The database self-initializes with the schema **and a demo seed**, so the UI shows
real content immediately — no API keys needed.

## Live data (optional)

To pull real recent papers instead of the seed, set `RF_OPENALEX_API_KEY` and
`RF_CONTACT_EMAIL` in `.env`, then run the jobs inside the backend container:

```bash
docker compose exec backend python -m app.jobs.sync_taxonomy   # full taxonomy (source of truth)
docker compose exec backend python -m app.jobs.ingest --days 7 # recent papers per field
docker compose exec backend python -m app.jobs.digest --window 3
```

> Run these against a **fresh** database (not the demo seed) to avoid mixing the
> illustrative seed ids with canonical OpenAlex ids. `docker compose down -v`
> drops the volume so the next `up` starts clean (seed) — skip the seed in the db
> image for a pure live setup.

## Reset

```bash
docker compose down -v       # also drops the Postgres volume (re-seeds on next up)
```

## Production topology (phase 1)

- **Frontend** → GitHub Pages (repo `researchfrontier.github.io`).
- **Backend** → Fly.io (`fly.toml` here; `fly deploy` from the backend repo).
- **Database** → Neon serverless Postgres; apply `researchfrontier-db/schema/*.sql`.
- **Jobs** → GitHub Actions cron in the backend repo.

See [ARCHITECTURE.md](./ARCHITECTURE.md) for the full picture.
