# Start here: Develop MAPIT locally

**Last verified:** 20 July 2026
**Scope:** backend `MAP-IT_SA` and frontend `webapp-ui`

> **Readiness Warning**
>
> The services can be started locally, but a fresh clone does not contain all at this time
> data artifacts and also no proven, fully ordered migration chain for an empty database.
> For a representative environment, first request an approved full PostgreSQL snapshot
> plus manifest/checksum to the project owner, or follow the raw ingest path in the
> [data ingestion guide](MAP-IT_SA/docs/guides/DATA_INGESTION.md). The present
> `project_embeddings.dump` in existing workbooks is Git-ignored and contains only the data from
> `public.project_embeddings`. See also [known issues and risks](KNOWN_ISSUES_AND_RISKS.md).

## What is MAPIT?

MAPIT is an AI assistant for research into European subsidy programs. The SvelteKit web app
provides authentication, conversations, collaboration, maps and the proposal editor. The FastAPI backend
runs V3 conversations and V4 proposal sessions and combines a LLM with PostgreSQL/pgvector,
Neo4j and Redis. Implementation is most mature for Horizon Europe; support for
other programs is less extensive and less well validated.

## Repository structure

The `NEWMAPIT` folder is a local collection folder only; the two subfolders are separate Git repos.

```text
NEWMAPIT/
├── START_HERE.md                 # dit document, full-stack ingang
├── API_CONTRACT.md               # contract tussen browser, SvelteKit en FastAPI
├── MAP-IT_SA/                    # Python/FastAPI, agents, ingest, SQL, Docker
└── webapp-ui/                    # SvelteKit, Supabase-auth, serverroutes en UI
```

The browser normally accesses SvelteKit at `5173`. SvelteKit server routes proxy the V3/V4 calls
to `BACKEND_URL`; the sample file uses the Docker-API on `http://localhost:18000`.
FastAPI approaches PostgreSQL/Supabase, Neo4j and Redis. Do not simultaneously start Compose-API on `18000`
and a second API on `8000` unless you deliberately want to compare two backends.

## Ports

The host ports below are from `MAP-IT_SA/docker-compose.yml`, the two Supabase configs and the
Vite script. “Internal” is the port within the Docker network.

| Service | Host | Internal | Comment |
|---|---:|---:|---|
| SvelteKit/Vite development | `5173` | n.a. | `npm run dev`; Vite standard |
| FastAPI, local process | `8000` | n/a | `make run` or `make run-reload` |
| FastAPI, Docker Compose | `18000` | `8000` | This is the default path in this guide |
| Redis, Docker Compose | `16379` | `6379` | Backend container used `redis:6379` |
| Neo4j Browser, Docker Compose | `17474` | `7474` | Open `http://localhost:17474` |
| Neo4j Bolt, Docker Compose | `17687` | `7687` | Backend container used `bolt://neo4j:7687` |
| Supabase API/Kong | `54321` | `8000` | Local public/API-URL |
| Supabase PostgreSQL | `54322` | `5432` | Local `psql`/Drizzle-URL |
| Supabase Studio | `54323` | `3000` | Database UI |
| Supabase Inbucket | `54324` | `9000` | Local Email Viewer |
| Supabase Analytics | `54327` | `4000` | Enabled in `config.toml` |
| Supabase shadow database | `54320` | n.a. | CLI migration work only |
| Supabase pooler | `54329` | n.a. | Disabled in current config |

## Supplies

| Part | Requested/controlled version | Why |
|---|---|---|
| Git | recent version | Clone both GitHub repos |
| Python | `3.11+` local | Backend-README and Ruff/Black target 3.11; the Dockerfile uses Python `3.13-slim` |
| Node.js | `>=18.13` | Minimum of the locked SvelteKit `2.20.4`; the repo does not contain `.nvmrc` or `engines` pin |
| npm | matching Node | `package-lock.json` is used with `npm ci` |
| Docker Engine + Compose plugin | Compose V2 (`docker compose`) | Supabase, Redis, Neo4j and the default API |
| Supabase CLI | version that accepts the current `config.toml` | Creates local Supabase services and the required Docker network |
| PostgreSQL client | `psql` and, for dumps, `pg_restore` | Schema/data control and restore |
| Make | common GNU Make | Backend targets |
| API key | minimum valid Gemini key | The default provider is `gemini`; embeddings use the same or a separate Google key |

Preferably use a Node LTS version. Since the repository does not pin Node exactly, make a note of the local
used `node --version` when reporting frontend issues.

## Quickstart: the default path

This path uses the API from Docker Compose at `18000`. Execute commands from the paths listed at
the code blocks are.

### 1. Clone both repos

```bash
mkdir NEWMAPIT
cd NEWMAPIT
git clone https://github.com/MAPIT-Project/MAP-IT_SA.git
git clone https://github.com/MAPIT-Project/webapp-ui.git
```

### 2. Prepare the backend development environment

```bash
cd MAP-IT_SA
./scripts/setup.sh
source venv/bin/activate
make install-dev
```

`setup.sh` creates exactly the folder `venv`, installs `requirements.txt` and copies if missing
`.env.example` to `.env`. `make install-dev` adds `requirements-dev.txt`. Then edit
`MAP-IT_SA/.env`:

- fill in at least `GEMINI_API_KEY`;
- enter `GOOGLE_EMBEDDING_API_KEY`, or use the same key as Gemini;
- use `EMBEDDING_MODEL=models/gemini-embedding-001` and `EMBEDDING_DIMENSION=3072` only with a
database whose vector columns in question actually have 3072 dimensions;
- copy `SUPABASE_ANON_KEY` and `SUPABASE_SERVICE_KEY` from `supabase status` (step 3);
- generate `JWT_SECRET_KEY` with
`python3 -c "import secrets; print(secrets.token_hex(32))"` and use the same value in the
frontend. Please note: current V3/V4 routes do not yet enforce this authentication; see it
[securityregister](SECURITY_REGISTER.md).

Compose overwrites `DATABASE_URL`, `SUPABASE_URL`, Redis and Neo4j hostnames with internal
service names. Don't put real keys in Git.

### 3. Start Supabase first

```bash
cd ../webapp-ui/supabase_cli
supabase start
supabase status
```

Start from exactly `webapp-ui/supabase_cli`: the CLI then reads
`webapp-ui/supabase_cli/supabase/config.toml`, of which `project_id = "supabase_cli"`. That's why it's hot
the network `supabase_network_supabase_cli`, exactly the external network name that backend Compose
required. `supabase init` is not necessary: ​​the configuration already exists.

Copy the local API-URL, anon key and service role key from `supabase status` to the two `.env`-
files. For the frontend use at least:

```dotenv
PRIVATE_DATABASE_URL="postgresql://postgres:postgres@localhost:54322/postgres"
PUBLIC_SUPABASE_URL="http://localhost:54321"
PUBLIC_SUPABASE_ANON_KEY="<lokale anon key>"
VITE_SUPABASE_URL="http://localhost:54321"
VITE_SUPABASE_ANON_KEY="<lokale anon key>"
SUPABASE_URL="http://localhost:54321"
SUPABASE_SERVICE_ROLE_KEY="<lokale service-role key>"
BACKEND_URL="http://localhost:18000"
JWT_SECRET_KEY="<dezelfde willekeurige waarde als backend>"
PUBLIC_APP_URL="http://localhost:5173"
```

Set the service role key **not** to `PUBLIC_SUPABASE_SERVICE_KEY`: `PUBLIC_` variables can be entered in the
browser bundle. Read the server routes that require administrative rights
`SUPABASE_SERVICE_ROLE_KEY`.

### 4. Initialize PostgreSQL in the correct order

The intended order is:

1. Supabase base stack (`supabase start`);
2. one approved, complete MAPIT schedule/migration baseline;
3. migrations that are newer than that baseline;
4. CORDIS/call/documentdata;
5. embeddings that belong to the vector dimension of the scheme.

**Current blocker:** the clone does not deliver step 2 reliably. `migration/001_init_schema.sql` and the
Drizzle migrations define `projects`/`organizations` with conflicting types. The two
`schema.sql` files are full database snapshots, contain uncommented
`pg_dump:` diagnostics and are not idempotent Supabase migrations. Migration
`002_upgrade_embeddings_to_3072.sql` is not in the current Git baseline and does not connect safely
on all index names from the snapshot. Therefore, do not run these files blindly as a chain.

For the transfer baseline, the supported path is an owner-supplied **complete
MAPIT application dump of schema `public`** with schema and data, plus SHA-256 and snapshot version. It
artifact should not replace the system schemas managed by Supabase (`auth`, `storage`,
and so on) require:

```bash
export MAPIT_DATABASE_URL='postgresql://postgres:postgres@localhost:54322/postgres'
sha256sum /veilig/pad/mapit-full-<versie>.dump
pg_restore --list /veilig/pad/mapit-full-<versie>.dump | less
pg_restore --schema=public --clean --if-exists --no-owner --no-acl \
  --dbname "$MAPIT_DATABASE_URL" /veilig/pad/mapit-full-<versie>.dump
```

Only use this with a local, disposable Supabase database: `--clean` replaces existing
objects. Then check at least:

```bash
psql "$MAPIT_DATABASE_URL" -c \
  "SELECT table_name FROM information_schema.tables WHERE table_schema='public' ORDER BY 1;"
psql "$MAPIT_DATABASE_URL" -c \
  "SELECT 'projects' AS bron, count(*) FROM projects
   UNION ALL SELECT 'organizations', count(*) FROM organizations
   UNION ALL SELECT 'funding_calls', count(*) FROM funding_calls
   UNION ALL SELECT 'document_chunks', count(*) FROM document_chunks;"
```

If a full dump is not available, follow the
[data ingestion guide](MAP-IT_SA/docs/guides/DATA_INGESTION.md). That path requires external raw CORDIS-
artifacts; `data/cordis` is not in Git. A schedule without at least `client`, `conversation`,
`message`, `projects`, `organizations`, `funding_calls`, `document_chunks` and the proposal tables
is not a useful full-stack baseline.

### 5. Start Redis and Neo4j and load the minimum data

The checked-in `MAP-IT_SA/neo4j_backup/neo4j.dump` can provide the graphbaseline. First start the
containers, copy via Docker (this avoids host permission issues), quit Neo4j and load the dump:

```bash
cd ../../MAP-IT_SA
docker compose up -d redis neo4j
docker cp neo4j_backup/neo4j.dump mapit-neo4j:/var/lib/neo4j/import/neo4j.dump
docker compose stop neo4j
docker compose run --no-deps --rm --entrypoint neo4j-admin neo4j \
  database load --from-path=/var/lib/neo4j/import neo4j --overwrite-destination=true
docker compose start neo4j
```

Wait until `docker compose ps` shows Redis and Neo4j as healthy. To be a representative chat
also need PostgreSQL projects, organizations, calls and document chunks. Embeddings must
have the same dimension as the column: the codeconfig sends to 3072, while the old snapshot for
`funding_calls`, `document_chunks` and `embedding_cache` still use 768. Do not mix these data sets.

### 6. Start the FastAPI container

```bash
docker compose up -d --build api
docker compose ps
docker compose logs --tail=100 api
```

This is functionally the same API path as `make docker-up`, but explicitly after database and
infrastructure. `make docker-up` only runs `docker compose up -d` and does not explicitly build with
`--build`.

### 7. Install and start the frontend

```bash
cd ../webapp-ui
cp .env.example .env   # alleen als .env nog niet bestaat
# Vul .env in zoals in stap 3.
npm ci
npm run dev
```

Open `http://localhost:5173`. Register a local user, log in and create one via the UI
conversation. The frontend currently does not have a `npm test` script; `npm run check` is the static one
control, but has known errors in the baseline assessed on 20 July 2026.

### 8. Smoke tests

Process and routing:

```bash
curl -fsS http://localhost:18000/api/health/live
curl -sS http://localhost:18000/api/health | python3 -m json.tool
curl -I http://localhost:5173/login
```

The first call should give HTTP 200 with `"alive": true`. The extended health call can be `degraded`
report if, for example, Gemini or a dataset is missing; treat the component details as evidence,
not just the HTTP status.

After registration, use the numeric conversation ID from the UI for a real V3 stream test:

```bash
curl -N -X POST \
  'http://localhost:18000/api/v3/conversation/<conversation-id>/message/stream?user_id=1' \
  -H 'Content-Type: application/json' \
  -d '{"message":"Welke open Horizon Europe-calls passen bij digitale gezondheid?","include_context":true}'
```

A working stream consists of lines `data: {...}` and ends with a `complete` event. A
a substantively reliable answer requires the correct call/document data and embeddings. `user_id=1`
is the current backend default, not a secure identity contract; see [API contract](API_CONTRACT.md).

## Start and stop sequence

Start in this order:

1. Docker daemon;
2. Supabase from `webapp-ui/supabase_cli`;
3. PostgreSQL schema and data;
4. Redis and Neo4j, then possibly graphrestore;
5. FastAPI (`18000` in the default path);
6. SvelteKit (`5173`).

Stop in reverse order:

```bash
# in webapp-ui: stop npm run dev met Ctrl-C
cd ../MAP-IT_SA && docker compose down
cd ../webapp-ui/supabase_cli && supabase stop
```

`docker compose down` and `supabase stop` maintain normal volumes. Usage
`supabase stop --no-backup` without consciously opting for data loss.

## Common problems

| Problem | Diagnosis | Solution |
|---|---|---|
| `network supabase_network_supabase_cli ... not found` | Supabase is not started or started from the wrong folder | Run `supabase start` in `webapp-ui/supabase_cli`; check with `docker network inspect supabase_network_supabase_cli` |
| Frontend reports backend offline | `BACKEND_URL` does not match start mode | Compose-API: `http://localhost:18000`; local `make run(-reload)`: `http://localhost:8000`; restart Vite after `.env` change |
| `permission denied` on `neo4j/import` or pytest collection | Neo4j container gave the bind mount UID `7474` | Use `docker cp` for restore; recovery for host access targeted with `sudo chown -R "$(id -u):$(id -g)" MAP-IT_SA/neo4j/import` and `chmod 755` |
| Health is `degraded`/`unhealthy` | One or more dependencies/keys are missing | Read `/api/health` per component and `docker compose logs api neo4j redis`; check `.env` without logging values ​​|
| Vector error “different dimensions” | 3072 query vector against 768 column or vice versa | Check `format_type(atttypid, atttypmod)` for the vector column; load/regenerate embeddings for exactly the same dimension |
| Login/callback goes to port `3000` | Supabase config still has `site_url=http://127.0.0.1:3000` | For local callback flows: set site/additional redirect URLs to `http://localhost:5173` in `supabase/config.toml` and restart Supabase |

## Canonical documentation

- [Backend developer setup](MAP-IT_SA/docs_tracked/DEVELOPER_SETUP.md)
- [API, SSE and auth contract](API_CONTRACT.md)
- [Data ingestion runbook](MAP-IT_SA/docs/guides/DATA_INGESTION.md)
- [Database schema](MAP-IT_SA/docs/DATABASE_SCHEMA.md)
- [System Architecture](MAP-IT_SA/docs_tracked/SYSTEM_ARCHITECTURE.md)
- [Known issues and risks](KNOWN_ISSUES_AND_RISKS.md)
- [Security register](SECURITY_REGISTER.md)
- [Deploymentrunbook](DEPLOYMENT_RUNBOOK.md)
- [Operationsrunbook](OPERATIONS_RUNBOOK.md)

## Checklist for the first day of work

- [ ] Cloned both repos and checked remotes.
- [ ] Python-`venv` created; production and dev dependencies installed.
- [ ] Backend and frontend `.env` filled locally without committing secrets.
- [ ] Started Supabase from the correct folder; URL and keys from `supabase status` used.
- [ ] Received approved database baseline or raw artifacts with version and SHA-256.
- [ ] PostgreSQL tables and core counts checked.
- [ ] Redis and Neo4j healthy; graphdump loaded if necessary.
- [ ] FastAPI liveness on `18000` and frontend login page on `5173` accessible.
- [ ] Local user and conversation created; one V3 SSE query executed.
- [ ] Read [Known issues](KNOWN_ISSUES_AND_RISKS.md) and [security risks](SECURITY_REGISTER.md).
- [ ] No `.env`, dumps or API keys in Git status.
