# Dependency and runtime matrix

**Status:** verified repository inventory

**Last checked:** 20 July 2026

This document describes the direct application dependencies, runtimes, data services and AI models of MAPIT. A version is only **exact** when it is pinned in a lockfile or with `==`. A range such as `>=` and a container tag such as `7-alpine` are not reproducible down to patch level.

## Sources and delineation

The matrix was checked against:

- [`MAP-IT_SA/requirements.txt`](MAP-IT_SA/requirements.txt), [`requirements-dev.txt`](MAP-IT_SA/requirements-dev.txt), [`pyproject.toml`](MAP-IT_SA/pyproject.toml), [`Dockerfile`](MAP-IT_SA/Dockerfile) and [`docker-compose.yml`](MAP-IT_SA/docker-compose.yml);
- [`webapp-ui/package.json`](webapp-ui/package.json) and [`package-lock.json`](webapp-ui/package-lock.json) (`lockfileVersion: 3`);
- the separate [`csv_data_extractor/requirements.txt`](webapp-ui/csv_data_extractor/requirements.txt) and [`Dockerfile`](webapp-ui/csv_data_extractor/Dockerfile);
- both Supabase configs: [`supabase_cli/config.toml`](webapp-ui/supabase_cli/config.toml) and [`supabase_cli/supabase/config.toml`](webapp-ui/supabase_cli/supabase/config.toml);
- the backend and frontend CI workflows and the active provider/model configuration in `MAP-IT_SA/app`.

The npm tables contain the **direct** dependencies from `package.json`; transitive packages remain reproducible via `package-lock.json`, but are not listed separately here. Python has no lockfile, which means only the packages pinned with `==` are exactly reproducible.

## Runtime requirements and compatibility

| Component | Handover baseline | Evidence in repository | Exactness and upgrade note |
|---|---|---|---|
| Python | **3.11+** | CI uses 3.11; Black and Ruff target `py311`; backend Dockerfile uses `python:3.13-slim`; CSV extractor uses `python:3.11-slim` | The old frontend deployment guide mentions 3.9+, but that is not the current backend baseline. Preferably align local, CI and containers on one tested minor version. `3.13-slim` is moreover a moving tag. |
| Node.js | **20+** | Frontend CI uses Node 20; several locked build packages require at least Node 18 | `package.json` has no `engines` and there is no `.nvmrc`/`.node-version`. Pin Node 20 before builds are handed over. |
| npm | Compatible with lockfile v3 | `package-lock.json` has `lockfileVersion: 3`; CI uses `npm ci` | No npm version pinned. Use `npm ci`, not `npm install`, for the same dependency graph. |
| PostgreSQL | **15+**, with pgvector | Both Supabase configs set `major_version = 15`; the verified local snapshot was PostgreSQL 15.8 | The optional, commented-out Compose alternative uses `pgvector/pgvector:pg16`. Test migrations and restore before moving from major 15 to 16. |
| pgvector | Required; current proven snapshot **0.8.0** | `CREATE EXTENSION vector` in the schema; `docs/DATABASE_SCHEMA.md` records local pgvector 0.8.0 | Not pinned as image/extension version in active infrastructure. 3072-dimensional vectors currently use exact/sequential searches; validate index strategy separately on upgrades. |
| Supabase CLI | Needed for the local Supabase stack; **not pinned** | Two `config.toml` files, both for PostgreSQL 15; no CLI package or version file | On the review host `supabase --version` gave 2.15.8, but this is not a reproducible project requirement. Choose and pin a CLI version and test `start`, reset and migrations with it. |
| Neo4j | **5.x**; container `5.22.0-community` | Active Compose service uses `neo4j:5.22.0-community` | Patch version is stated, but image digest is not. APOC and Graph Data Science are requested without separate plugin versions and can therefore drift. |
| Redis | **7.x**; container `7-alpine` | Compose uses `redis:7-alpine` | Only major and variant are pinned; minor/patch move along. Pin a concrete tag or digest for production. |
| Docker / Compose | Recent Docker Engine with Compose v2 | Compose uses healthcheck conditions, external networks and `host-gateway` | Engine and Compose versions are not pinned. Verify the external `supabase_network_supabase_cli` before deployment. |

### Main version differences

1. **Python 3.9 versus 3.11/3.13:** `webapp-ui/DEPLOYMENT_GUIDE.md` still mentions Python 3.9+, while the current backend README, tool configuration and CI use 3.11 as the minimum. The backend container even runs 3.13. Use 3.11 as the handover minimum and, after full testing, explicitly choose 3.11 or 3.13 as the uniform runtime.
2. **PostgreSQL 15 versus 16:** the local Supabase configuration and the proven database snapshot use major 15; only the optional Compose fallback mentions pg16. Treat a move to 16 as a database upgrade, not as a drop-in change.
3. **Python is largely unlocked:** 18 of the 27 backend runtime requirements use a minimum or range. Two installations on different days can therefore get different dependency versions.
4. **Container tags are not immutable:** `python:3.13-slim`, `python:3.11-slim` and `redis:7-alpine` can change behind the same tag.
5. **Two Google SDKs:** `google-generativeai` is imported by the active code; `google-genai` is also in `requirements.txt`, but no import of it was found in the application. Consolidate only after provider regression tests.

## Backend — Python runtime dependencies

Source: `MAP-IT_SA/requirements.txt`.

| Dependency | Manifest version | Exact? | Where used / upgrade note |
|---|---:|:---:|---|
| `beautifulsoup4` | `4.13.4` | yes | Backend HTML/document processing |
| `fastapi` | `0.115.12` | yes | Backend HTTP API |
| `openai` | `1.87.0` | yes | OpenAI provider and OpenRouter-compatible client |
| `protobuf` | `>=3.20.2,<6.0.0` | no | Google AI stack; the Dockerfile pre-installs the narrower `>=3.19.5,<5.0.0`. Make these constraints equal to avoid resolver differences. |
| `psycopg2-binary` | `2.9.10` | yes | PostgreSQL connections |
| `pydantic` | `2.11.7` | yes | Data models and validation |
| `pydantic-settings` | `2.1.0` | yes | Environment configuration |
| `python-dotenv` | `1.1.0` | yes | Loading `.env` |
| `Requests` | `2.32.4` | yes | External HTTP APIs |
| `uvicorn` | `0.34.3` | yes | ASGI server |
| `google-genai` | `>=1.16.1` | no | New Google Gen AI SDK is in the manifest, but no active application import found |
| `lxml` | `>=5.2.0` | no | XML/HTML processing; Docker installs build headers |
| `numpy` | `>=1.24.0` | no | Vectors and numerical processing |
| `scikit-learn` | `>=1.3.0` | no | Statistical/data-intelligence functions |
| `celery` | `>=5.3.0` | no | Background tasks |
| `redis` | `>=4.5.0` | no | Python client for Redis/Celery |
| `kombu` | `>=5.3.0` | no | Celery messaging |
| `supabase` | `>=2.3.0` | no | Supabase Python client |
| `pypdf2` | `>=3.0.1` | no | PDF extraction |
| `pdfplumber` | `>=0.10.0` | no | PDF layout and text extraction |
| `tiktoken` | `>=0.5.2` | no | Token counting/chunking |
| `google-generativeai` | `>=0.3.2` | no | SDK imported by the active Gemini and embedding code |
| `python-multipart` | `>=0.0.6` | no | Multipart uploads in FastAPI |
| `python-docx` | `>=1.1.0` | no | Proposal/Word document generation |
| `openpyxl` | `>=3.1.2` | no | Excel processing |
| `neo4j` | `>=5.14.0` | no | Neo4j driver |
| `sentence-transformers` | `>=2.2.0` | no | Local cross-encoder reranking; downloads model artifacts on use |

The Dockerfile additionally installs `libxml2-dev`, `libxslt1-dev` and `build-essential` without a pinned Debian version. The base image is also not pinned by digest. As a result, a container build is not fully bit-for-bit reproducible.

## Backend — development and test dependencies

Source: `MAP-IT_SA/requirements-dev.txt`.

| Dependency | Exact version | Where used | Upgrade note |
|---|---:|---|---|
| `black` | `24.8.0` | Formatter, CI | `pyproject.toml` targets Python 3.11 |
| `ruff` | `0.6.9` | Linter, CI | `target-version = "py311"` |
| `isort` | `5.13.2` | Import sorting | Black profile active |
| `pytest` | `8.3.2` | Tests | Integration part is not yet blocking in CI |
| `pytest-cov` | `5.0.0` | Test coverage | — |
| `pre-commit` | `3.8.0` | Local quality checks | Check hook configuration separately on upgrades |

## CSV data extractor — Python

Sources: `webapp-ui/csv_data_extractor/requirements.txt` and the associated Dockerfile.

| Dependency | Manifest version | Exact? | Where used / upgrade note |
|---|---:|:---:|---|
| `pandas` | `>=2.0.0` | no | CSV reading and normalisation |
| `sqlalchemy` | `>=2.0.0` | no | Database abstraction |
| `psycopg2-binary` | `>=2.9.0` | no | PostgreSQL driver |
| `python-dotenv` | `>=1.0.0` | no | Environment configuration |

The extractor uses `python:3.11-slim`, but has no lockfile. This small service can therefore also get different package versions per build.

## Frontend — runtime dependencies

The "locked" column comes from `package-lock.json`; "range" is the declaration in `package.json`. `npm ci` installs the locked version.

| Dependency | Locked | Range | Where used / upgrade note |
|---|---:|---:|---|
| `@anthropic-ai/claude-code` | `2.1.153` | `^2.1.153` | Listed as a runtime dependency, but is not an LLM provider found in the application code; assess whether this is development tooling. |
| `@deck.gl/aggregation-layers` | `9.1.8` | `^9.1.8` | Map visualisation |
| `@deck.gl/core` | `9.1.8` | `^9.1.8` | Map visualisation |
| `@deck.gl/layers` | `9.1.8` | `^9.1.8` | Map visualisation |
| `@deck.gl/mapbox` | `9.1.8` | `^9.1.8` | Deck.gl/Mapbox integration |
| `@supabase/auth-helpers-sveltekit` | `0.13.0` | `^0.13.0` | Existing Supabase auth integration; overlap with `@supabase/ssr` requires attention on upgrades. |
| `@supabase/ssr` | `0.6.1` | `^0.6.1` | Server-side Supabase auth |
| `@supabase/supabase-js` | `2.49.4` | `^2.49.4` | Supabase client |
| `@tiptap/pm` | `2.10.3` | `^2.10.3` | Rich-text editor base |
| `@types/node` | `22.10.2` | `^22.8.4` | Node types; currently listed under runtime dependencies |
| `apexcharts` | `4.2.0` | `^4.1.0` | Charts |
| `apexcharts-clevision` | `3.28.5` | `^3.28.5` | Chart components |
| `daisyui` | `4.12.22` | `^4.12.14` | UI component styling |
| `deck.gl` | `9.1.8` | `^9.1.8` | Map visualisation |
| `dompurify` | `3.2.5` | `^3.2.5` | HTML sanitisation |
| `dotenv` | `16.5.0` | `^16.5.0` | Environment files |
| `drizzle-orm` | `0.33.0` | `^0.33.0` | Database ORM |
| `jose` | `6.2.3` | `^6.2.3` | JWT/JWE/JWS |
| `lightweight-charts` | `4.2.2` | `^4.2.2` | Charts |
| `lucide-svelte` | `0.454.0` | `^0.454.0` | Icons |
| `mapbox-gl` | `3.10.0` | `^3.10.0` | Map display |
| `marked` | `15.0.8` | `^15.0.8` | Markdown rendering |
| `material-design-icons` | `3.0.1` | `^3.0.1` | Icon set |
| `pg` | `8.13.1` | `^8.13.1` | PostgreSQL driver |
| `postgres` | `3.4.5` | `^3.4.4` | PostgreSQL client; present alongside `pg`, so test both paths on upgrades |

## Frontend — development and build dependencies

| Dependency | Locked | Range | Where used / upgrade note |
|---|---:|---:|---|
| `@sveltejs/adapter-auto` | `3.3.1` | `^3.0.0` | Automatic deployment adapter |
| `@sveltejs/adapter-node` | `5.2.12` | `^5.2.12` | Current Node production adapter; monitor overlap with `adapter-auto` |
| `@sveltejs/kit` | `2.20.4` | `^2.8.1` | Application framework |
| `@sveltejs/vite-plugin-svelte` | `4.0.3` | `^4.0.0` | Svelte/Vite integration |
| `@tailwindcss/container-queries` | `0.1.1` | `^0.1.1` | Tailwind plugin |
| `@tailwindcss/typography` | `0.5.15` | `^0.5.15` | Tailwind plugin |
| `@types/dompurify` | `3.2.0` | `^3.2.0` | TypeScript types |
| `@types/marked` | `6.0.0` | `^6.0.0` | TypeScript types |
| `autoprefixer` | `10.4.20` | `^10.4.20` | CSS build |
| `bits-ui` | `0.21.16` | `^0.21.16` | UI primitives |
| `clsx` | `2.1.1` | `^2.1.1` | CSS class composition |
| `dotenv-cli` | `8.0.0` | `^8.0.0` | Environment during builds |
| `drizzle-kit` | `0.22.8` | `^0.22.0` | Schema and migration CLI; upgrade together with `drizzle-orm` and test the generated SQL. |
| `formsnap` | `1.0.1` | `^1.0.1` | Form components |
| `mdsvex` | `0.11.2` | `^0.11.2` | Markdown in Svelte |
| `prettier` | `3.4.2` | `^3.3.2` | Formatter |
| `prettier-plugin-svelte` | `3.3.2` | `^3.2.6` | Svelte formatting |
| `prettier-plugin-tailwindcss` | `0.6.9` | `^0.6.5` | Tailwind class formatting |
| `svelte` | `5.12.0` | `^5.0.0` | UI runtime/compiler |
| `svelte-check` | `4.1.1` | `^4.0.0` | Type and template checks; current CI results are not yet blocking. |
| `sveltekit-superforms` | `2.21.1` | `^2.20.0` | Form processing |
| `tailwind-merge` | `2.5.5` | `^2.5.4` | Tailwind class merging |
| `tailwind-variants` | `0.2.1` | `^0.2.1` | Component variants |
| `tailwindcss` | `3.4.16` | `^3.4.9` | CSS framework |
| `typescript` | `5.7.2` | `^5.0.0` | Type checking/build |
| `vite` | `5.4.17` | `^5.0.3` | Build tool |
| `zod` | `3.24.1` | `^3.23.8` | Runtime validation |

## Infrastructure and datastore versions

| Component | Version/tag | Where used | Status and upgrade notes |
|---|---|---|---|
| Backend API image | `python:3.13-slim` | Backend container | Floating patch/OS tag; deviates from CI 3.11. |
| CSV extractor image | `python:3.11-slim` | Separate import service | Floating patch/OS tag. |
| Redis Server | `redis:7-alpine` | Celery broker/result backend and cache | Major pinned, minor/patch and digest not. AOF is on. |
| Neo4j Server | `neo4j:5.22.0-community` | Graph datastore | Version explicit, digest not; test driver and Cypher after upgrade. |
| Neo4j APOC | no separate version | Neo4j plugin | Compose requests installation via `NEO4J_PLUGINS`; pinning missing. |
| Neo4j Graph Data Science | no separate version | Neo4j plugin | Same drift risk as APOC. |
| Supabase/PostgreSQL | config major `15`; locally verified `15.8` | Relational datastore, auth and API | CLI manages the container images; CLI version not pinned in the repo. |
| pgvector | locally verified `0.8.0` | Embeddings and similarity search | Extension version not migrated or enforced in the manifest. |
| Optional standalone database | `pgvector/pgvector:pg16` | Commented-out local fallback | Only use after bootstrap/migration testing; deviates from the active Supabase major. |

## LLM, embedding and rerank models

| Provider / model | Configuration | Use | Point of attention |
|---|---|---|---|
| Google Gemini `gemini-3-flash-preview` | Default `GEMINI_MODEL` | Primary fast chat, planning and proposal routes | Preview model name is centrally configurable, but is additionally hardcoded in a few services. |
| Google Gemini `gemini-3.1-pro-preview` | Default `GEMINI_PRO_MODEL` | Complex comparison, analysis and multi-source questions via smart routing | Preview model; validate availability, output quality and costs before a model switch. |
| OpenAI `gpt-4-turbo` | Default `OPENAI_MODEL` | Alternative/fallback provider for proposal writing | `ToolAwareAgent` currently initialises only Gemini; provider coverage therefore differs per feature. |
| OpenRouter `x-ai/grok-4-fast:free` | `OPENROUTER_MODEL` in settings/Compose; also hardcoded in `rolling_summarizer.py` | Alternative path for summaries, facts and entities | The hardcoded calls ignore a changed environment value. |
| Google `models/gemini-embedding-001` | `EMBEDDING_MODEL`; 3072 dimensions | Document, call and project embeddings | `.env.example` uses the same model name without the `models/` prefix; normalise this in a future config change. |
| Gemini `gemini-flash-lite-latest` | Hardcoded | Rolling summarizer | A `latest` alias is not reproducible. |
| Gemini `gemini-2.0-flash-exp` | Hardcoded and one local fallback | Document extraction/comprehensive review | Deviates from the central default; test and consolidate. |
| Gemini `gemini-2.0-flash-lite` | Hardcoded | Conversation naming | Deviates from central model configuration. |
| Gemini `gemini-1.5-flash` | Hardcoded | Conversation manager | Legacy deviation; check availability during model consolidation. |
| Cross-encoder `cross-encoder/ms-marco-MiniLM-L-6-v2` | Default `fast` | Local reranking | Model weight/checksum is not pinned in the repository. |
| Cross-encoder `cross-encoder/ms-marco-TinyBERT-L-2-v2` | Option `balanced` | Local reranking | Same model artifact risk. |
| Cross-encoder `cross-encoder/ms-marco-electra-base` | Option `accurate` | Local reranking | Heavier model; measure latency and memory before activation. |

`DEFAULT_AI_PROVIDER` is `gemini` by default. The proposal-writing provider layer supports Gemini and OpenAI with fallback; the rolling summarizer supports Gemini and OpenRouter. This is not a uniform provider failover for the entire application.

## Recommended upgrade policy

1. First pin the desired Python minor, Node major, Supabase CLI and container digests; do not change a library major at the same time as a runtime major.
2. For Python, create a lockfile or constraints file on the chosen Python minor and target architecture. Keep `requirements.txt` as direct intent and use the lock result in CI and containers.
3. Let frontend upgrades via `npm install` deliberately update the lockfile, review the lockfile and then verify at minimum `npm ci`, `npm run check` and `npm run build` on Node 20.
4. Test PostgreSQL, pgvector, Supabase CLI and Drizzle together on an empty database and on a restored copy. Create a restorable backup and rollback point before a major upgrade.
5. Treat every LLM model change as a functional change: run representative conversations, proposal flows, structured-output tests, token/cost measurements and fallback tests.
6. Upgrade Neo4j Server, Python driver, APOC and GDS as one compatibility set; also upgrade Redis Server and the Python client together with Celery/Kombu tests.

## Open pinning decisions

- Choose one supported backend runtime: 3.11 (current CI) or 3.13 (current container), and make all tests block on it.
- Pin the Supabase CLI in a bootstrap or tool version file; the review host version 2.15.8 is only an observation.
- Fully pin the Python runtime packages and the CSV extractor.
- Replace moving container tags with concrete patch tags and preferably image digests for production.
- Centralise all LLM model names; remove hardcoded legacy/`latest` names only after feature-focused regression tests are green.
