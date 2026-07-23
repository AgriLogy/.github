# AgriLogy — Architecture & Staff Handoff

> **Purpose.** One place for any engineer to understand how the platform fits
> together, what state the big migration is in, and where to pick up the next
> feature. All diagrams are Mermaid (render natively on GitHub).
>
> Per-repo build/test/PR instructions live in each repo's `CONTRIBUTING.md`.
> This file is the map; those files are the ground.
>
> _Last updated: 2026-07-23._

---

## 1. Ecosystem at a glance

```mermaid
flowchart TB
    subgraph field["🌾 Field hardware"]
        biv["Bivocom routers<br/>(HTTP push)"]
        lora["LoRaWAN sensors"]
    end

    subgraph edge["Ingest edge"]
        bridge["agri-bridge (Node) :9090<br/>POST / · /uplink"]
        cs["ChirpStack :8081<br/>LoRaWAN network server"]
        mosq[["MQTT broker"]]
    end

    subgraph backend["Backend — droplet 157.245.43.196, behind nginx + TLS"]
        nginx{{"nginx<br/>back.agrogo-datafarm.com"}}
        fast["fastapp (FastAPI/uvicorn) :8001"]
        django["Django shell (gunicorn) :8000"]
        mqttc["agri-api-mqtt<br/>paho subscriber"]
        worker["Celery worker (fastapp)"]
        beat["Celery beat (fastapp)"]
        redis[("Redis — broker/result")]
    end

    subgraph libs["Shared Python libs (tag-pinned)"]
        core["agri-core<br/>logic · handlers · agronomy"]
        db["agri-db<br/>SQLAlchemy models + Alembic"]
    end

    pg[("Postgres — agrydata container")]

    subgraph obs["Observability"]
        promtail["Promtail"] --> loki[("Loki")] --> graf["Grafana<br/>grafana.agrogo-datafarm.com"]
        graf -.->|429 alert| disc["Discord"]
    end

    subgraph clients["Clients (Vercel / stores)"]
        web["agri-web — farmer web<br/>app.agrogo-datafarm.com"]
        admin["agri-admin — back-office"]
        identity["agri-identity — SSO gateway"]
        landing["agri-landing-page — marketing"]
        mobile["agri-mobile (RN) · agri-ios (SwiftUI)"]
    end

    biv --> bridge --> nginx
    lora --> cs --> mosq --> mqttc
    bridge -.->|optional mirror| mosq
    web & admin & identity & mobile --> nginx
    nginx -->|migrated prefixes| fast
    nginx -->|"catch-all /"| django
    fast & django & mqttc --> core --> db --> pg
    worker & beat --> core
    fast & worker & beat <--> redis
    backend -.->|JSON logs| promtail
```

**One-way dependency rule:** `api → core → db`, pinned by git tag, released with
python-semantic-release. Frameworks (Django/DRF/FastAPI) are allowed only in
`agri-api`; `agri-core` and `agri-db` are framework-agnostic (SQLAlchemy only).
`agri-db` is the **schema of record** — nothing else may create or alter tables.

---

## 2. Repository map

| Repo | Role | Stack | Prod branch |
| --- | --- | --- | --- |
| **agri-api** | HTTP API + Celery + MQTT subscriber. Mid-migration Django→FastAPI (§4). | Python / Django + FastAPI | `alpha` |
| **agri-core** | Business logic, endpoint handlers, device adapters, FAO-56 agronomy math | Python / SQLAlchemy | tag |
| **agri-db** | Schema of record — SQLAlchemy models + the Alembic chain | Python / Alembic | tag |
| **agri-bridge** | Device-uplink bridge (router push → backend ingest) | Node.js | — |
| **agri-web** | ★ The farmer web app (Turborepo: `apps/web` + `@agri/*`) | Next.js | `alpha` |
| **agrilogy-front** | Predecessor farmer web app — **maintenance only**, superseded by agri-web | Next.js | `alpha` |
| **agri-admin** | Operations back-office console | Next.js | `main` |
| **agri-identity** | SSO gateway — single login, role-routed | Next.js | `main` |
| **agri-landing-page** | Public marketing site | Next.js | `main` |
| **agri-mobile** | Android app | React Native / Expo | `main` |
| **agri-ios** | Native iOS app | Swift / SwiftUI / XcodeGen | `main` |
| **shared-workflows** | Reusable CI workflows called by the other repos (`@v1`) | GitHub Actions | `main` |
| **python-template** | Starting point for new Python repos | Python / uv | `main` |
| **.github** | Org profile, default PR template, this document | Markdown | `main` |

Every repo carries a `CONTRIBUTING.md` at its root with the exact setup, dev-loop
and CI commands for that repo. **All AgriLogy org repos are PUBLIC** as of
2026-07-23 (done to restore CI when private-repo Actions minutes were exhausted).
Histories were credential-scanned first; the droplet IP + topology are now public,
which is why the firewall items below became urgent.

---

## 3. Domain model — who owns what data

```mermaid
erDiagram
    USER ||--o{ SECTOR : "organises"
    SECTOR ||--o{ ZONE : "groups"
    ZONE ||--o{ DEVICE : "hosts"
    DEVICE ||--o{ SENSOR : "exposes"
    DEVICE ||--o{ READING : "emits"
    ZONE ||--o{ ALERT : "triggers"
    USER ||--o{ NOTIFICATION : "receives"
    ZONE ||--o{ IRRIGATION_PROGRAM : "schedules"
```

- One farm per user. **Sector** is a purely organisational grouping (no geometry)
  sitting between the user and their zones; `/sectors` CRUD +
  `PUT /sectors/{id}/zones` + `/sectors/overview` back the `/farm` page in agri-web.
- **Ownership is device-keyed.** Readings are keyed by `device_id`; who owns a
  reading is resolved by joining the device row. Transferring a device between
  accounts is therefore a one-row update, not a data migration.

---

## 4. The big migration — Django → FastAPI strangler ⚠️ (~95% done, live)

The backend is a **strangler-fig migration**: fastapp (native FastAPI, `:8001`)
progressively takes prefixes from the legacy Django shell (`:8000`). nginx is the
switch — each phase adds a `location` block routing one prefix to fastapp, above
the catch-all `location /` that still falls through to Django.

**Source of truth for routing:** `agri-api/deploy/nginx/back.conf` (applied
manually on the droplet — auto-sync is a tracked follow-up).

```mermaid
flowchart LR
    req["Request →<br/>back.agrogo-datafarm.com"] --> nginx{{nginx}}

    nginx -->|"/weather /sensors /alerts /kc<br/>/notifications /zones /sectors<br/>/devices /technicians /irrigation<br/>/assistant /ingest · most /admin/*"| fast["fastapp :8001<br/>✅ MIGRATED"]

    nginx -->|"catch-all location /"| dj["Django :8000<br/>⏳ REMAINS"]

    subgraph remains["Still served by Django"]
        dj --> a1["/admin/ Django admin site"]
        dj --> a2["/admin/db generic CRUD<br/>(ported but HELD)"]
        dj --> a3["/users admin tree<br/>(except /users/me)"]
        dj --> a4["/auth/token legacy DRF JWT"]
        dj --> a5["/swagger /redoc drf-yasg"]
    end

    style fast fill:#2f855a,color:#fff
    style dj fill:#c05621,color:#fff
```

### Phase ledger

```mermaid
flowchart TB
    F1["F1 ✅ auth-parity probe"] --> F2["F2–F5 ✅ all user-facing HTTP<br/>weather, sensors, alerts, kc,<br/>notifications, zones, devices,<br/>technicians, irrigation"]
    F2 --> F6["F6 ✅ business-admin /admin/*"]
    F6 --> F7["F7 ✅ assistant/chat"]
    F7 --> F8["F8 ✅ all 13 Celery tasks ported"]
    F8 --> F9["F9 ✅ /auth login + token issuance<br/>+ /ingest device webhooks"]
    F9 --> F10["F10 ✅ Celery worker + beat<br/>cut to native fastapp app"]
    F10 --> F11["F11 ⏳ flip nginx default<br/>location / → fastapp"]
    F11 --> F12["F12 ⏳ delete Django"]

    style F11 fill:#c05621,color:#fff
    style F12 fill:#9b2c2c,color:#fff
```

### What is LEFT in Django (the last ~5%)

Django is down to **one container** (`agri-api-web`). Worker, beat and the MQTT
subscriber are fully on fastapp. Everything below is reached only via the nginx
catch-all:

| Surface | Why it's still on Django |
| --- | --- |
| `/admin/db/*` generic CRUD | **The real F12 blocker.** Ported to fastapp (`fastapp/routers/admin_db.py`, introspects SQLAlchemy metadata) but **HELD** — it can't reproduce Django's `verbose_name` / `help_text` / `choices` / `editable` field metadata that agri-admin's *Database* page renders. Core CRUD contract already byte-matches. |
| `/admin/` Django admin site | No fastapp equivalent; decide if staff still need the built-in admin UI. |
| `/users` admin tree (except `/users/me`) | fastapp has the router, but the `/users` prefix is shared with a still-Django users-admin router — the edge can't cleanly split it yet. |
| `/auth/token/*` legacy DRF SimpleJWT | Kept for older clients; new `/auth/sessions` + `/auth/signup` already on fastapp. |
| `/swagger`, `/redoc` (drf-yasg) | Low priority; FastAPI's built-in `/docs` can replace it. |

**Next two steps:**
1. **F11** — flip nginx `location /` default to fastapp; pin only `/admin/`,
   `/admin/db` (and optionally `/auth/token`) back to `:8000`. Lowest-risk step —
   every migrated path is already explicitly routed.
2. **F12** — delete Django, gated on: a field-metadata source for `/admin/db`,
   the `/admin/` site decision, and resolving the `/users` prefix split.

---

## 5. Ingest — two transports, one pipeline

HTTP push was first; MQTT was added **additively** — same handlers, same
ownership rules, different transport.

```mermaid
sequenceDiagram
    participant HW as Field device
    participant CS as ChirpStack / router
    participant EDGE as agri-bridge · MQTT broker
    participant API as agri-api (fastapp / agri-api-mqtt)
    participant CORE as agri-core handlers
    participant PG as Postgres

    alt HTTP push (Bivocom)
        HW->>CS: raw uplink
        CS->>EDGE: POST /uplink
        EDGE->>API: POST /ingest (translated payload)
    else MQTT (LoRaWAN)
        HW->>CS: LoRaWAN uplink
        CS->>EDGE: publish application/<app>/device/<eui>/event/up
        EDGE-->>API: subscriber receives message
    end
    API->>CORE: normalise → resolve device → map sensors
    CORE->>PG: INSERT reading (keyed by device_id)
    Note over CORE,PG: Ownership resolved by JOIN on the device row —<br/>transferring a device is a one-row update.
```

> A device is moved from HTTP to MQTT by deleting its ChirpStack HTTP
> integration. **MQTT is now the only route from ChirpStack** —
> `application_integration` holds no HTTP rows. `MQTT_HOST` lives in
> `/root/agri-api/.env` (the compose-project env file; compose `environment:`
> overrides `back/.env`, so editing the repo one changes nothing).
> `agri-api-mqtt` is **not** recreated by the deploy workflow — force-recreate it
> after each deploy (see `agri-api/CONTRIBUTING.md`).

---

## 5b. Operational state — what is actually live

_Snapshot 2026-07-23. Established by read-only investigation; keep it honest._

**There are exactly two real hardware feeds in production** (both currently
**offline on purpose** — the owner disconnected the devices). Everything else in
the database is simulated or seeded — worth knowing before trusting a dashboard.

| Feed | Owner | Transport | State |
| --- | --- | --- | --- |
| Router02 **weather** (wind, temp, humidity, pressure, solar) | user 4 / zone 1 | Bivocom HTTP → `agryserver:9090` → legacy `agrybackend` | ✅ live |
| ET₀ / VPD for zone 1 (derived) | user 4 / zone 1 | Celery `compute_et0_vpd_hourly` | ✅ live |
| Router02 **soil probe** (moisture / temp / EC × low+medium) | user 4 / zone 1 | same Bivocom path, separate POST stream | ⏸ owner-disconnected since 2026-07-13 |
| LoRa node `ph` (`a840412ca45d64d0`) | user 8 / zone 11 | ChirpStack → MQTT → `agri-api-mqtt` | ⏸ owner-disconnected since 2026-07-13 |
| Simulator, ~30 tables, zones 6 & 7 | user 1 | Celery `simulate_sensor_ingest` | ⏹ retired by the F10 beat cutover |

The soil-probe and LoRa feeds stopping was **not a failure** — the hardware was
unplugged deliberately (investigated in agri-api#430). The last reading in each
case was received and stored cleanly, then simply stopped: the signature of a
clean disconnect. `agri-api-mqtt` and the broker path were repaired anyway (#426
subscriber-on-broker-network, #427 broker rebound to loopback), so LoRa ingest
resumes the moment those devices are reconnected.

> **If staleness alerting is ever added** (agri-api#429): a naive `max(received_at)`
> alert would page for these intentionally-off devices. It must watch only feeds
> meant to be always-on, or support marking a device "offline on purpose."
> Device-health scanning itself is fixed (#433) — it was silently failing on a
> missing column before.

### Schema drift — a bug class now closed at the source

Production was long **stamped behind** the Alembic head (the DB was hand-built by
ensure-scripts, so the stamp understated what physically existed). As of
2026-07-23 the cutover is complete: **prod is stamped at head**, and
`RUN_DB_MIGRATIONS=true` is on, so every deploy runs `upgrade head` before the
app containers boot. A schema-dependent deploy no longer 500s against an
un-migrated column (the RBAC `access_level` deploy did exactly that — it took the
whole authenticated API down for ~3 min until the migration was applied by hand;
that class is now prevented, agri-api#448).

**The durable rule this leaves:** code touching an object whose migration may not
be applied yet must **degrade, not raise** — copy the `sectors.py` `begin_nested()`
SAVEPOINT, or the cached probe in `fastapp/schema_compat.py`. Belt-and-suspenders
on top of auto-apply.

---

## 6. Data flow — ET₀ 7-day forecast (worked example)

The dashboard's **"ET₀ — prévision sur 7 jours"** widget is a good end-to-end
example of a fully-migrated feature. ⚠️ **Its weather input is currently
synthetic** (a deterministic mock), while the ET₀ math is real FAO-56.

```mermaid
sequenceDiagram
    participant UI as agri-web<br/>EtForecastMain.tsx
    participant NG as nginx
    participant FA as fastapp<br/>routers/weather.py
    participant DB as agri-db (SQLAlchemy)
    participant PV as forecast_provider.py
    participant CO as agri-core<br/>et_forecast (FAO-56)

    UI->>NG: GET /weather/et-forecast?zone_id&days=7
    NG->>FA: proxy :8001
    FA->>DB: zone (elevation) + user (lat/lon)
    DB-->>FA: owner-scoped row (404 if not owner)
    FA->>PV: get_daily_forecast(start, days, lat)
    Note over PV: ET_FORECAST_PROVIDER=mock (default)<br/>→ deterministic seasonal model.<br/>"openweather" is a stub → falls back to mock.
    PV-->>FA: daily weather series (synthetic)
    FA->>CO: et0_forecast(daily, lat, lon, elev)
    CO-->>FA: [{date, et0_mm}] (real FAO-56 Penman-Monteith)
    FA-->>UI: {zone_id, provider, days}
    Note over UI: Graph renders a source label from `provider`:<br/>"demo model" while mock, real name once wired.
```

**To make it a real forecast:** implement `_openweather_forecast` (or add an
Open-Meteo provider — no API key needed) in
`agri-api/back/src/apps/sensors/forecast_provider.py`, then set
`ET_FORECAST_PROVIDER` + `WEATHER_API_KEY`. The frontend label updates
automatically. This provider will move into `agri-core` in a later phase.

---

## 7. Background jobs (Celery)

All 13 tasks + beat run on the **native fastapp Celery app**
(`fastapp/celery_app.py`), queue `agriapi`. The old Django worker/beat are gone.

```mermaid
flowchart LR
    beat["beat (fastapp)<br/>DatabaseScheduler"] -->|periodic| Q[("Redis queue<br/>agriapi")]
    api["HTTP handlers<br/>send_task"] -->|on-demand| Q
    Q --> worker["worker (fastapp)"]

    worker --> comms["comms: alert email/digest,<br/>WhatsApp, SMS, zone-outbound"]
    worker --> compute["compute: et0_vpd_hourly,<br/>periodic_notifications, idle_zones,<br/>device_health, proactive_insights,<br/>due_irrigation_programs"]
```

> Beat uses `DatabaseScheduler` — live cadences come from `PeriodicTask` rows,
> not the static code schedule. On-demand `send_task` must target the `agriapi`
> queue. Outbound email goes over Resend's HTTPS API (the droplet blocks SMTP);
> quota exhaustion surfaces as a `quota_exceeded` log event and a Grafana alert.

---

## 8. Observability

```mermaid
flowchart LR
    subgraph app["agri-api processes"]
        w["web"] & wk["worker"] & bt["beat"] & mq["mqtt"]
    end
    app -->|"structured JSON logs<br/>+ request-id"| pt["Promtail"]
    pt --> lk[("Loki")]
    lk --> gf["Grafana<br/>grafana.agrogo-datafarm.com :3300"]
    gf --> d1["Dashboard: API traffic"]
    gf --> d2["Dashboard: notifications"]
    gf -->|"Resend 429 rule"| dc["Discord contact point"]
```

Stack lives in `agri-api/deploy/observability/` on its own compose file.
`LOG_LEVEL` / `LOG_FORMAT` are the knobs; log rotation is configured on the app
compose.

---

## 9. Release & deploy flow

```mermaid
flowchart LR
    pr["Feature branch<br/>+ PR (Closes #N)"] -->|squash, conventional title| main["main = STAGING"]
    main -->|semantic-release| ver["auto version + deploy staging"]
    main -->|manual merge at release time| alpha["alpha = PRODUCTION"]
    alpha -->|auto-deploy| prod["🚀 production"]

    schema["agri-db migration"] -.->|apply BEFORE the api deploy<br/>that expects it| main
```

- **`main` = staging** — every merge auto-deploys + cuts a semantic-release version.
- **`alpha` = production** — promoted manually (`main` → `alpha`); never auto-synced.
- PR titles must be Conventional Commits (enforced) — they become release commits.
- **Release order across repos:** `agri-db` → `agri-core` → `agri-api`; downstream
  repos pin upstream by git tag.
- **Schema order:** now automated — `RUN_DB_MIGRATIONS=true` on the droplet runs
  `agri-migrate upgrade head` before app containers boot, so schema lands with the
  code. A failed migration aborts the deploy (`script_stop`) — the safe failure.
- Shared CI lives in `shared-workflows`, consumed as `@v1`.

### Environments

| Environment | Frontend | Backend | Database |
| --- | --- | --- | --- |
| local | `npm run dev` | `make up` (compose) | local Postgres via agri-db |
| staging | Vercel preview / staging project | agri-api `*-staging` stack on the droplet (`:90xx`) | `agrydata-staging` |
| production | Vercel prod (`app.` / `admin.` / `identity.`) | agri-api stack behind `back.agrogo-datafarm.com` | `agrydata` |

> The staging tier is **started but BLOCKED** (agri-api#418): DNS is live
> (`staging.agrogo-datafarm.com` → the droplet) and the stack is defined, but the
> 2 GB droplet has only ~550 MB free — it cannot host a second stack + Postgres
> without OOM-risking prod. Needs a resize (2→4 GB) or a lighter design.

---

## 10. Auth & SSO

```mermaid
flowchart LR
    u((User)) --> id["agri-identity<br/>identity.agrogo-datafarm.com"]
    id -->|"POST /auth/sessions"| api["agri-api"]
    api -->|"access + refresh"| id
    id -->|"set cookie agrogo_sso<br/>Domain=.agrogo-datafarm.com"| br[(Browser)]
    br --> web["agri-web"] & admin["agri-admin"]
    web & admin -->|hydrateSsoSession| sess["local session"]
    id -->|"is_staff ? admin : web"| route{{role routing}}
```

The cookie is deliberately **not** httpOnly so sibling apps can adopt it
client-side; it is `secure`, `sameSite=lax`, scoped to the shared parent domain —
which is also why cross-app SSO cannot be exercised on `localhost`. Adding a new
consumer app means adding its origin to the backend's CORS/CSRF allowlists.
A staff account can force-logout any user: `sessions_revoked_at` invalidates every
token issued before that instant.

---

## 11. Contributing

Every repo has a root `CONTRIBUTING.md` with its verified setup, dev loop, layout,
a worked example, CI and deploy path. Platform-wide rules:

1. **Clone siblings side-by-side** — `agri-api`, `agri-core`, `agri-db` expect to
   sit in the same parent dir (tag-pinned git deps; a PAT is needed to fetch the
   private `agri-db` package).
2. **Schema first:** `cd agri-db && make upgrade-dev` **before** `cd agri-api && make up`.
3. **Never** run `manage.py migrate` in agri-api — schema is owned by agri-db/Alembic.
4. **PR rules:** one dedicated, scope-matched issue per PR (`Closes #N`), issue and
   PR both assigned to the author; Conventional-Commit titles; branch off `main`.
5. Every board item carries a **Size + Estimate** weight — set it when the ticket
   is created, not after the fact.
6. Each repo's `CLAUDE.md` + `.claude/` holds repo-specific gotchas.

---

## 12. Where to pick up next

**Backend migration (finish the strangler)**
- ⚠️ **F11** — nginx default flip to fastapp (§4).
- **F12** — delete Django; first unblock `/admin/db` field metadata.
- Auto-sync `deploy/nginx/back.conf` into the deploy pipeline (currently manual).

**Platform**
- ⚠️ **Staging environment** — BLOCKED on droplet RAM (§9, agri-api#418).
- ⚠️ **Firewall** — Postgres 5432 + Redis 6379 are now closed via host `DOCKER-USER`
  iptables (SSH-safe, but **not reboot-durable**). The proper fix is a DO Cloud
  Firewall at the edge (needs a DO API token). mosquitto 1883 rebound to loopback
  (#427). Adminer 8080 already closed. This is the top open security item now that
  the repos are public.
- Deploy leaves the Celery worker/beat containers stranded in `created`; the fix is
  a `start_period` on the web healthcheck.

**Shipped 2026-07-23** (all five field-review features live in prod)
- **MAP-1** device GPS · **GRP-1** sensor groups · **CAL-1** per-sensor calibration
  (incl. single-point read-path application) · **USR-1** RBAC Admin/Editor/Monitor
  end-to-end · **RPT-1** alert + irrigation-decision history & report view.
- Prod schema cutover to head; `RUN_DB_MIGRATIONS` turned on (§9).
- MQTT ingest path repaired (#426/#427); assistant now renders Markdown (#105).

**Weather / agronomy**
- ⚠️ **Real ET₀ forecast provider** — wire Open-Meteo (keyless) or OpenWeather in
  `forecast_provider.py`; replaces the demo mock (§6).
- Move the forecast provider into `agri-core`.

**Notifications & alerts**
- Per-alert channel matrix (email/SMS/WhatsApp) — partly built.
- Custom notification zones / geofencing.
- Resend daily-quota ceiling — periodic emails 429 on the free tier.

**Product**
- Chatbot assistant — swap the mock engine for the real backend (`/assistant/*`
  routers already exist on fastapp).
- Header Sector·Zone picker across the web app (§3).
- agri-mobile: output-command API + push/FCM are the two backend gaps.

---

_See each repo's `CONTRIBUTING.md`, `CLAUDE.md` and `docs/` for depth. This file is
the map, not the territory — keep it current when a phase lands._
