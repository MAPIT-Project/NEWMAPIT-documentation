# Operations and incident runbook

**Status date:** 20 July 2026  
**Topology:** the VPS path from [DEPLOYMENT_RUNBOOK](DEPLOYMENT_RUNBOOK.md)  
**Operations owner / on-call:** TBD  
**Time zone for run logs:** UTC

This document describes daily operations and sets initial SLO, backup and DR targets. The repository does not yet contain dashboards, an alert manager, scheduler, on-call roster or evidence of restore drills. Everything marked "proposed" below is therefore a control to be introduced, not an existing guarantee.

## 1. Daily start check

```bash
sudo systemctl is-active mapit-api nginx docker
pm2 status
cd /srv/mapit/current/backend
docker compose --env-file /etc/mapit/backend.env -p mapit ps
curl --fail http://127.0.0.1:8000/api/health/live
curl --fail http://127.0.0.1:8000/api/health/ready
curl --silent http://127.0.0.1:8000/api/health
curl --silent http://127.0.0.1:3000/api/backend-status
```

For the last two calls, also read the JSON body. The full health route does not automatically return HTTP 503 with an `unhealthy` body; the frontend status reports `connected` as soon as HTTP is reachable, even when `backend_status` is degraded/unhealthy.

Then check:

- errors/restarts since the previous shift;
- backup jobs and checksum/retention;
- disk, inodes, RAM/swap and container status;
- expired/nearly expired TLS certificates and provider quotas;
- data freshness and failed ingest/embedding run logs;
- open critical/high incidents in [KNOWN_ISSUES_AND_RISKS](KNOWN_ISSUES_AND_RISKS.md).

## 2. Health endpoints

| Endpoint | Actual check | Status semantics | Use |
|---|---|---|---|
| `GET /api/health/live` | Only the FastAPI process/event loop responds. | Always 200 with `alive: true` as long as the handler runs. | Liveness, every 30 s. |
| `GET /api/health/ready` | PostgreSQL; Gemini only if a Gemini key is set. | 200 when ready, 503 on a failing critical check. | Load balancer/readiness, every 30 s. |
| `GET /api/health` and `/api/health/` | PostgreSQL, Neo4j, Redis, Gemini, embedding service and Supabase in parallel. | Body `healthy`, `degraded` or `unhealthy`; handler does not set 503 itself. Redis/embedding are non-critical, PostgreSQL and configured Gemini are. | Synthetic dependency check, every 2 min. |
| `GET /api/health/dependencies` | The same dependencies plus critical/fallback labels and configuration summary. | JSON, no promised non-200 on degradation. | Diagnosis, do not expose publicly. |
| `GET /api/health/metrics` | PostgreSQL, Neo4j and Redis status/latency. | JSON with 1/0, 5/0, **no Prometheus exposition format**. | Internal JSON scrape/exporter. |
| `GET /api/analytics/health` | In-memory analytics service, buffer and last flush. | `healthy` or `disabled`; endpoint is currently not authenticated. | Internal diagnosis; block externally. |
| `GET /api/alerts/health` | Saved-search service. | Simple service health. | Internal diagnosis; block externally. |
| `GET /api/v4/proposal-session/health` | Intended for session manager, proposal agent and database. | Declared in source, but probably matched by the earlier `GET /{session_id}` and may yield 404. | Do not use as a probe until route order is fixed and tested. |
| `GET /api/backend-status` on SvelteKit | SvelteKit fetches FastAPI `/api/health`. | 200 `connected` on a reachable response; 503 only on fetch error. | End-to-end proxy reachability, assess the JSON body. |

OpenAPI is available on the backend `/docs`, `/redoc` and `/openapi.json`, but in production must be placed behind admin access or disabled.

## 3. Logs and diagnosis

The applications configure logging to stdout/stderr by default; there is no application `FileHandler` or central log collector in the code.

| Component | Location/command | Note |
|---|---|---|
| FastAPI/systemd | `journalctl -u mapit-api --since -1h` | Add `-f` for follow; journald retention is OS configuration. |
| SvelteKit/PM2 | `pm2 logs mapit-ui --lines 200` | PM2 files are by default under the PM2 user's home in `.pm2/logs/`; use the command as canonical access. |
| Redis/Neo4j | `docker compose --env-file /etc/mapit/backend.env -p mapit logs --since 1h redis neo4j` | Neo4j additionally has a named `neo4j_logs` volume; manage via container/Compose. |
| NGINX | `/var/log/nginx/access.log`, `/var/log/nginx/error.log` | Ensure log rotation and masking of tokens/query data. |
| Supabase managed | Provider project dashboard/log explorer | Manage access and retention outside this repo. |
| Deploy/ingest/backup | Central run log still to be created | Record command version, SHA, data artifact checksum, start/end, counts and operator. |

Safe triage:

```bash
journalctl -u mapit-api --since -30min --no-pager
pm2 describe mapit-ui
pm2 logs mapit-ui --lines 100 --nostream
cd /srv/mapit/current/backend
docker compose --env-file /etc/mapit/backend.env -p mapit ps
docker compose --env-file /etc/mapit/backend.env -p mapit logs --since 30m redis neo4j
sudo nginx -t
sudo tail -n 100 /var/log/nginx/error.log
```

Do not log JWTs, cookies, service-role keys, full prompts with personal data or database URLs. A discovered secret counts as a security incident: revoke/rotate access and determine scope.

## 4. Metrics and analytics

Available telemetry:

- `/api/analytics/overview?days=30`: usage, performance, content, proposals and feedback from PostgreSQL;
- `/api/analytics/tools?hours=24`: tool performance, slow tools, combinations and process-local session metrics;
- `/api/analytics/trends`, `/queries/popular`: time series and popular queries;
- V3 `/api/v3/conversation/analytics/summary`, `/low-confidence`, `/search`, `/export`;
- `tool_usage_analytics`, `tool_combination_analytics`, `conversation_analytics` and `response_feedback` in PostgreSQL;
- request ID and timing via backend middleware;
- JSON dependency latencies via `/api/health/metrics`.

Limitations:

- analytics can silently initialise as `disabled` on database errors;
- tool analytics buffers up to 50 events or approximately 60 seconds in process memory; a crash can lose buffer data;
- process-local session metrics are not shared between Uvicorn workers;
- analytics and export endpoints have no backend auth; do not publish them;
- there is no real `/metrics` route and no Prometheus/Grafana configuration in the repo.

Proposed minimum dashboard panels: request volume/status/latency, SSE start and stream duration, active streams, provider errors/429/timeouts, tool success/latency, PostgreSQL connections/query latency, Redis memory/evictions, Neo4j nodes/relationships/query latency, container/process restarts, disk/inodes, backup age and data/embedding freshness.

## 5. Alerts

Proposed alerts; adjust thresholds after 30 days of baseline.

| Alert | Initial threshold | Urgency |
|---|---|---|
| Liveness fails | 2 consecutive checks | P1 |
| Readiness fails | 3 checks / 2 minutes | P1 |
| Full health `unhealthy` | 2 checks; PostgreSQL/Supabase immediately | P1 |
| Full health `degraded` | 10 minutes | P2 |
| HTTP 5xx | >2% for 5 minutes and at least 20 requests | P1/P2 by impact |
| SSE start latency | p95 >5 s for 15 minutes | P2 |
| Provider 429/timeouts | >5% for 10 minutes | P2; P1 if all providers fail |
| Tool failures | >10% per frequently used tool in 15 minutes | P2 |
| Redis/Neo4j down | >5 minutes | P2 as long as fallback works, otherwise P1 |
| Database connections | >80% of limit for 10 minutes | P2 |
| Disk/inodes | >80% warning, >90% critical | P2/P1 |
| Process/container restart | >3 in 10 minutes | P1 |
| Backup | last successful PostgreSQL backup >26 hours; checksum/verify fails immediately | P1 |
| Data freshness | agreed dataset SLA exceeded | P2 |
| TLS | <21 days until expiry; renewal test fails | P2/P1 |

Alerts must go to a staffed channel with acknowledgement, runbook link and escalation. Owner/channel are currently **TBD**; this is itself a release blocker.

## 6. Proposed SLOs

These targets have not yet been measured or contractually committed.

| SLI | Proposed SLO per calendar month |
|---|---:|
| Public web app + login available | 99.5% |
| FastAPI readiness | 99.5% |
| Successful non-streaming API requests (no 4xx) | 99.0% |
| V3 streams that start a first SSE event | 95% within 5 s; 99% within 15 s |
| V3 streams with `complete` without `error` | 98% |
| Critical database writes | 99.9% without loss/duplication |
| Data freshness | according to the per-dataset SLA below |

Planned maintenance only counts outside the availability SLI when announced in advance and measured separately. External provider errors may be broken out separately in reporting, but user impact remains visible.

## 7. Backups

### Policy and retention (proposed)

| Store | Frequency | Retention | RPO target | Authority |
|---|---|---|---|---|
| PostgreSQL/Supabase | Provider PITR if available + daily logical dump | 7 daily, 5 weekly, 12 monthly | 1 hour with PITR, otherwise 24 hours | Primary for app/auth/content/analytics. |
| Neo4j | Daily offline dump after the PostgreSQL backup | 7 daily, 5 weekly | 24 hours | Derived graph data, but the raw source is now partly missing; backup therefore necessary. |
| Redis | Daily RDB/AOF copy for diagnosis; not as a primary recovery source | 2 days | maximum 24 hours, but sessions/cache may be lost | Cache/short-term state, not source of truth. |
| Config/secrets | Encrypted secret-store versions on every change | At least 12 months audit trail | Last valid version | No plaintext secrets in Git/backup bundle. |

Encrypt backups client-side or in the backup service, store them outside the VPS/account failure domain, restrict access, create SHA-256 checksums and test restores quarterly.

### PostgreSQL

For managed Supabase, use provider PITR/snapshots first. Additional logical dump:

```bash
install -d -m 0700 /srv/mapit/backups/postgres
set -a
. /etc/mapit/backend.env
set +a
pg_dump --format=custom --no-owner --no-acl \
  --file=/srv/mapit/backups/postgres/mapit-$(date -u +%Y%m%dT%H%M%SZ).dump \
  "$DATABASE_URL"
sha256sum /srv/mapit/backups/postgres/mapit-*.dump \
  > /srv/mapit/backups/postgres/SHA256SUMS.latest
```

Use a dedicated backup role with sufficient read rights; record which Supabase auth/storage objects the provider backup covers. Check the dump with `pg_restore --list <file>` and then copy it to immutable/off-site storage.

### Neo4j

Neo4j Community 5.22 uses an offline dump. This causes a short maintenance window:

```bash
cd /srv/mapit/current/backend
install -d -m 0750 /srv/mapit/backups/neo4j
docker compose --env-file /etc/mapit/backend.env -p mapit stop neo4j
docker compose --env-file /etc/mapit/backend.env -p mapit run --rm --no-deps \
  -v /srv/mapit/backups/neo4j:/backups \
  neo4j neo4j-admin database dump neo4j \
  --to-path=/backups --overwrite-destination=true
docker compose --env-file /etc/mapit/backend.env -p mapit up -d neo4j
sha256sum /srv/mapit/backups/neo4j/neo4j.dump \
  > /srv/mapit/backups/neo4j/neo4j.dump.sha256
```

If `/backups` is not writable, first determine the container UID with `docker compose --env-file /etc/mapit/backend.env -p mapit run --rm --no-deps neo4j id` and give only that UID write permission. Do not arbitrarily change permissions on `neo4j_data`.

### Redis

Compose enables AOF. Additionally create a synchronous RDB snapshot with `redis-cli --rdb`:

```bash
cd /srv/mapit/current/backend
install -d -m 0750 /srv/mapit/backups/redis
docker compose --env-file /etc/mapit/backend.env -p mapit exec redis \
  redis-cli --rdb /tmp/mapit-redis.rdb
docker compose --env-file /etc/mapit/backend.env -p mapit cp \
  redis:/tmp/mapit-redis.rdb /srv/mapit/backups/redis/dump.rdb
docker compose --env-file /etc/mapit/backend.env -p mapit exec redis \
  rm /tmp/mapit-redis.rdb
sha256sum /srv/mapit/backups/redis/dump.rdb \
  > /srv/mapit/backups/redis/dump.rdb.sha256
```

Because Redis contains cache and temporary conversation state, starting empty is usually safer than restoring old state.

## 8. Restore and quarterly test

Never restore for the first time during an incident. Use an isolated restore environment with the same PostgreSQL/pgvector, Neo4j and Redis major versions.

### PostgreSQL restore test

```bash
sha256sum --check /srv/mapit/backups/postgres/SHA256SUMS.latest
pg_restore --no-owner --no-acl --exit-on-error \
  --dbname="$RESTORE_DATABASE_URL" /path/to/mapit-UTC.dump
```

Create the empty target project/database via the provider and then use `pg_restore --dbname="$RESTORE_DATABASE_URL"`. Check tables/counts, RLS/publications/functions, vector dimensions, login and V3/V4 smoke with the queries from [DATABASE_SCHEMA](MAP-IT_SA/docs/DATABASE_SCHEMA.md). For managed Supabase auth/storage, a provider restore may be needed in addition to a public-schema dump.

### Neo4j restore test

```bash
cd /srv/mapit/restore/backend
sha256sum --check /srv/mapit/backups/neo4j/neo4j.dump.sha256
docker compose --env-file /etc/mapit/backend.env -p mapit-restore stop neo4j
docker compose --env-file /etc/mapit/backend.env -p mapit-restore run --rm --no-deps \
  -v /srv/mapit/backups/neo4j:/backups \
  neo4j neo4j-admin database load neo4j \
  --from-path=/backups --overwrite-destination=true
docker compose --env-file /etc/mapit/backend.env -p mapit-restore up -d neo4j
```

Run this on a separate restore host, or override the host ports so they do not conflict with production. Verify connection, node/relationship counts and a representative graph query. If the graph is demonstrably fully derivable, re-ingest from verified raw artifacts is an alternative; some artifacts are currently missing, so do not rely on that without a manifest.

### Redis restore test

Preferred recovery is an **empty Redis**: stop the API, make a copy of the existing volume, start Redis empty and accept cache/session loss. Test an RDB in isolation first:

```bash
sha256sum --check /srv/mapit/backups/redis/dump.rdb.sha256
docker volume create mapit_redis_restore
docker run --rm --user 0 \
  -v mapit_redis_restore:/data \
  -v /srv/mapit/backups/redis:/backup:ro \
  redis:7-alpine sh -c \
  'cp /backup/dump.rdb /data/dump.rdb && chown redis:redis /data/dump.rdb'
docker run -d --name mapit-redis-restore \
  -p 127.0.0.1:26379:6379 \
  -v mapit_redis_restore:/data \
  redis:7-alpine redis-server --appendonly yes
redis-cli -p 26379 ping
redis-cli -p 26379 dbsize
```

Check representative keys and TTLs and then remove the temporary container/volume according to the local change process. Only if temporary state genuinely must be restored: shut down FastAPI and production Redis, make a second copy of `mapit_redis_data`, empty only that validated volume and place the checked `dump.rdb` as above. This is a destructive production action and requires explicit change approval; never overwrite the active volume during normal traffic.

### Restore acceptance

- backup checksum valid and artifact decryptable;
- restore command exit 0 and no unexpected warnings;
- schema/counts/vector dimensions conform to the database guide;
- Supabase login, RLS negative test and Realtime work;
- Neo4j counts/representative relationships are correct;
- V3 chat, persistence and V4 session smoke pass;
- measured RPO/RTO, operator and deviations in the restore log;
- production backup remains untouched.

## 9. Scheduled jobs and data freshness

### Actual current state

There is **no working scheduler configuration** in the repository. Celery worker/beat refer to a non-existent `app.services.celery_app`. The following functions are manual or on-demand only:

| Function | Current trigger | Current freshness guarantee |
|---|---|---|
| Funding-call detail enrichment | On demand by service/tool; cache TTL default 3600 s | Cache can refresh after an hour, but this is not a full call sync. |
| `sync_topics_from_projects.py` | Manual script | No schema/cadence recorded. |
| Document and programme ingest | Manual scripts | No automatic freshness SLA. |
| Project/chunk/call embeddings | Manual generate/re-embed scripts | No automatic drift or freshness guarantee. |
| Saved-search alerts | Manual `POST /api/alerts/process-alerts`; intended for a non-existent Celery task | No reliable processing; email integration has a TODO. |
| Analytics flush | During requests, buffer of 50 events or approximately 60 s | Can lose data on process crash. |

Do not publish `process-alerts` as an unauthenticated cron URL. First add service auth, idempotency/locking, timeout, retries, metrics and audit logging.

### Proposed cadence after implementation

| Job | Cadence | Freshness SLA |
|---|---|---|
| Open/upcoming funding calls sync | Daily 02:00 UTC; extra run after source release | <26 hours |
| Deadline/status normalization | Every 6 hours | <8 hours |
| Document/programme guide ingest | Weekly and event-driven on new guides | <8 days, critical rule change <48 hours |
| Incremental embeddings | After every successful ingest, otherwise daily | <30 hours after source data change |
| Neo4j derived graph rebuild/increment | After project batch, at minimum weekly | <8 days |
| Saved-search daily/weekly | According to user frequency | daily <26 hours; weekly <8 days |
| Quality scenario sample | Nightly on staging; full for release | Result <24 hours / release-bound |

Every job writes a run record with code SHA, input version/checksum, start/end, inserted/updated/skipped/failed counts and next retry. No overlap: use a distributed lock and a unique run ID.

## 10. Capacity and limits

| Component | Configured/known limit | Operational action |
|---|---|---|
| PostgreSQL pool | Backend default 5–50; analytics extra pools up to 5 per process | Count across all Uvicorn workers; alert for the provider limit. |
| Supabase REST | `max_rows=1000` locally | Paginate; no full exports via a single REST call. |
| Neo4j | Compose: heap 512 MiB–2 GiB, page cache 512 MiB, single Community node | Monitor heap/page cache/disk; no HA. |
| Redis | Single `redis:7-alpine`, AOF, no auth/TLS on loopback | Bind internally/firewall only; monitor memory/evictions. |
| API | VPS runbook starts 2 Uvicorn workers; no proven load baseline | Load test staging before scaling; share state via Redis. |
| SSE/NGINX | Proxy read/send timeout proposed 600 s | Alert on hanging streams; test disconnect/cancel. |
| Upload | Frontend 10 MiB, backend 50 MiB, NGINX runbook 10 MiB | 10 MiB is the effective supported limit. |
| Vector | Target 3072; snapshot partly 768; no ANN index for >2000 in migration `002` | Keep model/schema/data aligned; monitor sequential scan latency. |
| Rate limit | App-global off; runbook proposes NGINX 20 r/s per IP | No per-user/cost-aware/distributed protection; still a security blocker. |
| Provider quotas | Not recorded in the repo | Record quotas/budgets per provider and alert on 429/costs. |

## 11. Incident response

| Level | Definition | Initial response | Escalation |
|---|---|---:|---|
| P0 | Confirmed secret/service-role exposure, cross-tenant data leak, destructive data loss or active compromise | 15 min | Incident commander + security + data owner immediately; authorities/customers per policy. |
| P1 | Full outage, login/database unavailable, broadly incorrect answers or RPO/RTO at risk | 15 min | On-call → service owner after 15 min; management after 30 min. |
| P2 | Degradation with workaround, one datastore/provider/tool down, data too old | 1 hour | Service owner the same working day. |
| P3 | Small bug/maintenance issue without direct user impact | Next working day | Backlog with owner/target. |

Contact names, phone, channel, DPO/security and management are **TBD** and must be recorded in the on-call register outside this public document before production.

Standard procedure:

1. Acknowledge, designate incident commander/scribe and record the UTC time.
2. Classify user, data, security and compliance impact; preserve evidence.
3. Limit damage: stop traffic, disable a feature, revoke provider/key or activate the previous release.
4. Diagnose read-only with health, logs, release SHA, schema/data/provider status.
5. Recover via known rollback/restore; avoid ad-hoc production data edits.
6. Validate with two users, an RLS negative test and core smokes.
7. Communicate status/cadence; security/privacy via a formal process.
8. Close only after monitoring stability; postmortem within 5 working days for P0/P1.

## 12. Disaster recovery

Proposed targets:

| Scenario | RPO | RTO |
|---|---:|---:|
| PostgreSQL/Supabase with PITR | 1 hour | 4 hours |
| Without PITR | 24 hours | 8 hours |
| Neo4j | 24 hours | 8 hours; faster if reliable re-ingest is possible |
| Redis | Best effort / 24 hours | 1 hour; cache/session loss accepted |
| Complete VPS region/account failure | According to the slowest primary store | 8 hours after a new VPS/DNS is available |

DR sequence:

1. Declare P1/P0, freeze writes and determine the last consistent recovery point.
2. Provision a new VPS/account according to the deployment runbook; recover secrets via the secret store and rotate on compromise.
3. Restore Supabase/PostgreSQL first; verify auth, schema, RLS, publications and counts.
4. Restore Neo4j from the same or a later derived snapshot; otherwise controlled re-ingest.
5. Start an empty Redis or restore only verified temporary state.
6. Deploy exactly compatible backend/frontend SHAs and environment version.
7. Run health, login, tenant negative, V3/V4 and data representativeness checks.
8. Switch DNS/traffic with reduced TTL; monitor intensively.
9. Document actual RPO/RTO and reconcile writes lost after the recovery point.

Conduct a tabletop exercise quarterly and a technical restore/regional rebuild at least semi-annually. A DR target only counts as proven after another operator has executed the runbook within RTO.
