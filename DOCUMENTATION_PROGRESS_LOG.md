# Progress log — Wave 1 P0 documentation remediation

**Date:** 20 July 2026  
**Scope:** documentation only in `NEWMAPIT`, `MAP-IT_SA` and `webapp-ui`  
**Final status:** all requested P0 documentation delivered; technical/security release blockers deliberately not resolved with code

## Delivered canonical documents

| Document | Action | Status | Most important verification |
|---|---|---|---|
| `START_HERE.md` | New | ready | Repo relationship, prerequisites, Supabase/Compose network, host/internal ports, venv, start order and smoke contract checked against config/scripts. |
| `KNOWN_ISSUES_AND_RISKS.md` | New | ready | Final 54.8% and earlier 61.0% quality snapshots, release blockers, security, limits, debt and doc contradictions centralised. |
| `MAP-IT_SA/docs_tracked/DEVELOPER_SETUP.md` | Fully rewritten | ready | `venv`, all Make targets, Supabase network, real V3/V4 prefixes and Neo4j permissions checked. |
| `MAP-IT_SA/docs/guides/DATA_INGESTION.md` | Fully rewritten | ready | All present ingest scripts/flags/default paths, missing artifacts, local counts, idempotency and 768→3072 transition checked. |
| `MAP-IT_SA/docs/DATABASE_SCHEMA.md` | Substantially improved | ready | Read-only catalogue snapshot: 35 public tables, 15 views, 17 functions, RLS/publications, counts, ERD, Neo4j and non-linear migration ledger. |
| `API_CONTRACT.md` | New | ready | All 9 V3 and 12 V4 routes, request/response models, real SSE emissions, proxies, identity and other core routes inventoried. |
| `SECURITY_REGISTER.md` | New | ready | Threat model, trust boundaries, 16 code-localised issues, secrets/cookies/CORS/CSRF/RLS and deployment checklist. |
| `DEPLOYMENT_RUNBOOK.md` | New | ready | One VPS topology chosen; K8s/Compose limitations, build gates, env, NGINX, deploy, health and rollback worked out. |
| `OPERATIONS_RUNBOOK.md` | New | ready | Actual health/log/analytics coverage plus proposed alerts/SLOs, backups/restores, scheduling/freshness, incident response and DR. |

## Archived documents

All files below received only a Dutch archive banner with date, working relative link to `START_HERE.md` and a specific reason:

| Document | Reason |
|---|---|
| `MAP-IT_SA/docs_tracked/README_CELERY.md` | Described Celery stack/entrypoint/Compose are missing. |
| `MAP-IT_SA/docs_tracked/FRONTEND_STREAMING_V3_SPEC.md` | Removed V2 presented as current and active V3 as pending. |
| `MAP-IT_SA/docs/prd/MAPIT_STABILITY_PRD.md` | Older, unticked duplicate PRD. |
| `MAP-IT_SA/docs/prd/MAPIT_PRODUCTION_QUALITY_VALIDATION_PRD.md` | Succeeded by V3 and current quality reports. |
| `webapp-ui/docs/CONVERSATIONAL_AI_ARCHITECTURE.md` | Pre-V3 proposal with non-current services/routes. |
| `webapp-ui/docs/CONVERSATION_IMPLEMENTATION_GUIDE.md` | Refers to non-existent backend files and old routes. |
| `webapp-ui/SESSION_SUMMARY.md` | Time-bound incident/deployment session. |
| `webapp-ui/docs/project_setup.md` | Destructive manual migration and outdated VPS steps. |

## Documentation validation performed

- `DOCUMENTATION_REVIEW.md` fully reread.
- Claims tested against Makefile, setup script, Compose, Dockerfile, package/lockfile, Supabase config, env examples and frontend proxies.
- Routes, Pydantic models, SSE producers, error handlers, auth, cookies, rate limiting and CORS checked against source code.
- PostgreSQL and Neo4j catalogue/counts checked read-only; schema files, Drizzle and migrations compared.
- Ingest scripts and `quality_gate.py --help` checked against documented flags.
- Compose config and Make targets checked read-only/dry-run.
- All local Markdown links in the 9 canonical docs and 8 banners exist.
- Code fences in all canonical docs are balanced.
- Archive banners are at the top of all 8 requested files.
- `git diff --check` was run on the documentation changes.

## Deliberately left open

This wave was not allowed to change code or infrastructure. As a result, among others remain open: backend auth/identity, RLS/membership checks, secret/cookie fixes, rate limiting, a linear migration chain, data artifact distribution, test/check errors, scheduler, deployment config-as-code and actual backup/restore/DR drills. The impact and required acceptance are recorded in `KNOWN_ISSUES_AND_RISKS.md` and `SECURITY_REGISTER.md`.

The backend worktree already contained local non-documentation changes and untracked dumps/scripts/tests before this wave. Those were preserved and not modified.
