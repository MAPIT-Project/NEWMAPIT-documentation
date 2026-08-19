# Deployment runbook

**Status:** candidate runbook; production rollout blocked by open P0s  
**Last verified against repository:** 20 July 2026  
**Selected topology:** one Ubuntu/Debian VPS, NGINX, SvelteKit adapter-node under PM2, FastAPI under systemd, Redis and Neo4j in Docker, external/managed Supabase  
**Owner:** TBD

This is the only supported deployment path in the current documentation. The existing Kubernetes assets are **not deployable**: they start a nonexistent `app.services.celery_app`, use a nonexistent `/metrics` endpoint, tag `latest`, and do not deploy a frontend. `MAP-IT_SA/docker-compose.yml` is likewise not a production API manifest: it uses `--reload`, a source bind mount, and an external local Supabase network.

> **Release blocker:** the latest quality baseline is 54.8% and “NOT PRODUCTION READY”; frontend checks and backend test collection are not green, and critical security issues remain open. Use this runbook for a restricted staging environment for now. Production requires that the release criteria in [KNOWN_ISSUES_AND_RISKS](KNOWN_ISSUES_AND_RISKS.md) and [SECURITY_REGISTER](SECURITY_REGISTER.md) be closed or formally accepted.

## Topology

```text
Internet :443
    |
    v
NGINX (TLS, rate limit, only 22/80/443 public)
    |
    v
SvelteKit adapter-node, PM2, 127.0.0.1:3000
    |                         |
    | server-side proxy      +--> managed Supabase Auth/REST/Realtime (TLS)
    v
FastAPI/systemd, 127.0.0.1:8000
    |              |              |
    v              v              v
PostgreSQL/     Redis Docker   Neo4j Docker
Supabase TLS    127.0.0.1:16379 127.0.0.1:17687
```

The browser uses the public Supabase URL and anon key. The service role key remains server-side only. FastAPI is not publicly accessible while backend authentication/authorization is not enforced.

## 1. Prerequisites

### Server and accounts

The following sizing is a **starting point, not benchmark-based**:

| Component | Minimum for staging/small production | Note |
|---|---:|---|
| VPS | 4 vCPU, 8 GiB RAM, 80 GiB SSD | Neo4j is limited in Compose to 2 GiB heap plus 512 MiB page cache; Supabase runs externally. |
| OS | Current supported Ubuntu LTS or Debian stable, x86_64 | Install security updates automatically. |
| Python | 3.11.x | `pyproject.toml` targets py311 for formatter/linter; the Dockerfile uses 3.13, but this path does not use an API container. |
| Node | 22.x LTS + npm | Aligns with `@types/node` 22; the repo defines no `engines`, so pin the selected runtime in server management. |
| Docker | Engine with Compose v2 | Only Redis 7-alpine and Neo4j 5.22.0-community from Compose. |
| NGINX/PM2 | Current distribution version / recorded PM2 version | Record the versions actually installed in the deployment record. |
| Supabase/PostgreSQL | Managed Supabase project with pgvector and schema | The current repository cannot yet safely migrate an empty production project; see below. |

Create a non-root service account, for example `mapit`, and this layout:

```text
/srv/mapit/releases/<release-id>/MAP-IT_SA
/srv/mapit/releases/<release-id>/webapp-ui
/srv/mapit/current/backend -> release/backend
/srv/mapit/current/frontend -> release/frontend
/srv/mapit/backups
/etc/mapit/backend.env
/etc/mapit/frontend.env
```

Permissions: release files owned by `mapit`, `/etc/mapit/*.env` root-owned with group `mapit` and mode `0640`, backup directory not web-accessible. Allow only SSH, HTTP, and HTTPS through the firewall. Ports 3000, 8000, 16379, 17474, and 17687 must not be externally accessible. Restrict SSH to management IPs and use keys, not password login.

### DNS and TLS

1. Create an A/AAAA record for the selected domain pointing to the VPS.
2. Install NGINX and Certbot.
3. First install the HTTP configuration, then request a certificate and enable the TLS configuration.
4. Check automatic renewal with `certbot renew --dry-run`.

### External dependencies

- An existing, backed-up Supabase project with database URL, project URL, anon key, and service role key.
- At least one working AI provider key; the default is Gemini.
- Google/Gemini embedding key if vector search is active.
- A Mapbox public token with permitted origins and minimal scopes.
- DNS/TLS and a secret store or root-only environment files.

## 2. Release identity and preflight

Never deploy a floating branch or `latest`. Record the commit SHA for both repositories, release time, operator, database schema hash, and data version.

```bash
git -C MAP-IT_SA rev-parse HEAD
git -C webapp-ui rev-parse HEAD
sha256sum webapp-ui/supabase_cli/schema.sql MAP-IT_SA/migration/*.sql
```

Hard pre-deployment gates:

```bash
cd MAP-IT_SA
source venv/bin/activate
make test
make lint

cd ../webapp-ui
npm run check
npm run lint
npm run build
```

The baseline from 20 July 2026 does not meet these gates. **Stop** on any error; do not use `npm run host` as a deployment gate, because the shell script may still start PM2 after the build step.

Also check:

- no secret or real `.env` in Git/build artifact;
- all critical/high blockers are closed or accepted in writing;
- backup and restore test are more recent than the agreed interval;
- schema, code, and embeddings use the same dimension;
- staging smoke, login, chat, V4, and rollback tests passed.

## 3. Environment configuration

### Backend `/etc/mapit/backend.env`

| Variable | Required | Production value/rule |
|---|---|---|
| `DATABASE_URL` | yes | TLS PostgreSQL URL from Supabase; use a server/private connection string. |
| `SUPABASE_URL` | yes | `https://<project>.supabase.co`. |
| `SUPABASE_ANON_KEY` | recommended | Public anon key; not the service role. |
| `SUPABASE_SERVICE_KEY` | yes | Service role key, exclusively in this private file. |
| `JWT_SECRET_KEY` | yes | Unique random secret, identical to frontend; at least 32 random bytes. |
| `GEMINI_API_KEY`, `OPENAI_API_KEY`, `OPENROUTER_API_KEY` | at least one | Populate only configured providers. |
| `DEFAULT_AI_PROVIDER` | yes | `gemini`, `openai`, or `openrouter`; code default is `gemini`. |
| `GOOGLE_EMBEDDING_API_KEY` | for vectors | Key for `gemini-embedding-001`; code falls back to Gemini key. |
| `EMBEDDING_MODEL` | for vectors | `models/gemini-embedding-001`. |
| `EMBEDDING_DIMENSION` | for vectors | `3072`; database must first be migrated and re-embedded. |
| `VECTOR_SEARCH_ENABLED` | yes | `false` while schema/data are not fully 3072; otherwise `true`. |
| `REDIS_URL` | yes | `redis://127.0.0.1:16379/0`. |
| `REDIS_HOST`, `REDIS_PORT` | yes | `127.0.0.1`, `16379`; short-term memory reads these separately. |
| `NEO4J_URI`, `NEO4J_USER`, `NEO4J_PASSWORD` | yes | `bolt://127.0.0.1:17687`, `neo4j`, unique secret. |
| `CORS_ORIGINS` | yes | JSON list containing exactly the HTTPS domain, for example `["https://mapit.example"]`; never `*` with credentials. |
| `DEBUG` | yes | `false`. |

Do not copy any placeholder value from `.env.example`. The Compose default `mapit_secure_password_2024` is not suitable for production.

### Frontend `/etc/mapit/frontend.env`

| Variable | Required | Production value/rule |
|---|---|---|
| `PRIVATE_DATABASE_URL` | yes | Private TLS database URL for Drizzle/server routes. |
| `PUBLIC_SUPABASE_URL`, `VITE_SUPABASE_URL` | yes | Public HTTPS project URL. |
| `PUBLIC_SUPABASE_ANON_KEY`, `VITE_SUPABASE_ANON_KEY` | yes | Anon key only. |
| `SUPABASE_SERVICE_ROLE_KEY` | admin routes only | Server-private service role; never as `PUBLIC_*` or `VITE_*`. |
| `BACKEND_URL` | yes | `http://127.0.0.1:8000`. |
| `JWT_SECRET_KEY` | yes | Exactly the same strong secret as the backend. |
| `MAPBOX_API_KEY` | for map | Origin-restricted public token; first remove/rotate the hardcoded token. |
| `PUBLIC_APP_URL` | yes | Canonical HTTPS origin. |
| `PUBLIC_APP_ENV` | yes | `production`. |
| `PUBLIC_ENABLE_DEBUG_TOOLS` | yes | `false`. |
| `PORT`, `HOST`, `ORIGIN` | yes | `3000`, `127.0.0.1`, `https://<domain>`. |

Do **not** define `PUBLIC_SUPABASE_SERVICE_KEY`. Anything with `PUBLIC_` or `VITE_` can end up in browser code/build output. Static public SvelteKit variables must be present during `npm run build`; a change therefore requires a rebuild.

## 4. Database preparation and migrations

The repository does not yet have a proven production migration chain. The full `webapp-ui/supabase_cli/schema.sql` is a dump including Supabase system objects; do not execute it blindly on a managed project. Backend `001_init_schema.sql` does not cover all frontend tables, and `008`/`009` plus Drizzle also exist. See the current ledger in [DATABASE_SCHEMA](MAP-IT_SA/docs/DATABASE_SCHEMA.md).

Therefore, two paths apply:

1. **Existing controlled production project:** create a backup, compare objects/dimensions using the queries in `DATABASE_SCHEMA.md`, execute only reviewed migrations that have not yet been applied, and record checksum/time/operator.
2. **Empty project:** **stop the deployment** until a clean staging rehearsal has fully proven the canonical chain. Pasting a dump into Supabase Studio is not an accepted production procedure.

The 768→3072 migration is destructive for three vector columns. Only after a backup, maintenance window, and a proven re-embedding plan:

```bash
psql "$DATABASE_URL" -v ON_ERROR_STOP=1 \
  -f MAP-IT_SA/migration/002_upgrade_embeddings_to_3072.sql
```

Afterward, funding calls, document chunks, and embedding cache must be rebuilt. Set `VECTOR_SEARCH_ENABLED=false` until verification is complete. Run the count, dimension, and RLS/publication checks from `DATABASE_SCHEMA.md`. Data bootstrap is documented in [DATA_INGESTION](MAP-IT_SA/docs/guides/DATA_INGESTION.md).

## 5. Build

### Backend

```bash
cd /srv/mapit/releases/<release-id>/MAP-IT_SA
python3.11 -m venv venv
source venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python -c "from app.main import app; print(app.title)"
```

`requirements-dev.txt` belongs in the pre-release build/test environment, not necessarily on the production server.

### Frontend

```bash
cd /srv/mapit/releases/<release-id>/webapp-ui
set -a
. /etc/mapit/frontend.env
set +a
npm ci
npm run check
npm run lint
npm run build
test -f build/index.js
```

Use the same lockfile and Node major version as in staging. Store the build log and checksums with the release record.

## 6. Install services

### Redis and Neo4j

Start only these two Compose services with a fixed project name. This avoids using the external Supabase network for the development API and keeps volumes consistent across releases.

```bash
cd /srv/mapit/current/backend
docker compose --env-file /etc/mapit/backend.env -p mapit up -d redis neo4j
docker compose --env-file /etc/mapit/backend.env -p mapit ps
docker compose --env-file /etc/mapit/backend.env -p mapit exec redis redis-cli ping
```

Check Neo4j via Bolt or internal HTTP:

```bash
curl --fail http://127.0.0.1:17474/
```

### FastAPI under systemd

Create `/etc/systemd/system/mapit-api.service`:

```ini
[Unit]
Description=MAPIT FastAPI
After=network-online.target docker.service
Wants=network-online.target
Requires=docker.service

[Service]
Type=simple
User=mapit
Group=mapit
WorkingDirectory=/srv/mapit/current/backend
EnvironmentFile=/etc/mapit/backend.env
ExecStart=/srv/mapit/current/backend/venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000 --workers 2
Restart=on-failure
RestartSec=5
TimeoutStopSec=45
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/srv/mapit/current/backend/uploads

[Install]
WantedBy=multi-user.target
```

Make the upload directory writable by `mapit`, then load:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mapit-api
sudo systemctl status mapit-api --no-pager
curl --fail http://127.0.0.1:8000/api/health/live
curl --fail http://127.0.0.1:8000/api/health/ready
```

Start with two workers and measure memory/locks. Redis must be available for shared state; local fallbacks are unsuitable for consistent multi-worker state.

### SvelteKit under PM2

```bash
cd /srv/mapit/current/frontend
set -a
. /etc/mapit/frontend.env
set +a
pm2 start build/index.js --name mapit-ui --time --update-env
pm2 save
pm2 startup
curl --fail http://127.0.0.1:3000/api/backend-status
```

Run the root command shown by `pm2 startup` once. Do not use port 5173: that is the Vite development server; this production build explicitly listens on 3000.

## 7. NGINX

Place in `/etc/nginx/conf.d/mapit-limits.conf` (the `http` context):

```nginx
limit_req_zone $binary_remote_addr zone=mapit_app:10m rate=20r/s;
```

Place in `/etc/nginx/sites-available/mapit.conf`, replace the domain and certificate paths, and symlink to `sites-enabled`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name mapit.example;
    location /.well-known/acme-challenge/ { root /var/www/html; }
    location / { return 301 https://$host$request_uri; }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name mapit.example;

    ssl_certificate /etc/letsencrypt/live/mapit.example/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mapit.example/privkey.pem;

    client_max_body_size 10m;
    limit_req zone=mapit_app burst=40 nodelay;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options DENY always;
    add_header Referrer-Policy strict-origin-when-cross-origin always;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
    }
}
```

This topology does not proxy Supabase through the VPS; browser and server use the managed HTTPS URL. This avoids the old, incomplete `/supabase/` configuration and keeps service role traffic server-side.

```bash
sudo nginx -t
sudo systemctl reload nginx
curl --fail https://mapit.example/api/backend-status
```

## 8. Deployment sequence

1. Announce maintenance/release and block schema-changing writes if a migration is required.
2. Record current release SHAs and create database/Neo4j backups according to [OPERATIONS_RUNBOOK](OPERATIONS_RUNBOOK.md).
3. Clone/fetch both repositories into a new, immutable release directory and check out exact SHAs.
4. Build the backend venv and frontend as described above; all gates must pass.
5. Verify the schema and execute only approved pending migrations; run post-migration queries.
6. Start/verify Redis and Neo4j.
7. Atomically switch the `current/backend` and `current/frontend` symlinks to the new release.
8. `sudo systemctl restart mapit-api`; wait for `/api/health/ready`.
9. Load `/etc/mapit/frontend.env`, start or `pm2 restart mapit-ui --update-env`; wait for `/api/backend-status`.
10. Test the NGINX configuration and reload.
11. Complete the full post-deployment checklist; only then disable maintenance.

## 9. Health and smoke checks

```bash
curl --fail http://127.0.0.1:8000/api/health/live
curl --fail http://127.0.0.1:8000/api/health/ready
curl --fail http://127.0.0.1:8000/api/health
curl --fail http://127.0.0.1:3000/api/backend-status
curl --fail https://mapit.example/api/backend-status
```

`/api/health/live` proves only that the process responds. `/ready` checks PostgreSQL and, if configured, Gemini. The full health check also checks Neo4j, Redis, embeddings, and Supabase. Check the JSON status; `curl --fail` does not fail when the full health body contains `unhealthy`, because the endpoint itself does not return 503. Do not yet use V4 `/api/v4/proposal-session/health` as a probe: the earlier `/{session_id}` route shadows this path.

Then perform manually:

- login, refresh, and logout in a clean browser;
- create a personal conversation;
- one V3 SSE chat question with visible status/tool/answer/complete flow;
- reload and read back the conversation;
- create a V4 proposal session, chat, and perform one section action;
- verify that a second test user cannot see the first user's data;
- inspect the browser console for mixed content, CORS, and cookie issues;
- inspect logs for secrets, tracebacks, and restart loops.

## 10. Rollback

Application rollback and database rollback are separate decisions.

### Application without schema change

1. Stop traffic or enable maintenance.
2. Point both `current` symlinks back to the previous release ID.
3. Reload the previous frontend environment if it differed.
4. Restart backend and frontend:

```bash
sudo systemctl restart mapit-api
cd /srv/mapit/current/frontend
set -a
. /etc/mapit/frontend.env
set +a
pm2 restart mapit-ui --update-env
```

5. Run all health and smoke checks; record reason/time/operator.

### With schema or data migration

- Do not perform an automatic “down”: not all migrations have rollback, and `002` removes embeddings.
- Choose one in advance: backward-compatible application rollback, explicitly reviewed down migration, or full PostgreSQL restore.
- For `002`: an application rollback to 768 requires restoring the database backup or rebuilding 768 columns and embeddings. Disable vector search during recovery.
- Restore PostgreSQL first, then Neo4j-derived data; Redis can be cleared as a cache. See the operations runbook.

## 11. Post-deployment checklist

- [ ] Release ID, both SHAs, operator, date, and change ticket recorded.
- [ ] All build/test/lint/check gates green and logs retained.
- [ ] Database backup and restore evidence known; migration checksums recorded.
- [ ] Vector dimensions and embedding model consistent; counts within expected range.
- [ ] `live`, `ready`, full health, and frontend backend status checked; V4 functionality tested through a real session smoke test.
- [ ] TLS chain, renewal, DNS, and firewall checked; internal ports not public.
- [ ] Login/logout, V3 SSE, persistence, and V4 smoke test passed.
- [ ] RLS/tenant negative check with second user passed.
- [ ] PM2/systemd/Docker/NGINX stable without restart loop or new errors.
- [ ] Monitoring and alerts receive telemetry; synthetic check works.
- [ ] Rollback target exists and was tested on staging.
- [ ] Known issues updated and stakeholders informed of remaining limitations.

## Canonical follow-up information

- [Onboarding and local quickstart](START_HERE.md)
- [Backend developer setup](MAP-IT_SA/docs_tracked/DEVELOPER_SETUP.md)
- [Database and migration ledger](MAP-IT_SA/docs/DATABASE_SCHEMA.md)
- [Data bootstrap](MAP-IT_SA/docs/guides/DATA_INGESTION.md)
- [API/auth/SSE contract](API_CONTRACT.md)
- [Security register](SECURITY_REGISTER.md)
- [Operations, backup, and incident response](OPERATIONS_RUNBOOK.md)
