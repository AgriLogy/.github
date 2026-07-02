# AgriLogy

Smart-farming platform: field sensors (Bivocom routers, LoRaWAN/ChirpStack) → ingestion → agronomy analytics (ET, irrigation, alerts) → farmer web/mobile apps and an operations back-office.

## Repository map

| Repo | What it is | Stack |
| --- | --- | --- |
| [agri-api](https://github.com/AgriLogy/agri-api) | Backend API (thin Django shell, django-ninja routers, Celery workers) | Python / Django |
| [agri-core](https://github.com/AgriLogy/agri-core) | Business logic, endpoint handlers, device adapters — framework-agnostic library | Python / SQLAlchemy |
| [agri-db](https://github.com/AgriLogy/agri-db) | Schema of record: SQLAlchemy models + Alembic migration chain | Python / Alembic |
| [agri-bridge](https://github.com/AgriLogy/agri-bridge) | Device-uplink bridge (router push → backend ingest) | Node.js |
| [agri-front](https://github.com/AgriLogy/agrilogy-front) | Farmer-facing web app | Next.js |
| agri-admin | Operations back-office console | Next.js |
| agri-identity | SSO gateway (single login, role-routed) | Next.js |
| [agri-landing-page](https://github.com/AgriLogy/agri-landing-page) | Public marketing site | Next.js |

## Architecture (3-tier backend)

```
                 ┌────────────────────────────┐
  devices ──────►│  agri-bridge  (Node ingest)│
  (Bivocom,      └─────────────┬──────────────┘
   ChirpStack)                 ▼
┌──────────────────────────────────────────────┐
│  agri-api      Django shell: HTTP, auth,     │
│                Celery, django-ninja routers  │
└─────────────────────┬────────────────────────┘
                      ▼  tag-pinned git dependency
┌──────────────────────────────────────────────┐
│  agri-core     business logic + handlers +   │
│                DB access (SQLAlchemy)        │
└─────────────────────┬────────────────────────┘
                      ▼  tag-pinned git dependency
┌──────────────────────────────────────────────┐
│  agri-db       ORM models + Alembic chain    │
│                (schema of record)            │
└──────────────────────────────────────────────┘

frontends (agri-front / agri-admin / agri-identity) ──► agri-api
```

Dependencies flow one way: **api → core → db**, pinned by git tag and released with python-semantic-release. Shared CI lives in [shared-workflows](https://github.com/AgriLogy/shared-workflows).

## Release flow

- **`main` = staging** — every merge auto-deploys to staging and cuts a semantic-release version from conventional commits.
- **`alpha` = production** — promoted manually by merging `main` → `alpha` at release time; `alpha` auto-deploys to production.
- Squash-merge titles must be Conventional Commits (enforced by the PR-title gate) — they become the release commits.
- **Deploy order for schema changes:** apply agri-db migrations **before** deploying the agri-api version that expects them.
