# Known issues and risks

**Status date:** 20 July 2026  
**Scope:** `MAP-IT_SA` (backend) and `webapp-ui` (frontend)  
**Release decision:** **NOT PRODUCTION READY**  
**Register owner:** TBD

This is the central, candid status register for handover. An item is only closed after the solution has been implemented in code or infrastructure and reproducibly verified. A checked-off PRD item does not count as evidence. See also the [security register](SECURITY_REGISTER.md), the [database schema](MAP-IT_SA/docs/DATABASE_SCHEMA.md), and the [onboarding guide](START_HERE.md).

## Quality baseline

The latest available final quality report is dated **23 January 2026**; there is no more recent release evidence in the repository.

| Source | Result | Meaning |
|---|---:|---|
| `MAP-IT_SA/tests/quality/results/FINAL_VERIFICATION_SUMMARY.md` | 23/42 scenarios passed; **54.8%**; average quality 82.0/100; factual 97.9% | Explicit status: **NOT PRODUCTION READY**; the required pass rate was 80%. |
| `MAP-IT_SA/tests/quality/results/PRODUCTION_READINESS_SUMMARY.md` | 25/41 identified scenarios passed; **61.0%**; average quality 81.4/100 | Earlier run/summary; likewise explicitly **NOT READY**. |
| `MAP-IT_SA/tests/quality/results/production_readiness_report.json` | Same 61.0% run, generated 23 January 2026 15:53 | Machine-readable source accompanying the earlier summary. |
| Documentation review of 20 July 2026 | `make test`/`pytest -q` stops during collection at `neo4j/import`; `npm run check` reports 64 errors and 76 warnings; `npm run lint` reports 119 files | The current overall quality gates are not green. The frontend has no `test` script. |

The two scenario pass rates are not alternative success criteria: they are different snapshots of the same validation campaign. The final, later-generated 54.8% report is authoritative. Neither proves production readiness.

## Meaning of severity and status

- **critical:** real risk of loss of data isolation, secret exposure, non-reproducible core functionality, or an irresponsible production launch.
- **high:** blocks reliable handover, release validation, or recovery; may affect a large proportion of users.
- **medium:** significant limitation or debt with a defined workaround.
- **low:** limited impact or primarily a maintenance burden.
- **open:** unresolved; **in-progress:** demonstrably in progress; **accepted:** explicitly accepted with an owner and reassessment date.

## Known issues

| ID | Description and evidence | Severity | Impact | Workaround | Owner | Status |
|---|---|---|---|---|---|---|
| QUAL-001 | Final scenario pass rate is 54.8%, below the required 80%; 19 of 42 scenarios fail. | critical | Responses and multi-turn flows have not been sufficiently validated for production. | Controlled demo/staging only; verify critical responses manually. | TBD | open |
| QUAL-002 | Backend tests are collected by `pyproject.toml` from both `tests` and `.`; collection reaches the unreadable bind mount `neo4j/import`. | high | `make test` does not provide a reliable regression assessment. | Fix permissions or exclude runtime volumes in the pytest configuration before treating the results as a gate. | TBD | open |
| QUAL-003 | According to the review, `npm run check` has 64 errors/76 warnings; lint has 119 formatting deviations. | high | The frontend build and runtime behavior have not been reliably cleared for release. | No production deployment; enforce `npm run check`, `npm run lint`, and `npm run build` as hard gates. | TBD | open |
| QUAL-004 | `webapp-ui/package.json` contains no `test` script, and neither repository contains a CI workflow. | high | No automated contract, browser, or regression protection. | Manual smoke tests from [START_HERE](START_HERE.md); this is not an equivalent replacement. | TBD | open |
| DATA-001 | The referenced PostgreSQL dump is not present in a fresh clone: `/neo4j_backup/` is ignored and only a local dump was found. | critical | The representative core dataset cannot be built independently. | Request a version, checksum, and transfer channel from the data owner, or perform raw ingestion according to [DATA_INGESTION](MAP-IT_SA/docs/guides/DATA_INGESTION.md). | TBD | open |
| DATA-002 | `scripts/ingest_cordis_to_neo4j.py` expects `data/cordis` by default, but `/data/` is ignored and the source artifact is missing. | high | Neo4j cannot be populated from a clone. | Provide CORDIS artifacts outside Git with a manifest/checksum; do not start with an empty directory without provenance. | TBD | open |
| DATA-003 | The schema snapshot has 768-dimensional vectors for `funding_calls`, `document_chunks`, and `embedding_cache`, while config/code specify 3072 and migration `002`; existing values are deleted in the process. | critical | Vector insert/search may produce dimension errors; the migration deliberately requires re-embedding. | Disable vector search until the schema and data use the same version; create a backup first and then re-embed everything. | TBD | open |
| DATA-004 | There is no canonical migration chain applied by tooling: full dumps, `001`, `002`, `008`, `009`, and the Drizzle schema coexist. | high | Schema drift and incomplete deployments are likely. | Temporarily follow the ledger in [DATABASE_SCHEMA](MAP-IT_SA/docs/DATABASE_SCHEMA.md) and record the applied checksums for each environment. | TBD | open |
| API-001 | V3/V4 FastAPI routes do not apply the available `get_current_user` dependency; V3 request models use `user_id=1` by default. | critical | A direct backend caller can choose/share an identity; tenant isolation is not enforced. | Do not expose the backend publicly; allow only the controlled frontend proxy. This is defense-in-depth, not a definitive fix. | TBD | open |
| API-002 | SSE schemas appeared only in an outdated V2/V3 specification, and events do not consistently have the same top-level fields. | high | The frontend may miss or mishandle events. | Use [API_CONTRACT](API_CONTRACT.md) as the current code snapshot; add contract tests. | TBD | open |
| SEC-001 | A Mapbox token is hardcoded in the frontend source. | medium | Risk of token misuse and quota/cost overruns; rotation requires a code fix. | Restrict the token by origin/scopes in Mapbox and rotate it; details are in [SECURITY_REGISTER](SECURITY_REGISTER.md). | TBD | open |
| SEC-002 | Backend JWT validation uses a fallback secret when configuration is missing. | critical | Tokens can be forged if the fallback is active. | Set a unique, strong `JWT_SECRET_KEY`, block startup when a placeholder/fallback is used, and rotate existing tokens. | TBD | open |
| SEC-003 | Auth access cookies are set with `httpOnly: false`. | high | XSS can read tokens; increased risk of account takeover. | Limit session duration and use CSP; the structural fix is server-only HttpOnly cookie management. | TBD | open |
| SEC-004 | `PUBLIC_SUPABASE_SERVICE_KEY` suggests a public service-role key and is populated as such in `.env.example`. | critical | Service-role exposure bypasses RLS and can compromise the entire database. | Use only the server-private `SUPABASE_SERVICE_ROLE_KEY`; scan builds/logs and rotate in the event of possible exposure. | TBD | open |
| SEC-005 | Global rate limiting in `webapp-ui/src/hooks.server.ts` is commented out. | high | Login/API/LLM routes are more vulnerable to abuse and cost escalation. | Apply temporary limits in NGINX/firewall; an application-level shared limiter remains necessary. | TBD | open |
| SEC-006 | Collaboration routes contain three explicit TODOs for membership checks. | critical | Possible access to conversations belonging to other collaborations. | Do not enable Collaboration for untrusted multi-user production. | TBD | open |
| SEC-007 | In the schema snapshot, only `document_chunks` and `horizon_documents` have public-table RLS; among others, conversation/collaboration/message do not have RLS. | critical | Direct Supabase access may expose or modify data outside the intended user/tenant. | Restrict anon/authenticated grants, use a server-side service key, and implement tested policies before production. | TBD | open |
| DEP-001 | Backend Compose uses `--reload`, bind-mounts the source, and requires the external network `supabase_network_supabase_cli`. | high | This is a development stack, not an immutable or self-contained production artifact. | Use the explicit VPS path in [DEPLOYMENT_RUNBOOK](DEPLOYMENT_RUNBOOK.md); do not expose the Compose API publicly. | TBD | open |
| DEP-002 | K8s manifests launch `app.services.celery_app`, which does not exist, reference `/metrics`, which does not exist, use `latest`, and contain no frontend deployment. | high | `kubectl apply -k` does not produce a complete, working production environment. | Kubernetes is not supported; use the selected VPS topology. | TBD | open |
| OPS-001 | Backups, restore drills, alerts, SLOs, and on-call coverage have not been implemented or assigned. | high | Data and recovery time are not controlled. | Implement the proposed procedures from [OPERATIONS_RUNBOOK](OPERATIONS_RUNBOOK.md) and record every restore test. | TBD | open |
| OPS-002 | There is no working scheduler: Celery worker/beat are absent; `process-alerts` is a manual endpoint, and incremental embeddings are designed only for cron/Celery. | high | Call alerts, synchronization, and embeddings are not guaranteed to be refreshed. | Run manually with a run log; only then configure a managed scheduler with locking/alerting. | TBD | open |
| FEAT-001 | Collaboration and consortium screens explicitly indicate that functionality is temporarily disabled. | medium | The product scope is smaller than some roadmap claims. | Treat it as unavailable and communicate this to users. | TBD | open |
| FEAT-002 | Saved-search email notifications have a TODO for email integration. | medium | Alerts are not delivered by email. | Retrieve results manually; do not promise email alerts. | TBD | open |

## Known limits

| Area | Current limit |
|---|---|
| Vector search | The active embedding config is `gemini-embedding-001` with 3072 dimensions, but the schema snapshot is partly 768. pgvector indexing above 2000 dimensions also requires version/index choices; migration `002` currently selects exact/sequential search. |
| Program maturity | Horizon Europe has the richest project, call, graph, and scenario support. Erasmus+ and Creative Europe have specific tools and scenarios. LIFE, Digital Europe, and EU4Health are present in routing/knowledge but do not have comparable scenario coverage in the final 42-scenario run; treat them as limited/experimental. |
| Data reproducibility | Complete project data, CORDIS raw files, and a version/checksum manifest are missing from Git. An empty schema is reproducible; a representative dataset is not. |
| Tooling | `tool_registry.py` registers 42 tools, but the existing tool guide describes only a subset. Registration does not prove that a tool has been populated and tested for every dataset or program. |
| Supabase API | Local `config.toml` limits REST responses to 1000 rows. Client code must paginate. |
| Uploads | The backend default is 50 MiB; the frontend config specifies 10 MiB. The lower limit is effective until they are aligned. |
| External providers | Gemini/OpenAI/OpenRouter, Google embeddings, and the EU Portal fall outside MAPIT's availability controls. Timeouts, quotas, model changes, and costs may degrade responses. |
| Redis | Memory/cache may degrade to local/in-memory fallbacks. These are not shared between processes and are lost on restart. |
| Scalability | Local Compose runs one API process and single-node Redis/Neo4j. There are no reliable cluster/load tests or active distributed rate limits. |
| Analytics | Analytics endpoints and health metrics exist, but are not authenticated, and `/api/health/metrics` is JSON, not Prometheus exposition format. There is no dashboard or Alertmanager configuration. |

## Technical debt and disabled components

| Item | Evidence/location | Consequence |
|---|---|---|
| Global limiter disabled | `webapp-ui/src/hooks.server.ts` | Only any route/proxy/NGINX limits protect traffic. |
| Membership checks missing | `webapp-ui/src/routes/api/conversations/+server.ts`, `.../[conversation_id]/+server.ts`, `api/chat/websocket/+server.ts` | Collaboration isolation has not been demonstrated. |
| Email integration missing | `MAP-IT_SA/app/services/alerts/saved_search_service.py` | Saved-search alerts cannot be considered an email product. |
| Celery design without implementation | `MAP-IT_SA/k8s/deployment.yaml`; archived Celery guide | Worker, beat, and periodic processing do not start. |
| Collaboration/consortium UI disabled | Two dashboard pages in `webapp-ui/src/routes/(protected)` | Navigation may show features that are present but unavailable. |
| Analytics can silently disable itself | `conversation_analytics_service.py`, `tool_analytics_service.py` | Missing DB tables/connections cause gaps in observability. |
| No release automation | No CI workflows; no changelog/release tag policy in scope | Build, schema, and documentation drift are not stopped automatically. |
| Dev container presented as deployment | `docker-compose.yml` uses reload/bind mount/default Neo4j password | The configuration is suitable for local use, not for production hardening. |

`pass` in abstract base classes or exception cleanup is not automatically a defect and is therefore not included as a separate issue. The debt above consists of concrete open TODOs, missing entry points, or demonstrably disabled features.

## Contradictions in documentation

| Contradictory claim | Canonical truth as of 20 July 2026 |
|---|---|
| README/roadmap describes components as “production-ready”. | The final quality baseline says **NOT PRODUCTION READY**, 54.8%. This register is authoritative for readiness. |
| Old setup says `source .venv/bin/activate`. | `scripts/setup.sh` creates `venv`; use `source venv/bin/activate`. |
| Old setup suggests that `make run` starts Compose. | `make run` starts local Uvicorn on 8000; `make docker-up` starts Compose and publishes the API on 18000; `make run-reload` is local Uvicorn with reload on 8000. |
| Setup implies that backend Compose starts independently. | Compose expects the external network `supabase_network_supabase_cli`; start Supabase from `webapp-ui/supabase_cli` first. |
| Old V4 documentation names `/api/v4/proposals/sessions`. | The actual prefix is `/api/v4/proposal-session`; see [API_CONTRACT](API_CONTRACT.md). |
| Streaming document treats V2 as current and V3 as pending. | Only V3/V4 is mounted; the old streaming document is archived. |
| Celery and K8s documentation claim workers/beat. | `app.services.celery_app` and a working Celery stack are missing; do not deploy. |
| Data ingestion guide claimed an included dump and `data/cordis`. | Both are missing from a fresh clone; see the new ingestion runbook. |
| Data ingestion guide used `quality_gate.py --check-embeddings/--check-calls`. | Those flags do not exist. Valid flags are `--results-dir`, `--quality-baseline`, `--threshold`, and `--fail-on-missing`. |
| Rate limit guide says that global 100/min limiting is active. | The global middleware is commented out; consider global application rate limiting disabled. |
| Deployment guide specifies SvelteKit production on 5173. | 5173 is Vite development. Adapter-node uses internal port 3000 for this runbook via explicit `PORT=3000`. |
| K8s README claims Prometheus at `/metrics`. | Only `/api/health/metrics` exists and returns JSON; there is no Prometheus endpoint at `/metrics`. |

## Maintaining this register

For every release, the owner must:

1. record the status date, commit SHAs, and data/schema/embedding version;
2. rerun the backend and frontend gates and link the results;
3. link closed items to test evidence and the migration/deployment record;
4. add new security and operations findings;
5. set “production ready” only when all critical/high release blockers have been formally closed or explicitly accepted with an owner, compensating control, and end date.
