# ResearchFrontier — Architecture

A front door to the state of the art of a research field: the newest papers, the
directions the field is moving in, and a short Mon/Wed/Fri brief — with clear
peer-reviewed vs preprint labelling and every paper linked to its DOI. Not a
replacement for reading papers; a simplified way in.

## Repositories

| Repo | Role | Runtime |
|------|------|---------|
| `researchfrontier-frontend` | SvelteKit SPA (static) | GitHub Pages (`researchfrontier.github.io`) |
| `researchfrontier-backend` | FastAPI + ingestion/digest jobs | PaaS (Fly.io) |
| `researchfrontier-db` | Canonical SQL schema + seed | Managed Postgres (Neon) / local container |
| `researchfrontier-infra` | docker-compose, deploy manifests, this doc | local / CI |

The three services run as **separate containers** locally via `docker-compose`
(the self-host / dev shape). In the public launch, GitHub Pages serves only the
static frontend; the backend and database run on managed hosts the frontend calls
over CORS — because Pages cannot run a server or a database.

## Data flow

```
 OpenAlex / arXiv / Crossref            (authoritative sources)
            │  ingestion jobs (GitHub Actions cron)
            ▼
   PostgreSQL  ── taxonomy (domain→field→subfield→topic)
            │     works (+ derived peer-review badge, denormalized field pointers)
            │     digests (Mon/Wed/Fri editions)
            ▼
     FastAPI  ── /api/fields/hot · /{id}/papers · /{id}/directions · /{id}/digest
            │     (+ /openapi.json → typed FE client)
            ▼
  SvelteKit SPA (GitHub Pages)  ── home · fields · field page · about
```

## Key decisions (from the design research, 2026)

- **Taxonomy → OpenAlex Topics** (4 domains → 26 fields → 252 subfields → ~4,516
  topics), CC0, continuously maintained; its Oct-2026 classifier tags papers from
  title+abstract with no citations, so day-old papers are classified. New/emerging
  fields appear automatically when the taxonomy sync re-runs. A product
  **"research field" == an OpenAlex subfield**; its **directions == the topics** below it.
- **Sources → OpenAlex backbone** (DOIs, topics, source type) **+ Crossref**
  (DOI authority, work-type signal, freshest deposits) **+ arXiv** (same-day STEM
  preprints). Later: bioRxiv/medRxiv, Europe PMC/PubMed (MeSH), DOAJ (peer-review
  verification). Dedup: DOI → preprint↔published cluster → fuzzy title+author+year.
  Ingestion is watermarked on ingest date, not publication date.
- **Peer-review badge** derived (never guessed) from OpenAlex (source type,
  version, work type, `is_retracted`) with Crossref confirmation for retractions:
  `peer_reviewed` · `preprint_published` · `preprint` · `retracted` · `unknown`,
  each stored with a confidence level and its evidence (shown as "why").
- **Stack → FastAPI + PostgreSQL + SvelteKit.** SvelteKit (`adapter-static`) for
  tiny JS, compiler-checked a11y and first-class motion; typed FE client generated
  from the API's OpenAPI schema (phase 2). Editorial design, not card-and-shadow.
- **Auth (phase 2) → ORCID + Google via OIDC** (Authlib), backend owns the full
  authorization-code flow; the static SPA never holds a client secret. Session via
  **bearer token in the `Authorization` header** (not cookies — Pages→API is
  cross-site and third-party cookies are blocked). ORCID public record (keywords/
  works) seeds a researcher's initial fields of interest.
- **Hosting →** Pages (FE) + Fly.io (API, ~$2/mo, scale-to-zero) + Neon (serverless
  Postgres). Scheduled ingestion + digest on GitHub Actions cron.

## Phases

1. **Slice (this build)** — ingest OpenAlex + arXiv; browse taxonomy; hot fields;
   follow fields (localStorage); field page with badged, DOI-linked papers +
   directions + an in-app brief. No accounts. Ships with a seed so it runs offline.
2. **Accounts** — ORCID/Google sign-in, server-side profile & preferences, ORCID
   seeding, personalised home + Mon/Wed/Fri digest.
3. **Scale** — more sources, retraction monitoring, richer trend visuals (ECharts
   streamgraph / D3 signature visual), optional email delivery.

## Peer-review badge rules (summary)

Top-to-bottom, first match wins (see `researchfrontier-backend/app/services/badges.py`):
1. `is_retracted` (confirm via Crossref `update-to`) → **retracted**
2. primary venue is journal/conference **and** a `publishedVersion` exists → **peer_reviewed**
3. a preprint that also has a published journal copy → **preprint_published**
4. preprint / repository with no published version → **preprint**
5. journal article with no version info → **peer_reviewed** (low confidence)
6. otherwise → **unknown**

## Known slice simplifications

- The bundled DB seed uses illustrative taxonomy ids; a real taxonomy sync is the
  source of truth and should not be mixed with the seed in one database.
- Date windows are measured from the latest paper in the DB in demo mode
  (`RF_DEMO_RELATIVE_DATES=true`) so the seed is always visible; production uses
  the real current date.
- Dedup beyond DOI (fuzzy title+author+year) and the Crossref/DOAJ enrichment
  passes are specified but not yet implemented in the ingestion job.
