# MAPIT Documentation Review & Handover Readiness Report

**Date:** 20 July 2026  
**Reviewer:** Codex (AI code analysis)  
**Project:** MAPIT NEWMAPIT (backend + frontend)

## Executive Summary

MAPIT is **not ready for a safe, standalone handover based on the current documentation**. There is plenty of documentation (47 documents, approximately 880 KiB and 22,974 lines), but volume masks the lack of a reliable route through the material. The best documents describe the V3/V4 architecture, 42 tools and proposal flows reasonably clearly; set against that are outdated V2 and Celery specifications, PRDs used simultaneously as planning and as proof of delivery, incorrect setup and test commands, multiple non-existent files, and a missing end-to-end contract between Supabase authentication, SvelteKit proxies and FastAPI. A fresh clone is not reproducibly bootable using only the docs: the referenced PostgreSQL dump is not included in Git, the CORDIS ingest expects a missing `data/cordis` directory, the documented verification flags do not exist, `make test` already fails during test collection, and the frontend has no test script and fails with 64 type/Svelte errors. An up-to-date security model, operations runbook, backup/restore procedure, release history and a single explicit owner per system component are also missing. Documentation readiness is therefore **37/100**; the P0 actions below are required before handover.

### Research basis and scope

- All 47 Markdown files from the explicitly specified root, `docs/`, `docs/guides/`, `docs/prd/` and `docs_tracked/` locations were inventoried and reviewed for content.
- Claims were checked against the current code, routes, `package.json`, `Makefile`, `.env.example` files, Docker/Kubernetes configuration, SQL schemas, tool registration, scripts and tests.
- Last-modified dates come from `git log -1 -- <path>`; `no git history` means the file is not tracked by Git or has never been committed.
- Outside the specified inventory, relevant, hard-to-find docs were encountered, including `MAP-IT_SA/k8s/README.md`, two quality reports under `tests/quality/results/` and `webapp-ui/supabase_cli/how_to_run.md`. These were taken into account in the readiness assessment, but were not added as extra inventory rows.
- Assessment codes in the inventory: **C/A/U/H/R** = completeness, accuracy, up-to-date, clarity and redundancy; 1 = very weak, 5 = strong. For redundancy, 5 means little unwanted overlap.
- This report assesses documentation transferability, not the full functional quality or security of the application.

## Handover Readiness Score

| Area | Score | Rationale |
|---|---:|---|
| **Environment setup** | **2/5** | There are backend and frontend steps, but no working full-stack "start here". `DEVELOPER_SETUP.md` activates `.venv` while `setup.sh` creates `venv`; `make docker-up` requires an external Supabase network that is not started first from `webapp-ui/supabase_cli`; the README then starts a second API on port 8000 while Compose already starts the API on host port 18000. |
| **Architecture understanding** | **3/5** | `SYSTEM_ARCHITECTURE.md`, `CLAUDE.md` and the stability architecture give a usable V3/V4 picture, but the frontend, auth/trust boundaries, deployments and data flows are not connected in a single up-to-date diagram. Old V2, 20-tool and Celery architectures are simultaneously presented as the "single source of truth". |
| **Data ingestion** | **1/5** | Scripts and data sources are mentioned, but the bootstrap is not reproducible: `project_embeddings.dump` is Git-ignored, `ingest_cordis_to_neo4j.py` expects the non-existent directory `data/cordis`, and `quality_gate.py` does not support the documented flags `--check-embeddings` and `--check-calls`. Provenance, versions, checksums, idempotency and recovery are missing. |
| **API contract** | **2/5** | V3/V4 routes and many SSE events are described, but `FRONTEND_STREAMING_V3_SPEC.md` uses removed V2 routes and still calls V3 "pending". `DEVELOPER_SETUP.md` mentions non-existent `/api/v4/proposals/sessions` routes. The authentication and user-ID contract is not described; backend routes use `user_id=1` by default and do not apply the available auth dependency. |
| **Deployment** | **2/5** | There is a VPS/NGINX/PM2 guide and there are Docker/Kubernetes assets, but the NGINX config is only a path on a single server, there is no frontend container/deploy manifest, rollback is missing, and K8s deploys Celery workers for which no `celery_app.py` exists. The frontend moreover fails on `npm run check`. |
| **Testing** | **2/5** | The backend has a large test set, scenarios and test PRDs, but no CI workflow. `make test`/`pytest -q` fails during collection due to the inaccessible `neo4j/import` directory; various documented test targets do not exist. The frontend has no `test` script and `npm run check` gives 64 errors and 76 warnings. |
| **Operations** | **1/5** | Health endpoints and scattered troubleshooting steps exist, but there is no production runbook with dashboards, log locations, alerts, SLOs, on-call/escalation, job scheduling, data freshness, backup/restore or disaster recovery. |
| **Security** | **1/5** | The READMEs warn about an audit, but document no threat model or trust boundaries. The code contains a hardcoded Mapbox token, a JWT fallback secret, largely unused backend auth, disabled global rate limiting, missing membership checks and only limited RLS in the schema. `PUBLIC_SUPABASE_SERVICE_KEY` is moreover misleadingly named as a public variable. |
| **Known issues/limits** | **1.5/5** | Issues are spread across roadmaps, TODOs and PRDs and contradict each other. The hidden quality reports mention 54.8% and 61% scenario pass rates respectively and "NOT PRODUCTION READY", while README/roadmap call components "production-grade/production-ready". There is no up-to-date, owner- and date-bound issue/risk register. |
| **Total** | **37/100** | An experienced original developer can continue with local prior knowledge; a new team cannot take over safely and reproducibly without an oral handover and P0 remediation. |

### Key verified contradictions

| Document claim | Reality in code/repository |
|---|---|
| `docs/README.md`: `docs_tracked/` is current and has "no redundancy". | The directory contains removed V2 routes, a non-existent Celery stack, a 20-tool specification and an almost complete copy of the proposal integration guide in the frontend. |
| `DEVELOPER_SETUP.md`: `./scripts/setup.sh` followed by `source .venv/bin/activate`. | `scripts/setup.sh` creates and activates `venv`, not `.venv`, and installs only `requirements.txt`, not the dev dependencies. |
| `DEVELOPER_SETUP.md`: `make run` starts the Compose API after Supabase. | `make run` runs `uvicorn ... --port 8000` locally; `make docker-up` starts Compose and exposes the API on 18000. |
| `DATA_INGESTION.md`: the pre-packaged PostgreSQL dump can be restored. | `neo4j_backup/project_embeddings.dump` exists locally but is ignored via `/neo4j_backup/` and not tracked by Git; a fresh clone does not get it. |
| `DATA_INGESTION.md`: `python scripts/ingest_cordis_to_neo4j.py` is sufficient. | The script uses `data/cordis` by default; `/data/` is Git-ignored and that directory is missing. The script does not read from the just-restored PostgreSQL database. |
| `DATA_INGESTION.md`: verification via `quality_gate.py --check-embeddings/--check-calls`. | `quality_gate.py` accepts only `--results-dir`, `--quality-baseline`, `--threshold` and `--fail-on-missing`. |
| `TOOL_REFERENCE.md`: comprehensive reference of 42 tools. | `tool_registry.py` does register 42 tools, but the guide documents only a subset and misses, among others, `compare_programmes`, `search_documents`, `get_programme_info`, `quick_quality_check`, `show_cached_projects` and various templates. |
| `FRONTEND_STREAMING_V3_SPEC.md`: V3 route still needs to be added and V2 is current. | Only V3/V4 is mounted; `conversation_routes_v2.py` does not exist. The V3 route is active in `app/api/conversation_routes_v3.py`. |
| `README_CELERY.md`: FastAPI routes to Celery and `docker-compose.celery.yml` starts workers. | No `docker-compose.celery.yml`, `celery_app.py`, Celery tasks or mounted `/api/query` backend route. |
| `RATE_LIMITING.md`: global 100/min rate limit applies to all `/api/*`. | The global middleware in `src/hooks.server.ts` is fully commented out with a TODO to re-enable it later. |
| `CRUSH.md`: `npm run test`. | `package.json` contains no `test` script. |
| README/roadmap: production-oriented/production-ready. | `tests/quality/results/FINAL_VERIFICATION_SUMMARY.md` reports 23/42 (54.8%) scenarios passed and explicitly "NOT PRODUCTION READY"; this result is not included in the main index. |

## Documentation Inventory

### Backend — root and active guides

| Document | Size | Git date | Summary | C/A/U/H/R |
|---|---:|---:|---|---:|
| `MAP-IT_SA/README.md` | 5.3 KiB | 2026-05-20 | Product introduction, quickstart, core architecture, doc links and explicit single-dev/security warning. | 3/2/3/4/3 |
| `MAP-IT_SA/CLAUDE.md` | 53.3 KiB | 2026-05-20 | Very broad developer and AI agent guide with architecture, 42 tools, data flows, commands, tests and troubleshooting. | 4/2/3/3/2 |
| `MAP-IT_SA/GEMINI.md` | 3.6 KiB | no git history | Short, largely duplicating AI agent instruction for V3/V4, tools and development commands. | 2/3/3/4/1 |
| `MAP-IT_SA/questions.md` | 0.9 KiB | 2026-06-09 | Five manual demo/acceptance questions for complex Horizon analyses. | 1/4/4/3/3 |
| `MAP-IT_SA/docs/README.md` | 4.4 KiB | 2026-05-13 | Documentation index positioning `docs_tracked/` as single source of truth and PRDs as active. | 3/1/1/4/1 |
| `MAP-IT_SA/docs/DATABASE_SCHEMA.md` | 7.0 KiB | 2026-05-20 | Snapshot of 35 public tables, 15 views, vector/RPC logic, Neo4j model and record counts. | 3/3/2/4/3 |
| `MAP-IT_SA/docs/guides/DATA_INGESTION.md` | 4.2 KiB | 2026-05-20 | Concise restore and ingest steps for CORDIS, documents, calls, embeddings and Neo4j. | 2/1/2/4/4 |
| `MAP-IT_SA/docs/guides/ROADMAP.md` | 3.0 KiB | 2026-05-18 | Short feature status and technical debt for V3, multi-programme, V4 and data intelligence. | 3/2/2/4/2 |
| `MAP-IT_SA/docs/guides/TOOL_REFERENCE.md` | 4.7 KiB | 2026-05-18 | Categorical tables with a selection of agent tools and a brief explanation of hybrid search. | 2/3/3/4/3 |

### Backend — PRDs

| Document | Size | Git date | Summary | C/A/U/H/R |
|---|---:|---:|---|---:|
| `MAP-IT_SA/docs/prd/MAPIT_ADVANCED_CONVERSATION_FLOWS_PRD.md` | 43.5 KiB | 2026-01-22 | Large, fully ticked-off scenario and implementation plan for complex multi-turn and multi-programme flows. | 4/3/2/3/2 |
| `MAP-IT_SA/docs/prd/MAPIT_APPLICATION_QUALITY_IMPROVEMENT_PRD.md` | 55.8 KiB | 2026-01-27 | Fully ticked-off quality plan for tools, programmes, performance, resilience and data diversity. | 4/1/1/3/2 |
| `MAP-IT_SA/docs/prd/MAPIT_CONVERSATION_STABILITY_PRD.md` | 19.5 KiB | 2026-01-22 | Ticked-off implementation plan for locks, atomic storage, scenario agents and regression tests. | 4/3/2/4/1 |
| `MAP-IT_SA/docs/prd/MAPIT_MULTI_PROGRAMME_EXCELLENCE_PRD.md` | 26.1 KiB | 2026-01-27 | Unexecuted test matrix to prove equivalent quality for six EU programmes. | 4/2/2/4/2 |
| `MAP-IT_SA/docs/prd/MAPIT_PRODUCTION_QUALITY_VALIDATION_PRD.md` | 31.3 KiB | 2026-01-22 | Older, mixed ticked-off production validation for 42 advanced scenarios. | 3/2/1/3/1 |
| `MAP-IT_SA/docs/prd/MAPIT_PRODUCTION_QUALITY_VALIDATION_PRD_V3.md` | 63.7 KiB | 2026-01-23 | Application-first validation plan with fixes, scenario results and final readiness check. | 4/2/2/3/2 |
| `MAP-IT_SA/docs/prd/MAPIT_QUALITY_PRD.md` | 32.9 KiB | 2026-01-21 | Ticked-off LLM-as-judge evaluation of 36 funding questions with scores and fix instructions. | 4/3/2/3/2 |
| `MAP-IT_SA/docs/prd/MAPIT_STABILITY_PRD.md` | 17.5 KiB | 2026-01-27 | Older, entirely unticked version of virtually the same conversation stability plan. | 3/2/1/3/1 |
| `MAP-IT_SA/docs/prd/MAPIT_V3_RESPONSE_FORMATTER_PRD.md` | 27.8 KiB | 2026-01-27 | Largely open formatter plan that still assumes 34 tools and mentions non-existent tests. | 4/2/1/4/2 |

### Backend — `docs_tracked`

| Document | Size | Git date | Summary | C/A/U/H/R |
|---|---:|---:|---|---:|
| `MAP-IT_SA/docs_tracked/CONVERSATION_STABILITY_ARCHITECTURE.md` | 33.1 KiB | 2026-05-13 | Detailed design and debug guide for locks, atomic storage, scenario formats and test agents. | 4/3/3/4/2 |
| `MAP-IT_SA/docs_tracked/DEVELOPER_SETUP.md` | 6.5 KiB | 2026-05-19 | Local backend/database/Neo4j setup and V3/V4 sequence diagrams. | 3/1/2/4/3 |
| `MAP-IT_SA/docs_tracked/ENHANCED_HORIZON_QUESTIONS_CAPABILITIES.md` | 4.4 KiB | 2025-12-03 | Claims document for advanced call filtering, external API integration and performance figures. | 2/2/1/4/3 |
| `MAP-IT_SA/docs_tracked/EU_PROGRAMME_DATA_DOWNLOADS.md` | 11.0 KiB | 2026-05-13 | Download links, proposed directory structure and checklist for EU programme and project data. | 3/3/2/4/3 |
| `MAP-IT_SA/docs_tracked/FRONTEND_INTEGRATION_PROMPT.md` | 33.2 KiB | 2025-12-03 | Frontend build prompt for SSE, feedback, tool display, budget/timeline cards and error handling. | 4/2/1/3/1 |
| `MAP-IT_SA/docs_tracked/FRONTEND_STREAMING_V3_SPEC.md` | 18.3 KiB | 2025-12-03 | Historical transition specification from V2 to V3 including SSE events and a JS example client. | 4/1/1/4/1 |
| `MAP-IT_SA/docs_tracked/IMPROVEMENT_ROADMAP.md` | 69.4 KiB | 2025-12-19 | Extensive implementation roadmap for reliability, performance, features, security and K8s. | 5/2/1/2/1 |
| `MAP-IT_SA/docs_tracked/MANUAL_TEST_SCENARIOS.md` | 9.2 KiB | 2025-12-10 | Manual query catalogue, smoke test, format checks and outdated data-quality commands. | 3/2/2/4/2 |
| `MAP-IT_SA/docs_tracked/PROPOSAL_WRITER_FRONTEND_INTEGRATION.md` | 45.3 KiB | 2025-12-10 | Extensive V4 REST/SSE integration guide with types and mostly React examples. | 4/3/2/3/1 |
| `MAP-IT_SA/docs_tracked/QUESTION_CATEGORIES.md` | 34.0 KiB | 2025-12-09 | Very broad capability and scenario catalogue including legacy APIs, 20 tools and future exports. | 4/2/1/3/1 |
| `MAP-IT_SA/docs_tracked/README_CELERY.md` | 7.2 KiB | 2026-05-13 | Setup and operations guide for a Celery/API stack that does not exist in the repository. | 3/1/1/4/2 |
| `MAP-IT_SA/docs_tracked/SYSTEM_ARCHITECTURE.md` | 18.4 KiB | 2026-05-20 | Main architecture for V3/V4, 42 tools, data stores, services, configuration and example flows. | 4/3/3/4/2 |
| `MAP-IT_SA/docs_tracked/example_questions.md` | 21.9 KiB | 2025-12-03 | Large list of example questions about funding, proposals, compliance, operations and scaling. | 2/4/2/3/2 |

### Frontend — root and `docs`

| Document | Size | Git date | Summary | C/A/U/H/R |
|---|---:|---:|---|---:|
| `webapp-ui/README.md` | 2.1 KiB | 2026-05-28 | Minimal product introduction with scripts, tech stack and single-dev/security warning. | 2/3/3/4/3 |
| `webapp-ui/CRUSH.md` | 1.3 KiB | 2025-08-01 | AI agent conventions for build, lint, tests, database and style. | 2/2/1/4/2 |
| `webapp-ui/DEPLOYMENT_GUIDE.md` | 11.2 KiB | 2026-05-28 | VPS/PM2/NGINX/Supabase/backend deployment and troubleshooting guide for `mapit.work`. | 3/2/3/4/2 |
| `webapp-ui/SESSION_SUMMARY.md` | 6.2 KiB | 2026-05-28 | Historical account of a single deployment/auth-fix session with decisions made and follow-up actions. | 2/2/2/4/1 |
| `webapp-ui/TROUBLESHOOTING.md` | 5.4 KiB | 2026-05-28 | Quick production commands for login, mixed content, backend, PM2, Supabase and NGINX. | 3/2/3/4/2 |
| `webapp-ui/AI_Proposal_Writer_Frontend_Integration_Guide.md` | 45.3 KiB | 2025-12-10 | Almost duplicate of the backend V4 proposal integration guide with REST/SSE and React code. | 4/3/2/3/1 |
| `webapp-ui/proposal_editor_report.md` | 11.9 KiB | 2025-12-10 | Snapshot of proposal editor UI, stores, types, used/desired endpoints and recommendations. | 3/2/2/4/2 |
| `webapp-ui/docs/CITATION_IMPLEMENTATION_VERIFICATION.md` | 8.3 KiB | 2025-10-09 | Unexecuted verification checklist and proposed unit/integration/performance tests for citations. | 3/2/1/4/3 |
| `webapp-ui/docs/CONVERSATIONAL_AI_ARCHITECTURE.md` | 17.7 KiB | 2025-09-06 | Pre-V3 proposed conversation context architecture with Python pseudocode and open phases. | 4/1/1/3/1 |
| `webapp-ui/docs/CONVERSATION_IMPLEMENTATION_GUIDE.md` | 29.6 KiB | 2025-09-06 | Large implementation guide for backend services and routes that no longer exist. | 4/1/1/2/1 |
| `webapp-ui/docs/PROPOSAL_EDITOR_IMPLEMENTATION.md` | 8.3 KiB | 2025-10-13 | Feature description and usage guide for the live proposal editor, including mocks and future work. | 3/2/2/4/2 |
| `webapp-ui/docs/RATE_LIMITING.md` | 7.4 KiB | 2025-08-01 | Design and use of in-memory IP rate limiting with claimed global and route-specific tiers. | 4/2/1/4/3 |
| `webapp-ui/docs/SUPABASE_LOCAL_REALTIME_SETUP.md` | 2.8 KiB | 2025-08-01 | Local SQL and Supabase steps to add `message` to the realtime publication. | 3/3/2/4/2 |
| `webapp-ui/docs/SUPABASE_REALTIME_SETUP.md` | 3.3 KiB | 2025-08-01 | Production instructions and example RLS for collaboration chat via Supabase Realtime. | 3/2/1/4/2 |
| `webapp-ui/docs/project_docs.md` | 0.6 KiB | 2025-08-01 | WIP stub with only tech stack and timestamp convention. | 1/3/1/3/2 |
| `webapp-ui/docs/project_setup.md` | 2.3 KiB | 2025-08-01 | Old, manual VPS demo setup path with SSH forwarding, Supabase Studio and PM2. | 2/1/1/3/1 |

## Critical Gaps (P0 — must be resolved before handover)

1. **No full-stack handover/onboarding document.**
   - **Why critical:** a new developer must combine three different setup stories and infer ports, start order, repo relationships and database ownership on their own.
   - **Action:** create `START_HERE.md` in the NEWMAPIT root with system purpose, repos, owners, port matrix, minimal/full-data mode, exact start/stop order, smoke test, links to canonical docs and a "first day" checklist.

2. **A fresh clone cannot reproduce the data.**
   - **Why critical:** `project_embeddings.dump` is not tracked; `data/cordis` is missing; the docs mention no artifact store, version, checksum or download procedure. Without data, the core features do not work representatively.
   - **Action:** choose and document one supported bootstrap path: versioned dump in secured artifact storage ór fully reproducible raw-data ingest. Add a manifest, SHA-256, schema/embedding version, size, licence, owner, restore commands and verification queries.

3. **Setup commands are demonstrably wrong or internally contradictory.**
   - **Why critical:** wrong virtualenv name, double API start path, missing Supabase network precondition and erroneous quality-gate flags block onboarding.
   - **Action:** test all commands in a clean VM/container and publish one golden path. Have CI run the quickstart; document alternatives only after the working standard path.

4. **No canonical database and migration contract.**
   - **Why critical:** `migration/schema.sql`, `001_init_schema.sql`, the local/untracked `002_upgrade_embeddings_to_3072.sql`, frontend Drizzle migrations and `supabase_cli/schema.sql` are not connected as an ordered chain. The snapshot contains 768-dim document/call vectors while current embedding code and migration 002 point to 3072.
   - **Action:** designate one schema authority, create a migration ledger with applied status and rollback, document the 768→3072 impact, RLS/publications/RPCs and add schema drift checks.

5. **API contract does not describe auth, identity and tenancy.**
   - **Why critical:** the frontend creates a JWT, but FastAPI V3/V4 routes do not use `get_current_user` and V3 uses `user_id=1` by default. Docs suggest a secured user contract that the code does not enforce; data isolation therefore cannot be assessed.
   - **Action:** make OpenAPI + separate SSE event schemas canonical; record auth flow, claims, user-ID mapping, error semantics, versioning and ownership. Add contract tests between `webapp-ui` proxies/clients and FastAPI.

6. **Security handover is missing while concrete risks are present.**
   - **Why critical:** hardcoded Mapbox token, JWT fallback secret, `httpOnly: false` for access cookies, misleading `PUBLIC_SUPABASE_SERVICE_KEY`, disabled global rate limiting, open backend routes, missing collaboration membership checks and minimal public-table RLS are not centrally registered.
   - **Action:** create a threat model and security register before handover; rotate/centralise secrets, remove the public service-role naming, specify RLS/authz per table/route, document CORS/CSRF/cookies/rate limits and carry out a human security review.

7. **The documented test and quality status is not reproducible.**
   - **Why critical:** backend collection fails on `neo4j/import`; there is no CI; the frontend has no `npm run test`; `npm run check` gives 64 errors and `npm run lint` reports format problems in 119 files. PRD checkboxes are therefore not reliable release evidence.
   - **Action:** first repair collection/check gates, define unit/integration/live/LLM test profiles and required services, add hermetic smoke tests and CI, and publish one up-to-date test report per release.

8. **Deployment documentation is not executable as a production runbook.**
   - **Why critical:** NGINX config and PM2 state exist only on a specific VPS; K8s refers to non-existent Celery entrypoints; the frontend has no containers/manifests; there is no rollback or immutable image/tag policy.
   - **Action:** choose the supported production topology, put all config as code in Git, pin images/versions, document build→migrate→deploy→verify→rollback and test this on staging.

9. **Operations, backup and disaster recovery are missing.**
   - **Why critical:** three datastores plus Supabase auth/realtime require consistent backups and a restore order. Health endpoints are not a runbook.
   - **Action:** document monitoring, logs, metrics, alerts, SLOs, scheduled jobs/data freshness, capacity limits, backup retention, restore drills, RPO/RTO and escalation contacts.

10. **The actual production-readiness is hidden and contradicts the main text.**
    - **Why critical:** the tracked quality reports state "NOT PRODUCTION READY" and 54.8%/61% pass rate, but the docs index does not link them and several main docs call components production-ready.
    - **Action:** create a single dated `KNOWN_ISSUES_AND_RISKS.md` with evidence, impact, workaround, owner and target; let "production-ready" follow only from established release criteria.

## Important Improvements (P1 — should resolve)

1. **Restructure documentation by audience and lifecycle.** Create `docs/current/`, `docs/runbooks/`, `docs/decisions/` and `docs/archive/`; give each document an owner, status (`current`, `proposal`, `historical`), version and review date.
2. **Archive or label historical specs.** Move V2 streaming, Celery, the old conversation context implementation, the duplicate stability PRD and completed session reports out of the active path.
3. **Build the 42-tool reference from the registry.** Document per tool the exact name, input schema/defaults, output, datastore, failure/degraded mode, latency/cost, examples and tests; generate the list from `tool_registry.py` where possible.
4. **Publish a complete environment matrix.** Per variable: backend/frontend, required/optional, secret/public, default, local/staging/prod and source. Also include undocumented tuning vars (`MAX_CONTEXT_*`, `LLM_TIMEOUT_SECONDS`, etc.) and correct `DATABASE_URL` versus `PRIVATE_DATABASE_URL` for Drizzle.
5. **Create an up-to-date integrated architecture diagram.** Show browser, SvelteKit server routes, Supabase Auth/DB/Realtime, FastAPI, Redis, Neo4j, LLM providers, EU API, artifacts and trust/network boundaries.
6. **Write SvelteKit-native frontend integration docs.** The V4 guide uses approximately 1,700 lines of React/Redux examples in a SvelteKit repo; replace them with the actually used services, stores and proxy route.
7. **Make data provenance operational.** Record source URL, download time, programme year, rights/licence, parsing version, chunking, embedding model/dimension, row counts and freshness SLA.
8. **Centralise TODO/FIXME/risks.** Include at minimum the disabled rate limiting, three missing collaboration checks, missing email integration and hardcoded/local-only configuration.
9. **Document dependency and service versions.** Python/Node/Supabase CLI/PostgreSQL/pgvector/Neo4j/Redis/LLM models plus upgrade policy and compatibility matrix; correct Python 3.9 versus 3.11.
10. **Add release governance.** `CHANGELOG.md`, semantic app/schema/data versions, release notes, migration compatibility, rollback and ADRs for major architecture decisions.

## Nice-to-have (P2)

1. Add a funding-domain glossary (RIA, IA, CSA, call/topic, programme/cluster, RRF, RAG).
2. Add doc linting for broken links, unknown code paths, invalid CLI flags, outdated endpoint versions and age/owner metadata.
3. Create executable examples/Postman or `.http` collections for V3, V4, feedback, health and proposal document routes.
4. Provide example questions with expected tool choice, minimum answer contract and golden test ID instead of only loose prompts.
5. Add short "how to extend" guides for a new programme, tool, route, migration, SSE event and frontend card.
6. Publish a component/route map of SvelteKit and an up-to-date database ERD alongside the existing class diagram assets.
7. Add cost and quota information for Gemini/OpenAI/OpenRouter, embeddings, Supabase and Mapbox.

## Recommended Action Plan

| Order | Priority | Action | Estimated effort | Acceptance criterion |
|---:|---:|---|---:|---|
| 1 | P0 | Appoint owner; pin current code/schema/data/release baseline | 0.5 day | One tagged baseline with a responsible party per repo and datastore. |
| 2 | P0 | `START_HERE.md` + port/service matrix + fresh-clone quickstart | 1 day | A new developer reaches frontend, backend health and one chat smoke test without oral help. |
| 3 | P0 | Reproducible data bootstrap and artifact manifest | 1.5–2 days | Clean environment loads schema + minimal dataset; checksums and verification queries pass. |
| 4 | P0 | Canonical migration chain and schema/RLS drift control | 1.5–2 days | Empty DB migrates deterministically; 35 public tables/15 views and vector versions are checked. |
| 5 | P0 | Record API/SSE/auth contract and add contract tests | 1.5–2 days | Frontend clients validate against real V3/V4 schemas; user identity does not come from default `1`. |
| 6 | P0 | Security register + secrets/authz/RLS/rate-limit review | 2–3 days documentation/review, excluding fixes | All routes/tables have an authn/authz/RLS decision; secrets and known risks have an owner. |
| 7 | P0 | Restore test gates and introduce CI | 1–2 days | Backend collection/tests and frontend check/lint/tests run in CI with a separated live-test profile. |
| 8 | P0 | Deployment and operations runbook as code | 1.5–2 days | Staging deploy and rollback have been performed by a second person; backups have been test-restored. |
| 9 | P0 | Publish `KNOWN_ISSUES_AND_RISKS.md` | 0.5 day | Current blockers, quality baseline, TODOs, impact, workarounds and owners are centrally recorded. |
| 10 | P1 | Clean up docs: current versus archive, remove duplicates | 1–2 days | Main index contains only current docs; historical PRDs/specs are explicitly labelled. |
| 11 | P1 | Renew tool, env, dependency and data provenance references | 2–3 days | References are validated against code and manifests by scripts/CI. |
| 12 | P2 | Doc QA, glossary, examples and ADR template | 1 day | Broken-link/path/age checks run automatically and contributors have templates. |

**Indication:** approximately **14–20 developer days** for documentation, validation and limited supporting automation; actual security, typecheck, deployment and data migration fixes may come on top of that.

## Per-Document Details

### Backend root and guides

#### `MAP-IT_SA/README.md` — status: thoroughly revised

- **Issues:** quickstart combines `make docker-up` (API on 18000, plus external Supabase network) with `make run-reload` (second API on 8000); Supabase CLI/start location, frontend, data artifact and smoke test are missing. "Production-grade" clashes with its own audit warning and hidden quality results.
- **Fix:** make this a short entry point to the full-stack quickstart, supported modes, current readiness and canonical docs; move credits/contact details to ownership/support.

#### `MAP-IT_SA/CLAUDE.md` — status: keep as AI agent guide, clean up technically

- **Issues:** refers to removed `AgenticManager`, `agentic_manager.py`, `enhanced_rag_service.py`, `intent_detection_service.py`, `app/api/routes.py`, `celery_app.py`, a missing CI workflow and a misnamed `DATABASE_SCHEMA_DOCUMENTATION.md`. Due to its size and overlap it is not a safe onboarding SSOT.
- **Fix:** limit it to repository conventions and generated current references; link to separate architecture/API/data/test runbooks and add an automatic path check.

#### `MAP-IT_SA/GEMINI.md` — status: consolidate

- **Issues:** not in Git, repeats README/CLAUDE and mentions no distinct Gemini-specific workflow.
- **Fix:** delete or commit as a thin reference to one agent instruction file; avoid a parallel source of truth.

#### `MAP-IT_SA/questions.md` — status: rename and formalise

- **Issues:** no title, purpose, expected outcome, tool set, dataset version or test status.
- **Fix:** create `docs/examples/demo_queries.md` or parameterise as acceptance tests with expected tools/claims.

#### `MAP-IT_SA/docs/README.md` — status: replace

- **Issues:** mentions the non-existent `docs/implementation/` and four Erasmus files; statistics are wrong; "no redundancy" is demonstrably incorrect; hidden K8s/Supabase/quality docs are missing.
- **Fix:** generate the index from metadata and label each document current/proposal/historical/archive with owner and last-reviewed.

#### `MAP-IT_SA/docs/DATABASE_SCHEMA.md` — status: keep, generate/verify

- **Issues:** static record counts without snapshot date/query; no columns, constraints, RLS, ownership, migrations or ERD. The current worktree moves document/call/cache embeddings to 3072 while the snapshot still shows 768.
- **Fix:** generate object lists from migrations/live schema, add ERD, vector version matrix, RLS/publication/RPC details and reproducible count queries.

#### `MAP-IT_SA/docs/guides/DATA_INGESTION.md` — status: P0 rewrite

- **Issues:** dump not in clone; Neo4j ingest source missing; the mentioned quality flags do not exist; no input files/CLI options, idempotency, resume, cleanup, provenance or error recovery.
- **Fix:** build a tested runbook per dataset with prerequisites, source artifact, schema version, exact command, expected counts, retry/rollback and verification.

#### `MAP-IT_SA/docs/guides/ROADMAP.md` — status: replace with evidence-based status

- **Issues:** calls V3/multi-programme/V4 "production-ready" without release criteria and while current quality reports state the opposite; collaboration "early stage" is not linked to concrete TODOs.
- **Fix:** use an issue/risk register with evidence, owner, status date, scope and links to tests.

#### `MAP-IT_SA/docs/guides/TOOL_REFERENCE.md` — status: expand/generate

- **Issues:** title/intro claim 42 tools but a significant portion is missing; parameters sometimes deviate from the registry and output/error contracts are missing.
- **Fix:** generate tool names/input schemas from `tool_registry.py` and add output, datastore, degraded mode, examples and test links.

### Backend PRDs

#### `MAPIT_ADVANCED_CONVERSATION_FLOWS_PRD.md` — status: historical evidence, not current spec

- **Issues:** all tasks are ticked off, but the mentioned quality baseline file and CI workflow are missing; scenario counts and files are not linked as release evidence.
- **Fix:** archive with completion date/commit and link to an up-to-date test report; move persistent requirements to current acceptance criteria.

#### `MAPIT_APPLICATION_QUALITY_IMPROVEMENT_PRD.md` — status: archive

- **Issues:** says 34 tools and "all verified", but mentions ten non-existent scripts (`run_prd_scenarios.py`, sync/check scripts) and offers no reproducible results manifest.
- **Fix:** archive as a plan; carry open requirements and real test IDs over to a current quality plan.

#### `MAPIT_CONVERSATION_STABILITY_PRD.md` — status: archive after reconciliation

- **Issues:** useful checklists, but refers to the missing `docs/MAPIT_IMPROVEMENT_REPORT.md` and CI; duplicates `MAPIT_STABILITY_PRD.md` and the architecture guide.
- **Fix:** keep requirements/results in one status page; put implementation details in the stability architecture.

#### `MAPIT_MULTI_PROGRAMME_EXCELLENCE_PRD.md` — status: active but explicitly label as open

- **Issues:** 70 open checkboxes, non-existent `tests/test_multi_programme_scenarios.py`, and `search_digital_europe_templates` is not a registered tool.
- **Fix:** link requirements to existing tests/tools or create tickets; add owner, target release and dataset version.

#### `MAPIT_PRODUCTION_QUALITY_VALIDATION_PRD.md` — status: superseded/archive

- **Issues:** mixed 32 checked/130 open items without clear supersession; overlaps the V3 version and the advanced-flows PRD.
- **Fix:** mark "superseded by …" at the top; keep only historical traceability.

#### `MAPIT_PRODUCTION_QUALITY_VALIDATION_PRD_V3.md` — status: separate results from requirements

- **Issues:** refers to removed `agentic_manager.py`, `replanner.py`, `result_evaluator.py` and old nested scenario paths; final results stand apart from the separate "NOT PRODUCTION READY" reports.
- **Fix:** keep immutable requirements separate from generated test runs; link to one latest run with code/data/model version.

#### `MAPIT_QUALITY_PRD.md` — status: historical evaluation report

- **Issues:** ticked-off prompts mix offline samples, live scores and implementation claims; no date/model/dataset per score and no current regression status.
- **Fix:** archive the run and migrate prompts to versioned test cases; report current pass/fail in CI.

#### `MAPIT_STABILITY_PRD.md` — status: remove from active docs

- **Issues:** entirely unticked older duplicate version of conversation stability; raises doubt about whether work is or is not done.
- **Fix:** archive as superseded or delete after retention in Git history.

#### `MAPIT_V3_RESPONSE_FORMATTER_PRD.md` — status: update active plan

- **Issues:** assumes 34 tools, while the registry contains 42; `tests/test_formatter_integration.py` does not exist and formatter coverage is not measured automatically.
- **Fix:** rebase on the 42 registered tools, link each output form to tests and publish real coverage.

### Backend `docs_tracked`

#### `CONVERSATION_STABILITY_ARCHITECTURE.md` — status: keep, validate commands

- **Issues:** strong design, but too much overlap with CLAUDE/PRDs; health/debug routes and scenario run commands lack a proven fresh-clone context and CI is absent.
- **Fix:** make this the canonical stability design doc and move run commands to a tested test runbook.

#### `DEVELOPER_SETUP.md` — status: P0 rewrite

- **Issues:** `venv`/`.venv` mismatch; `make run` meaning wrong; Supabase is initialised in the backend root instead of using the existing frontend config; the V4 sequence mentions non-existent `/api/v4/proposals/sessions` routes; Neo4j copy can fail due to volume permissions.
- **Fix:** replace with the validated full-stack golden path and correct `/api/v4/proposal-session/...` routes.

#### `ENHANCED_HORIZON_QUESTIONS_CAPABILITIES.md` — status: add evidence or archive

- **Issues:** metrics (83.3%, 0.837 s) have no test/date/commit; the `client-centricity.io` architecture is not traceable in current code; "all completed" is not verifiable.
- **Fix:** link claims to benchmark results and current services, or label as a historical delivery note.

#### `EU_PROGRAMME_DATA_DOWNLOADS.md` — status: convert into a provenance manifest

- **Issues:** download/checklist document without artifact status, exact files, checksums, licences, ingest mapping or freshness; older 2025 links and manual steps dominate.
- **Fix:** create a machine-readable manifest per dataset and document sync owner/SLA and which files are already in `newdocs/`.

#### `FRONTEND_INTEGRATION_PROMPT.md` — status: archive after extraction

- **Issues:** mentions 20 tools and is a long build prompt, not a maintained contract; overlaps the tool guide, streaming spec and existing Svelte components.
- **Fix:** migrate persistent UX/event requirements to SvelteKit component and contract docs; archive the prompt.

#### `FRONTEND_STREAMING_V3_SPEC.md` — status: mark as historical immediately

- **Issues:** V2 endpoint and removed files are "current"; V3 is "pending"; example uses V2; fallback does not exist. Complete event form and agent iteration events deviate from current code.
- **Fix:** replace with a schema from real emissions in `ConversationManager`/`ToolAwareAgent`, including top-level versus `metadata` fields, ordering, storage errors and reconnect/cancel semantics.

#### `IMPROVEMENT_ROADMAP.md` — status: split up and archive

- **Issues:** status at the top says many priorities have been implemented, but "Current State" says the opposite; 20 tools; many proposed migrations/files do not exist; 2,277 lines are unusable as current issue management.
- **Fix:** archive design examples and put remaining work in a short, owner-driven register or issue tracker.

#### `MANUAL_TEST_SCENARIOS.md` — status: keep after executable linking

- **Issues:** useful questions but no expected outputs/data baseline; two mentioned enrich scripts are missing; performance criteria are not linked to measurements.
- **Fix:** link each scenario to test ID, preconditions, expected tool/contract and run record; remove invalid scripts.

#### `PROPOSAL_WRITER_FRONTEND_INTEGRATION.md` — status: one canonical copy, create Svelte version

- **Issues:** virtually identically duplicated in the frontend; React/Redux examples do not fit SvelteKit; migration `015_proposal_sessions.sql` is missing, while the real routes are mostly correct.
- **Fix:** keep the contract with the backend/OpenAPI, create only Svelte services/stores/examples in the frontend and link both.

#### `QUESTION_CATEGORIES.md` — status: update capability matrix

- **Issues:** mentions 20 tools, legacy `/api/query*` routes that are not mounted by FastAPI, and features such as exports/version comparison without status evidence.
- **Fix:** split supported/tested/planned; generate endpoints/tools and link scenarios to evidence.

#### `README_CELERY.md` — status: remove from active docs

- **Issues:** almost all operational claims cannot be executed due to missing compose, workers, task module, endpoints and test script.
- **Fix:** explicitly archive as an abandoned design, or implement and test the stack before republishing.

#### `SYSTEM_ARCHITECTURE.md` — status: canonical basis, supplement

- **Issues:** the core V3/V4 and 42 tools are usable, but the env example uses `SUPABASE_KEY` instead of anon/service keys; frontend/auth/deployment/trust boundaries are missing; the RRF claim is not equally precise everywhere; the link to the tool guide overstates completeness.
- **Fix:** make this the integrated current architecture with version/owner/review date and generated service/route/tool references.

#### `example_questions.md` — status: reduce or make testable

- **Issues:** hundreds of questions without classification of supported/aspirational, expected tools or truth; overlap with `questions.md`, manual scenarios and PRDs.
- **Fix:** curate a small onboarding/demo set and move the rest to versioned evaluation data.

### Frontend root and docs

#### `webapp-ui/README.md` — status: expand

- **Issues:** no prerequisites/Node version, env setup, Supabase start, backend start, auth flow, test status, routes or troubleshooting links. Database commands hide the `DATABASE_URL` versus `PRIVATE_DATABASE_URL` mismatch.
- **Fix:** make the README the frontend entry point to the unified quickstart and document actually supported scripts and quality gates.

#### `webapp-ui/CRUSH.md` — status: correct/consolidate

- **Issues:** mentions the non-existent `npm run test`; "Svelte 5+ conventions" is not linked to current migration guidance; overlap with the README.
- **Fix:** generate the command list from `package.json`; keep only agent-specific conventions.

#### `webapp-ui/DEPLOYMENT_GUIDE.md` — status: make reproducible as code

- **Issues:** hardcoded domain/VPS and `/etc/nginx` state; no complete NGINX file, DNS/firewall/Supabase service lifecycle, frontend health check, rollback, backups or secret manager. Python 3.9 deviates from backend 3.11+. Cookie advice encourages `httpOnly: false`; the frontend check fails.
- **Fix:** version NGINX/PM2 or container manifests, separate local/staging/prod, pin runtimes and add build/check/migrate/deploy/rollback verification.

#### `webapp-ui/SESSION_SUMMARY.md` — status: archive

- **Issues:** a time-bound incident/work session is used as an onboarding source; the code example for `response.headers.set` with cookie options is incorrect in current TS and the fix is domain-specific.
- **Fix:** put persistent decisions in an ADR/runbook and archive the session report.

#### `webapp-ui/TROUBLESHOOTING.md` — status: rewrite as a runbook

- **Issues:** advises changing live code for cookie behaviour, uses a single production domain, lacks a safe diagnosis order/rollback and treats symptoms without auth/Supabase/backend ownership.
- **Fix:** add severity, preconditions, read-only checks, expected output, safe remediation, rollback and escalation.

#### `AI_Proposal_Writer_Frontend_Integration_Guide.md` — status: deduplicate

- **Issues:** almost a copy of the backend guide, React code in a Svelte repo, missing migration and no link to the actually implemented `proposalSessionService.ts`/store/proxy.
- **Fix:** replace with a short SvelteKit implementation guide that links to one backend contract.

#### `proposal_editor_report.md` — status: label as snapshot

- **Issues:** the "currently used" list contains the non-existent `/api/proposal/generate-template` and `/events`; recommendation and implementation status are mixed up; the frontend component moreover contains a hardcoded `http://localhost:8000` export URL.
- **Fix:** date/commit-label the snapshot or replace with a current component/route/state diagram with known gaps.

#### `CITATION_IMPLEMENTATION_VERIFICATION.md` — status: formalise open test plan

- **Issues:** the mentioned `citationRenderer.test.ts` and test infrastructure are missing; the checklist has not been executed; the actual persistence/schema/proxy flow is not recorded as a contract.
- **Fix:** implement or ticket tests, add a test runner and publish the run result/browser matrix.

#### `CONVERSATIONAL_AI_ARCHITECTURE.md` — status: archive as pre-V3 proposal

- **Issues:** proposes `conversation_context_service.py`, `rag_service.py`, old routes and DB changes that do not form the current V3 memory/stability architecture; all phases are open.
- **Fix:** mark superseded by the backend stability/system architecture and remove from onboarding.

#### `CONVERSATION_IMPLEMENTATION_GUIDE.md` — status: archive

- **Issues:** 945 lines of copy-paste Python for non-existent backend files in the frontend repo; old `/api/conversation` contracts and data models.
- **Fix:** keep only as history; do not write a backend implementation guide in frontend docs.

#### `PROPOSAL_EDITOR_IMPLEMENTATION.md` — status: update

- **Issues:** claims comprehensive error handling/mobile/accessibility and mock routes without automated evidence; links to the missing `API_DOCUMENTATION.md` and `COMPONENT_LIBRARY.md`; future work overlaps collaboration code that now exists.
- **Fix:** test claims against current UI/tests, repair links and separate implemented/partial/planned.

#### `RATE_LIMITING.md` — status: correct, security-critical

- **Issues:** global protection is described as active while the middleware is off; the in-memory limiter is not cluster-safe and production proxy trust/config is missing.
- **Fix:** document actual coverage per route, enablement flag, trusted proxy chain, Redis/distributed strategy, observability and contract tests.

#### `SUPABASE_LOCAL_REALTIME_SETUP.md` — status: integrate into unified setup

- **Issues:** manual SQL is already present in the large schema dump but migration ownership is missing; the `supabase/migrations` path does not exist in the `supabase_cli` structure used.
- **Fix:** put the publication change in the canonical migration chain and make this a short verification/troubleshooting section.

#### `SUPABASE_REALTIME_SETUP.md` — status: rewrite and validate RLS

- **Issues:** example RLS is not included as a migration and the actual schema dump does not enable RLS for `message`, `conversation`, `client` or collaboration tables. The temporary permissive policy lacks a hard warning/cleanup check.
- **Fix:** let security-reviewed policies apply as migrations and document claims, membership model, Realtime replica identity and tests.

#### `project_docs.md` — status: delete or complete

- **Issues:** explicit WIP stub without current architecture, setup or ownership; adds hardly any value next to the README.
- **Fix:** merge valid conventions into the README/database guide and archive the stub.

#### `project_setup.md` — status: label as legacy/destructive immediately

- **Issues:** advises deleting migrations, running the schema manually in Studio, opening the Supabase API port publicly and using an old PM2 path; conflicts with the more recent deployment guide.
- **Fix:** archive; refer exclusively to the validated, automated setup/deploy path.

## Conclusion for handover

The documentation shows much domain knowledge and gives a new developer useful individual building blocks, but does not yet function as a handover package. The decisive step is not "writing more documentation", but **creating one demonstrably working source of truth**: fresh-clone setup, versioned data/schema, a real API/auth contract, reproducible quality gates, a security register and a production runbook. Only when a second developer has completed that route without implicit knowledge and the P0 acceptance criteria have been recorded is MAPIT documentation-ready for handover.
