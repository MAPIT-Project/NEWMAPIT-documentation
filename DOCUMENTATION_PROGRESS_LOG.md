# Progress log — Wave 1 P0-documentatieremediatie

**Datum:** 20 juli 2026  
**Scope:** uitsluitend documentatie in `NEWMAPIT`, `MAP-IT_SA` en `webapp-ui`  
**Eindstatus:** alle gevraagde P0-documentatie opgeleverd; technische/security-releaseblockers bewust niet met code opgelost

## Opgeleverde canonical documenten

| Document | Actie | Status | Belangrijkste verificatie |
|---|---|---|---|
| `START_HERE.md` | Nieuw | gereed | Repo-relatie, prerequisites, Supabase-/Compose-netwerk, host/interne poorten, venv, startvolgorde en smokecontract tegen config/scripts gecontroleerd. |
| `KNOWN_ISSUES_AND_RISKS.md` | Nieuw | gereed | Finale 54,8%- en eerdere 61,0%-quality snapshots, releaseblockers, security, limits, schuld en doccontradicties gecentraliseerd. |
| `MAP-IT_SA/docs_tracked/DEVELOPER_SETUP.md` | Volledig herschreven | gereed | `venv`, alle Make-targets, Supabase-network, echte V3/V4-prefixen en Neo4j-permissions gecontroleerd. |
| `MAP-IT_SA/docs/guides/DATA_INGESTION.md` | Volledig herschreven | gereed | Alle aanwezige ingestscripts/flags/defaultpaden, ontbrekende artifacts, lokale counts, idempotentie en 768→3072-transitie gecontroleerd. |
| `MAP-IT_SA/docs/DATABASE_SCHEMA.md` | Ingrijpend verbeterd | gereed | Read-only catalogussnapshot: 35 public-tabellen, 15 views, 17 functies, RLS/publications, counts, ERD, Neo4j en niet-lineaire migration ledger. |
| `API_CONTRACT.md` | Nieuw | gereed | Alle 9 V3- en 12 V4-routes, request/responsemodellen, echte SSE-emissies, proxies, identity en overige kernroutes geïnventariseerd. |
| `SECURITY_REGISTER.md` | Nieuw | gereed | Threat model, trust boundaries, 16 code-gelokaliseerde issues, secrets/cookies/CORS/CSRF/RLS en deploymentchecklist. |
| `DEPLOYMENT_RUNBOOK.md` | Nieuw | gereed | Eén VPS-topologie gekozen; K8s/Compose-beperkingen, buildgates, env, NGINX, deploy, health en rollback uitgewerkt. |
| `OPERATIONS_RUNBOOK.md` | Nieuw | gereed | Werkelijke health/log/analyticsdekking plus voorgestelde alerts/SLO's, backups/restores, scheduling/freshness, incidentresponse en DR. |

## Gearchiveerde documenten

Alle onderstaande bestanden kregen alleen een Nederlandstalige archiveerbanner met datum, werkende relatieve link naar `START_HERE.md` en een specifieke reden:

| Document | Reden |
|---|---|
| `MAP-IT_SA/docs_tracked/README_CELERY.md` | Beschreven Celery-stack/entrypoint/Compose ontbreken. |
| `MAP-IT_SA/docs_tracked/FRONTEND_STREAMING_V3_SPEC.md` | Verwijderde V2 als current en actieve V3 als pending. |
| `MAP-IT_SA/docs/prd/MAPIT_STABILITY_PRD.md` | Oudere, onafgevinkte duplicaat-PRD. |
| `MAP-IT_SA/docs/prd/MAPIT_PRODUCTION_QUALITY_VALIDATION_PRD.md` | Opgevolgd door V3 en actuele quality reports. |
| `webapp-ui/docs/CONVERSATIONAL_AI_ARCHITECTURE.md` | Pre-V3-voorstel met niet-actuele services/routes. |
| `webapp-ui/docs/CONVERSATION_IMPLEMENTATION_GUIDE.md` | Verwijst naar niet-bestaande backendbestanden en oude routes. |
| `webapp-ui/SESSION_SUMMARY.md` | Tijdgebonden incident-/deploymentsessie. |
| `webapp-ui/docs/project_setup.md` | Destructieve handmatige migratie- en verouderde VPS-stappen. |

## Uitgevoerde documentatievalidatie

- `DOCUMENTATION_REVIEW.md` volledig herlezen.
- Claims getoetst aan Makefile, setupscript, Compose, Dockerfile, package/lockfile, Supabase-config, envvoorbeelden en frontendproxies.
- Routes, Pydantic-modellen, SSE-producers, errorhandlers, auth, cookies, rate limiting en CORS tegen broncode gecontroleerd.
- PostgreSQL- en Neo4j-catalogus/counts read-only gecontroleerd; schemafiles, Drizzle en migrations vergeleken.
- Ingestscripts en `quality_gate.py --help` tegen gedocumenteerde flags gecontroleerd.
- Composeconfig en Make-targets read-only/dry-run gecontroleerd.
- Alle lokale Markdownlinks in de 9 canonical docs en 8 banners bestaan.
- Code-fences in alle canonical docs zijn gebalanceerd.
- Archiveerbanners staan bovenaan alle 8 gevraagde bestanden.
- `git diff --check` is op de documentatiewijzigingen uitgevoerd.

## Bewust open gebleven

Deze wave mocht geen code of infrastructuur wijzigen. Daardoor blijven onder meer open: backend-auth/identity, RLS/membershipchecks, secret-/cookiefixes, rate limiting, een lineaire migration chain, data-artifactdistributie, test/check-fouten, scheduler, deploymentconfig-as-code en daadwerkelijke backup/restore-/DR-drills. De impact en vereiste acceptatie zijn vastgelegd in `KNOWN_ISSUES_AND_RISKS.md` en `SECURITY_REGISTER.md`.

De backendworktree bevatte vóór deze wave al lokale niet-documentaire wijzigingen en niet-getrackte dumps/scripts/tests. Die zijn behouden en niet aangepast.
