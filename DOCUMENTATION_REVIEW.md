# MAPIT Documentation Review & Handover Readiness Report

**Datum:** 20 juli 2026  
**Reviewer:** Codex (AI code analysis)  
**Project:** MAPIT NEWMAPIT (backend + frontend)

## Executive Summary

MAPIT is **niet klaar voor een veilige, zelfstandige overdracht op basis van de huidige documentatie**. Er is veel documentatie (47 documenten, circa 880 KiB en 22.974 regels), maar volume maskeert een gebrek aan een betrouwbare route door het materiaal. De beste documenten beschrijven de V3/V4-architectuur, 42 tools en proposal flows redelijk helder; daartegenover staan verouderde V2- en Celery-specificaties, PRD's die tegelijk als planning en als bewijs van oplevering worden gebruikt, foutieve setup- en testcommando's, meerdere niet-bestaande bestanden en een ontbrekend end-to-end contract tussen Supabase-authenticatie, SvelteKit-proxies en FastAPI. Een verse clone is niet reproduceerbaar op te starten met alleen de docs: de genoemde PostgreSQL-dump is niet in Git opgenomen, de CORDIS-ingest verwacht een ontbrekende `data/cordis`-map, de gedocumenteerde verificatieflags bestaan niet, `make test` faalt al bij testcollectie en de frontend heeft geen testscript en faalt op 64 type-/Svelte-fouten. Ook ontbreken een actueel securitymodel, operationeel runbook, backup/restore-procedure, releasehistorie en één expliciete eigenaar per systeemonderdeel. De documentatie-readiness is daarom **37/100**; vóór overdracht zijn de P0-acties hieronder nodig.

### Onderzoeksbasis en scope

- Alle 47 Markdown-bestanden uit de expliciet opgegeven root-, `docs/`, `docs/guides/`, `docs/prd/`- en `docs_tracked/`-locaties zijn geïnventariseerd en inhoudelijk beoordeeld.
- Claims zijn getoetst aan de huidige code, routes, `package.json`, `Makefile`, `.env.example`-bestanden, Docker/Kubernetes-configuratie, SQL-schema's, toolregistratie, scripts en tests.
- Laatste wijzigingsdatum komt uit `git log -1 -- <pad>`; `geen git-historie` betekent dat het bestand niet door Git wordt gevolgd of nooit is gecommit.
- Buiten de opgegeven inventaris zijn relevante, slecht vindbare docs aangetroffen, waaronder `MAP-IT_SA/k8s/README.md`, twee kwaliteitsrapporten onder `tests/quality/results/` en `webapp-ui/supabase_cli/how_to_run.md`. Deze zijn meegenomen bij de readiness-beoordeling, maar niet als extra inventarisrij.
- Beoordelingscode in de inventaris: **C/A/U/H/R** = completeness, accuracy, up-to-date, helderheid en redundantie; 1 = zeer zwak, 5 = sterk. Bij redundantie betekent 5 weinig ongewenste overlap.
- Dit rapport beoordeelt overdraagbaarheid van documentatie, niet de volledige functionele kwaliteit of security van de applicatie.

## Overdracht Readiness Score

| Gebied | Score | Rationale |
|---|---:|---|
| **Environment setup** | **2/5** | Er zijn backend- en frontendstappen, maar geen werkende full-stack “start here”. `DEVELOPER_SETUP.md` activeert `.venv` terwijl `setup.sh` `venv` maakt; `make docker-up` vereist een externe Supabase-network die niet eerst vanuit `webapp-ui/supabase_cli` wordt gestart; README start daarna nog een tweede API op poort 8000 terwijl Compose de API al op hostpoort 18000 start. |
| **Architecture understanding** | **3/5** | `SYSTEM_ARCHITECTURE.md`, `CLAUDE.md` en de stabiliteitsarchitectuur geven een bruikbaar V3/V4-beeld, maar de frontend, auth/trust boundaries, deployments en datastromen zijn niet in één actueel diagram verbonden. Oude V2-, 20-tool- en Celery-architecturen worden tegelijk als “single source of truth” aangeboden. |
| **Data ingestion** | **1/5** | Scripts en databronnen worden genoemd, maar de bootstrap is niet reproduceerbaar: `project_embeddings.dump` is Git-ignored, `ingest_cordis_to_neo4j.py` verwacht de niet-bestaande map `data/cordis`, en `quality_gate.py` ondersteunt de gedocumenteerde flags `--check-embeddings` en `--check-calls` niet. Provenance, versies, checksums, idempotentie en herstel ontbreken. |
| **API contract** | **2/5** | V3/V4-routes en veel SSE-events zijn beschreven, maar `FRONTEND_STREAMING_V3_SPEC.md` gebruikt verwijderde V2-routes en noemt V3 nog “pending”. `DEVELOPER_SETUP.md` noemt niet-bestaande `/api/v4/proposals/sessions`-routes. Het authenticatie- en user-ID-contract is niet beschreven; backendroutes gebruiken standaard `user_id=1` en passen de aanwezige auth dependency niet toe. |
| **Deployment** | **2/5** | Er is een VPS/NGINX/PM2-guide en er zijn Docker/Kubernetes-assets, maar de NGINX-config is alleen een pad op één server, er is geen frontend container/deploy manifest, rollback ontbreekt en K8s deployt Celery workers waarvoor geen `celery_app.py` bestaat. De frontend faalt bovendien op `npm run check`. |
| **Testing** | **2/5** | De backend heeft een grote testset, scenario's en test-PRD's, maar geen CI-workflow. `make test`/`pytest -q` faalt tijdens collectie door de ontoegankelijke `neo4j/import`-map; diverse gedocumenteerde testtargets bestaan niet. Frontend heeft geen `test`-script en `npm run check` geeft 64 fouten en 76 waarschuwingen. |
| **Operations** | **1/5** | Health endpoints en losse troubleshootingstappen bestaan, maar er is geen production runbook met dashboards, loglocaties, alerts, SLO's, on-call/escalatie, job scheduling, data freshness, backup/restore of disaster recovery. |
| **Security** | **1/5** | De READMEs waarschuwen voor een audit, maar documenteren geen threat model of trust boundaries. De code bevat een hardcoded Mapbox-token, een JWT fallback-secret, grotendeels ongebruikte backend-auth, uitgeschakelde globale rate limiting, ontbrekende membershipchecks en slechts beperkte RLS in het schema. `PUBLIC_SUPABASE_SERVICE_KEY` is bovendien misleidend als publieke variabele benoemd. |
| **Known issues/limits** | **1.5/5** | Issues staan verspreid over roadmaps, TODO's en PRD's en spreken elkaar tegen. De verborgen kwaliteitsrapporten noemen 54,8% respectievelijk 61% scenario-passrate en “NOT PRODUCTION READY”, terwijl README/roadmap onderdelen “production-grade/production-ready” noemen. Er is geen actuele, eigenaar- en datumgebonden issue-/risicoregister. |
| **Totaal** | **37/100** | Een ervaren oorspronkelijke ontwikkelaar kan met lokale voorkennis verder; een nieuw team kan niet veilig en reproduceerbaar overnemen zonder mondelinge overdracht en P0-remediatie. |

### Belangrijkste geverifieerde tegenstrijdigheden

| Documentclaim | Werkelijkheid in code/repository |
|---|---|
| `docs/README.md`: `docs_tracked/` is actueel en kent “no redundancy”. | De map bevat verwijderde V2-routes, een niet-bestaande Celery-stack, een 20-tool-specificatie en een vrijwel volledige kopie van de proposal integration guide in de frontend. |
| `DEVELOPER_SETUP.md`: `./scripts/setup.sh` gevolgd door `source .venv/bin/activate`. | `scripts/setup.sh` maakt en activeert `venv`, niet `.venv`, en installeert alleen `requirements.txt`, niet de dev-dependencies. |
| `DEVELOPER_SETUP.md`: `make run` start na Supabase de Compose-API. | `make run` voert lokaal `uvicorn ... --port 8000` uit; `make docker-up` start Compose en exposeert API op 18000. |
| `DATA_INGESTION.md`: de pre-packaged PostgreSQL-dump kan worden hersteld. | `neo4j_backup/project_embeddings.dump` bestaat lokaal maar is door `/neo4j_backup/` genegeerd en niet door Git gevolgd; een nieuwe clone krijgt hem niet. |
| `DATA_INGESTION.md`: `python scripts/ingest_cordis_to_neo4j.py` is voldoende. | Het script gebruikt standaard `data/cordis`; `/data/` is Git-ignored en die map ontbreekt. Het script leest niet uit de zojuist herstelde PostgreSQL-database. |
| `DATA_INGESTION.md`: verificatie via `quality_gate.py --check-embeddings/--check-calls`. | `quality_gate.py` accepteert alleen `--results-dir`, `--quality-baseline`, `--threshold` en `--fail-on-missing`. |
| `TOOL_REFERENCE.md`: uitgebreide referentie van 42 tools. | `tool_registry.py` registreert inderdaad 42 tools, maar de guide documenteert slechts een subset en mist o.a. `compare_programmes`, `search_documents`, `get_programme_info`, `quick_quality_check`, `show_cached_projects` en diverse templates. |
| `FRONTEND_STREAMING_V3_SPEC.md`: V3-route moet nog worden toegevoegd en V2 is current. | Alleen V3/V4 is gemount; `conversation_routes_v2.py` bestaat niet. De V3-route is actief in `app/api/conversation_routes_v3.py`. |
| `README_CELERY.md`: FastAPI routeert naar Celery en `docker-compose.celery.yml` start workers. | Geen `docker-compose.celery.yml`, `celery_app.py`, Celery tasks of gemounte `/api/query`-backendroute. |
| `RATE_LIMITING.md`: globale 100/min rate limit geldt voor alle `/api/*`. | De globale middleware in `src/hooks.server.ts` is volledig uitgecommentarieerd met TODO om hem later opnieuw in te schakelen. |
| `CRUSH.md`: `npm run test`. | `package.json` bevat geen `test`-script. |
| README/roadmap: productiegericht/production-ready. | `tests/quality/results/FINAL_VERIFICATION_SUMMARY.md` meldt 23/42 (54,8%) scenario's geslaagd en expliciet “NOT PRODUCTION READY”; dit resultaat is niet in de hoofdindex opgenomen. |

## Documentatie Inventarisatie

### Backend — root en actieve guides

| Document | Grootte | Git-datum | Samenvatting | C/A/U/H/R |
|---|---:|---:|---|---:|
| `MAP-IT_SA/README.md` | 5.3 KiB | 2026-05-20 | Productintroductie, quickstart, kernarchitectuur, doclinks en expliciete single-dev/securitywaarschuwing. | 3/2/3/4/3 |
| `MAP-IT_SA/CLAUDE.md` | 53.3 KiB | 2026-05-20 | Zeer brede developer- en AI-agentguide met architectuur, 42 tools, dataflows, commando's, tests en troubleshooting. | 4/2/3/3/2 |
| `MAP-IT_SA/GEMINI.md` | 3.6 KiB | geen git-historie | Korte, grotendeels duplicerende AI-agentinstructie voor V3/V4, tools en ontwikkelcommando's. | 2/3/3/4/1 |
| `MAP-IT_SA/questions.md` | 0.9 KiB | 2026-06-09 | Vijf handmatige demo-/acceptatievragen voor complexe Horizon-analyses. | 1/4/4/3/3 |
| `MAP-IT_SA/docs/README.md` | 4.4 KiB | 2026-05-13 | Documentatie-index die `docs_tracked/` als single source of truth en PRD's als actief positioneert. | 3/1/1/4/1 |
| `MAP-IT_SA/docs/DATABASE_SCHEMA.md` | 7.0 KiB | 2026-05-20 | Snapshot van 35 public tables, 15 views, vector/RPC-logica, Neo4j-model en recordaantallen. | 3/3/2/4/3 |
| `MAP-IT_SA/docs/guides/DATA_INGESTION.md` | 4.2 KiB | 2026-05-20 | Beknopte restore- en ingeststappen voor CORDIS, documenten, calls, embeddings en Neo4j. | 2/1/2/4/4 |
| `MAP-IT_SA/docs/guides/ROADMAP.md` | 3.0 KiB | 2026-05-18 | Korte featurestatus en technische schuld voor V3, multi-programme, V4 en data-intelligence. | 3/2/2/4/2 |
| `MAP-IT_SA/docs/guides/TOOL_REFERENCE.md` | 4.7 KiB | 2026-05-18 | Categorische tabellen met een selectie van agenttools en een korte uitleg van hybrid search. | 2/3/3/4/3 |

### Backend — PRD's

| Document | Grootte | Git-datum | Samenvatting | C/A/U/H/R |
|---|---:|---:|---|---:|
| `MAP-IT_SA/docs/prd/MAPIT_ADVANCED_CONVERSATION_FLOWS_PRD.md` | 43.5 KiB | 2026-01-22 | Groot, volledig afgevinkt scenario- en implementatieplan voor complexe multi-turn en multi-programme flows. | 4/3/2/3/2 |
| `MAP-IT_SA/docs/prd/MAPIT_APPLICATION_QUALITY_IMPROVEMENT_PRD.md` | 55.8 KiB | 2026-01-27 | Volledig afgevinkt kwaliteitsplan voor tools, programma's, performance, resilience en dataversheid. | 4/1/1/3/2 |
| `MAP-IT_SA/docs/prd/MAPIT_CONVERSATION_STABILITY_PRD.md` | 19.5 KiB | 2026-01-22 | Afgevinkt implementatieplan voor locks, atomic storage, scenarioagents en regressietests. | 4/3/2/4/1 |
| `MAP-IT_SA/docs/prd/MAPIT_MULTI_PROGRAMME_EXCELLENCE_PRD.md` | 26.1 KiB | 2026-01-27 | Onuitgevoerde testmatrix om gelijkwaardige kwaliteit voor zes EU-programma's te bewijzen. | 4/2/2/4/2 |
| `MAP-IT_SA/docs/prd/MAPIT_PRODUCTION_QUALITY_VALIDATION_PRD.md` | 31.3 KiB | 2026-01-22 | Oudere, gemengd afgevinkte productievalidatie voor 42 geavanceerde scenario's. | 3/2/1/3/1 |
| `MAP-IT_SA/docs/prd/MAPIT_PRODUCTION_QUALITY_VALIDATION_PRD_V3.md` | 63.7 KiB | 2026-01-23 | Application-first validatieplan met fixes, scenarioresultaten en finale readinesscheck. | 4/2/2/3/2 |
| `MAP-IT_SA/docs/prd/MAPIT_QUALITY_PRD.md` | 32.9 KiB | 2026-01-21 | Afgevinkte LLM-as-judge evaluatie van 36 fundingvragen met scores en fixinstructies. | 4/3/2/3/2 |
| `MAP-IT_SA/docs/prd/MAPIT_STABILITY_PRD.md` | 17.5 KiB | 2026-01-27 | Oudere, volledig onafgevinkte versie van vrijwel hetzelfde conversation-stabilityplan. | 3/2/1/3/1 |
| `MAP-IT_SA/docs/prd/MAPIT_V3_RESPONSE_FORMATTER_PRD.md` | 27.8 KiB | 2026-01-27 | Grotendeels openstaand formatterplan dat nog uitgaat van 34 tools en niet-bestaande tests noemt. | 4/2/1/4/2 |

### Backend — `docs_tracked`

| Document | Grootte | Git-datum | Samenvatting | C/A/U/H/R |
|---|---:|---:|---|---:|
| `MAP-IT_SA/docs_tracked/CONVERSATION_STABILITY_ARCHITECTURE.md` | 33.1 KiB | 2026-05-13 | Gedetailleerd ontwerp en debugguide voor locks, atomic storage, scenarioformaten en testagents. | 4/3/3/4/2 |
| `MAP-IT_SA/docs_tracked/DEVELOPER_SETUP.md` | 6.5 KiB | 2026-05-19 | Lokale backend-/database-/Neo4j-setup en V3/V4-sequentiediagrammen. | 3/1/2/4/3 |
| `MAP-IT_SA/docs_tracked/ENHANCED_HORIZON_QUESTIONS_CAPABILITIES.md` | 4.4 KiB | 2025-12-03 | Claimdocument voor geavanceerde callfiltering, externe API-integratie en performancecijfers. | 2/2/1/4/3 |
| `MAP-IT_SA/docs_tracked/EU_PROGRAMME_DATA_DOWNLOADS.md` | 11.0 KiB | 2026-05-13 | Downloadlinks, voorgestelde mapstructuur en checklist voor EU-programma- en projectdata. | 3/3/2/4/3 |
| `MAP-IT_SA/docs_tracked/FRONTEND_INTEGRATION_PROMPT.md` | 33.2 KiB | 2025-12-03 | Frontendbouwprompt voor SSE, feedback, toolweergave, budget/timelinecards en foutafhandeling. | 4/2/1/3/1 |
| `MAP-IT_SA/docs_tracked/FRONTEND_STREAMING_V3_SPEC.md` | 18.3 KiB | 2025-12-03 | Historische transitiespecificatie van V2 naar V3 inclusief SSE-events en JS-voorbeeldclient. | 4/1/1/4/1 |
| `MAP-IT_SA/docs_tracked/IMPROVEMENT_ROADMAP.md` | 69.4 KiB | 2025-12-19 | Omvangrijke implementatieroadmap voor reliability, performance, features, security en K8s. | 5/2/1/2/1 |
| `MAP-IT_SA/docs_tracked/MANUAL_TEST_SCENARIOS.md` | 9.2 KiB | 2025-12-10 | Handmatige querycatalogus, smoke test, formatchecks en verouderde data-qualitycommando's. | 3/2/2/4/2 |
| `MAP-IT_SA/docs_tracked/PROPOSAL_WRITER_FRONTEND_INTEGRATION.md` | 45.3 KiB | 2025-12-10 | Uitgebreide V4 REST/SSE-integratieguide met types en voornamelijk React-voorbeelden. | 4/3/2/3/1 |
| `MAP-IT_SA/docs_tracked/QUESTION_CATEGORIES.md` | 34.0 KiB | 2025-12-09 | Zeer brede capability- en scenariocatalogus inclusief legacy API's, 20 tools en toekomstige exports. | 4/2/1/3/1 |
| `MAP-IT_SA/docs_tracked/README_CELERY.md` | 7.2 KiB | 2026-05-13 | Setup- en operationsguide voor een Celery/API-stack die niet in de repository bestaat. | 3/1/1/4/2 |
| `MAP-IT_SA/docs_tracked/SYSTEM_ARCHITECTURE.md` | 18.4 KiB | 2026-05-20 | Hoofdarchitectuur voor V3/V4, 42 tools, data stores, services, configuratie en voorbeeldflows. | 4/3/3/4/2 |
| `MAP-IT_SA/docs_tracked/example_questions.md` | 21.9 KiB | 2025-12-03 | Grote lijst voorbeeldvragen over funding, voorstellen, compliance, operations en scaling. | 2/4/2/3/2 |

### Frontend — root en `docs`

| Document | Grootte | Git-datum | Samenvatting | C/A/U/H/R |
|---|---:|---:|---|---:|
| `webapp-ui/README.md` | 2.1 KiB | 2026-05-28 | Minimale productintroductie met scripts, tech stack en single-dev/securitywaarschuwing. | 2/3/3/4/3 |
| `webapp-ui/CRUSH.md` | 1.3 KiB | 2025-08-01 | AI-agentconventies voor build, lint, tests, database en stijl. | 2/2/1/4/2 |
| `webapp-ui/DEPLOYMENT_GUIDE.md` | 11.2 KiB | 2026-05-28 | VPS/PM2/NGINX/Supabase/backend deployment- en troubleshootingguide voor `mapit.work`. | 3/2/3/4/2 |
| `webapp-ui/SESSION_SUMMARY.md` | 6.2 KiB | 2026-05-28 | Historisch verslag van één deployment-/authfixsessie met gemaakte keuzes en vervolgacties. | 2/2/2/4/1 |
| `webapp-ui/TROUBLESHOOTING.md` | 5.4 KiB | 2026-05-28 | Snelle productiecommando's voor login, mixed content, backend, PM2, Supabase en NGINX. | 3/2/3/4/2 |
| `webapp-ui/AI_Proposal_Writer_Frontend_Integration_Guide.md` | 45.3 KiB | 2025-12-10 | Vrijwel duplicaat van de backend V4 proposal integration guide met REST/SSE en React-code. | 4/3/2/3/1 |
| `webapp-ui/proposal_editor_report.md` | 11.9 KiB | 2025-12-10 | Momentopname van proposal-editor UI, stores, types, gebruikte/gewenste endpoints en aanbevelingen. | 3/2/2/4/2 |
| `webapp-ui/docs/CITATION_IMPLEMENTATION_VERIFICATION.md` | 8.3 KiB | 2025-10-09 | Onuitgevoerde verificatiechecklist en voorgestelde unit/integratie/performance-tests voor citations. | 3/2/1/4/3 |
| `webapp-ui/docs/CONVERSATIONAL_AI_ARCHITECTURE.md` | 17.7 KiB | 2025-09-06 | Pre-V3 voorgestelde conversation-contextarchitectuur met Python-pseudocode en open fasen. | 4/1/1/3/1 |
| `webapp-ui/docs/CONVERSATION_IMPLEMENTATION_GUIDE.md` | 29.6 KiB | 2025-09-06 | Grote implementatiehandleiding voor inmiddels niet-bestaande backendservices en routes. | 4/1/1/2/1 |
| `webapp-ui/docs/PROPOSAL_EDITOR_IMPLEMENTATION.md` | 8.3 KiB | 2025-10-13 | Featurebeschrijving en usageguide voor de live proposal editor, inclusief mocks en future work. | 3/2/2/4/2 |
| `webapp-ui/docs/RATE_LIMITING.md` | 7.4 KiB | 2025-08-01 | Ontwerp en gebruik van in-memory IP-rate limiting met geclaimde globale en route-specifieke tiers. | 4/2/1/4/3 |
| `webapp-ui/docs/SUPABASE_LOCAL_REALTIME_SETUP.md` | 2.8 KiB | 2025-08-01 | Lokale SQL- en Supabase-stappen om `message` aan realtime publication toe te voegen. | 3/3/2/4/2 |
| `webapp-ui/docs/SUPABASE_REALTIME_SETUP.md` | 3.3 KiB | 2025-08-01 | Productie-instructies en voorbeeld-RLS voor collaboration chat via Supabase Realtime. | 3/2/1/4/2 |
| `webapp-ui/docs/project_docs.md` | 0.6 KiB | 2025-08-01 | WIP-stub met alleen tech stack en timestampconventie. | 1/3/1/3/2 |
| `webapp-ui/docs/project_setup.md` | 2.3 KiB | 2025-08-01 | Oud, handmatig VPS-demo-installatiepad met SSH-forwarding, Supabase Studio en PM2. | 2/1/1/3/1 |

## Kritieke Gaten (P0 — moet opgelost voor overdracht)

1. **Geen full-stack handover/onboardingdocument.**
   - **Waarom kritiek:** een nieuwe developer moet drie verschillende setupverhalen combineren en zelf poorten, startvolgorde, repo-relaties en database-eigenaarschap afleiden.
   - **Actie:** maak `START_HERE.md` in de NEWMAPIT-root met systeemdoel, repo's, owners, port matrix, minimale/full-data mode, exacte start-/stopvolgorde, smoke test, links naar canonical docs en een “first day” checklist.

2. **Een verse clone kan de data niet reproduceren.**
   - **Waarom kritiek:** `project_embeddings.dump` is niet tracked; `data/cordis` ontbreekt; de docs noemen geen artifact store, versie, checksum of downloadprocedure. Zonder data werken de kernfeatures niet representatief.
   - **Actie:** kies en documenteer één ondersteund bootstrap-pad: versieerde dump in beveiligde artifact storage óf volledig reproduceerbare raw-data-ingest. Voeg manifest, SHA-256, schema-/embeddingversie, omvang, licentie, eigenaar, restorecommando's en verificatiequeries toe.

3. **Setupcommando's zijn aantoonbaar fout of intern tegenstrijdig.**
   - **Waarom kritiek:** verkeerde virtualenvnaam, dubbel API-startpad, ontbrekende Supabase-networkvoorwaarde en foutieve quality-gateflags blokkeren onboarding.
   - **Actie:** test alle commando's in een schone VM/container en publiceer één golden path. Laat CI de quickstart uitvoeren; documenteer alternatieven alleen na het werkende standaardpad.

4. **Geen canoniek database- en migratiecontract.**
   - **Waarom kritiek:** `migration/schema.sql`, `001_init_schema.sql`, de lokale/ongetrackte `002_upgrade_embeddings_to_3072.sql`, frontend Drizzle migrations en `supabase_cli/schema.sql` zijn niet als ordered chain verbonden. De snapshot bevat 768-dim document/call vectors terwijl huidige embeddingcode en migration 002 naar 3072 wijzen.
   - **Actie:** benoem één schema-authority, maak een migration ledger met applied status en rollback, documenteer 768→3072-impact, RLS/publications/RPC's en voeg schema-driftchecks toe.

5. **API-contract beschrijft auth, identity en tenancy niet.**
   - **Waarom kritiek:** frontend maakt een JWT, maar FastAPI V3/V4-routes gebruiken `get_current_user` niet en V3 gebruikt standaard `user_id=1`. Docs suggereren een beveiligd gebruikerscontract dat de code niet afdwingt; data-isolatie kan daardoor niet worden beoordeeld.
   - **Actie:** maak OpenAPI + aparte SSE event schema's canoniek; leg auth flow, claims, user-ID-mapping, error semantics, versioning en ownership vast. Voeg contracttests tussen `webapp-ui` proxies/clients en FastAPI toe.

6. **Security handover ontbreekt terwijl concrete risico's aanwezig zijn.**
   - **Waarom kritiek:** hardcoded Mapbox-token, JWT fallback-secret, `httpOnly: false` voor access cookies, misleidende `PUBLIC_SUPABASE_SERVICE_KEY`, uitgeschakelde globale rate limiting, open backendroutes, ontbrekende collaboration membershipchecks en minimale public-table-RLS zijn niet centraal geregistreerd.
   - **Actie:** maak vóór overdracht een threat model en security register; roteer/centraliseer secrets, verwijder publieke service-rolebenaming, specificeer RLS/authz per tabel/route, documenteer CORS/CSRF/cookies/rate limits en voer een menselijke securityreview uit.

7. **De gedocumenteerde test- en kwaliteitsstatus is niet reproduceerbaar.**
   - **Waarom kritiek:** backendcollectie faalt op `neo4j/import`; er is geen CI; frontend heeft geen `npm run test`; `npm run check` geeft 64 fouten en `npm run lint` meldt formatproblemen in 119 bestanden. PRD-checkboxes zijn daardoor geen betrouwbaar releasebewijs.
   - **Actie:** herstel eerst collection/check gates, definieer unit/integration/live/LLM-testprofielen en required services, voeg hermetische smoke tests en CI toe en publiceer één actueel testrapport per release.

8. **Deploymentdocumentatie is niet uitvoerbaar als production runbook.**
   - **Waarom kritiek:** NGINX-config en PM2-state staan alleen op een specifieke VPS; K8s verwijst naar niet-bestaande Celery entrypoints; frontend heeft geen container/manifests; er is geen rollback of immutable image/tagbeleid.
   - **Actie:** kies de ondersteunde productietopologie, zet alle config als code in Git, pin images/versies, documenteer build→migrate→deploy→verify→rollback en test dit op staging.

9. **Operations, backup en disaster recovery ontbreken.**
   - **Waarom kritiek:** drie datastores plus Supabase auth/realtime vereisen consistente backups en herstelvolgorde. Health endpoints zijn geen runbook.
   - **Actie:** documenteer monitoring, logs, metrics, alerts, SLO's, scheduled jobs/data freshness, capacity limits, backupretentie, restore drills, RPO/RTO en escalatiecontacten.

10. **De feitelijke production-readiness is verborgen en spreekt de hoofdtekst tegen.**
    - **Waarom kritiek:** de getrackte kwaliteitsrapporten melden “NOT PRODUCTION READY” en 54,8%/61% passrate, maar de docs-index linkt ze niet en meerdere hoofd-docs noemen componenten production-ready.
    - **Actie:** maak één gedateerde `KNOWN_ISSUES_AND_RISKS.md` met evidence, impact, workaround, owner en target; laat “production-ready” alleen volgen uit vastgelegde releasecriteria.

## Belangrijke Verbeteringen (P1 — zou moeten oplossen)

1. **Herstructureer documentatie op doelgroep en levenscyclus.** Maak `docs/current/`, `docs/runbooks/`, `docs/decisions/` en `docs/archive/`; geef elk document owner, status (`current`, `proposal`, `historical`), versie en reviewdatum.
2. **Archiveer of label historische specs.** Verplaats V2 streaming, Celery, oude conversation-contextimplementatie, dubbele stability-PRD en afgehandelde sessieverslagen uit de actieve route.
3. **Maak de 42-toolreferentie uit de registry.** Documenteer per tool exacte naam, inputschema/defaults, output, datastore, failure/degraded mode, latency/cost, voorbeelden en tests; genereer de lijst waar mogelijk uit `tool_registry.py`.
4. **Publiceer een volledige environment matrix.** Per variabele: backend/frontend, required/optional, secret/public, default, local/staging/prod en bron. Neem ook ongedocumenteerde tuningvars op (`MAX_CONTEXT_*`, `LLM_TIMEOUT_SECONDS`, enz.) en corrigeer `DATABASE_URL` versus `PRIVATE_DATABASE_URL` voor Drizzle.
5. **Maak een actueel geïntegreerd architectuurdiagram.** Toon browser, SvelteKit serverroutes, Supabase Auth/DB/Realtime, FastAPI, Redis, Neo4j, LLM providers, EU API, artifacts en trust/network boundaries.
6. **Schrijf SvelteKit-native frontendintegratiedocs.** De V4-guide gebruikt circa 1.700 regels React/Redux-voorbeelden in een SvelteKit-repo; vervang die door de daadwerkelijk gebruikte services, stores en proxyroute.
7. **Maak data provenance operationeel.** Leg bron-URL, downloadtijd, programme year, rechten/licentie, parsingversie, chunking, embeddingmodel/dimensie, row counts en freshness SLA vast.
8. **Centraliseer TODO/FIXME/risico's.** Neem minimaal de uitgeschakelde rate limiting, drie ontbrekende collaboration checks, ontbrekende e-mailintegratie en hardcoded/local-only configuratie op.
9. **Documenteer dependency- en serviceversies.** Python/Node/Supabase CLI/PostgreSQL/pgvector/Neo4j/Redis/LLM-modellen plus upgradebeleid en compatibility matrix; corrigeer Python 3.9 versus 3.11.
10. **Voeg release governance toe.** `CHANGELOG.md`, semantische app/schema/data-versies, release notes, migratiecompatibiliteit, rollback en ADR's voor majeure architectuurkeuzes.

## Nice-to-have (P2)

1. Voeg een funding-domainwoordenlijst toe (RIA, IA, CSA, call/topic, programme/cluster, RRF, RAG).
2. Voeg doc-linting toe voor broken links, onbekende codepaden, ongeldige CLI-flags, verouderde endpointversies en leeftijd/owner metadata.
3. Maak executable examples/Postman- of `.http`-collections voor V3, V4, feedback, health en proposal-documentroutes.
4. Voorzie voorbeeldvragen van verwachte toolkeuze, minimaal antwoordcontract en golden test-ID in plaats van alleen losse prompts.
5. Voeg korte “how to extend” guides toe voor een nieuw programma, tool, route, migration, SSE-event en frontendcard.
6. Publiceer een component-/routekaart van SvelteKit en een actuele database-ERD naast de bestaande classdiagram-assets.
7. Voeg kosten- en quota-informatie toe voor Gemini/OpenAI/OpenRouter, embeddings, Supabase en Mapbox.

## Aanbevolen Actie Plan

| Volgorde | Prioriteit | Actie | Geschatte effort | Acceptatiecriterium |
|---:|---:|---|---:|---|
| 1 | P0 | Owner aanwijzen; huidige code/schema/data/release baseline vastzetten | 0,5 dag | Eén getagde baseline met verantwoordelijke per repo en datastore. |
| 2 | P0 | `START_HERE.md` + port/service matrix + fresh-clone quickstart | 1 dag | Een nieuwe developer bereikt frontend, backend health en één chat-smoke-test zonder mondelinge hulp. |
| 3 | P0 | Reproduceerbare data bootstrap en artifactmanifest | 1,5–2 dagen | Schone omgeving laadt schema + minimale dataset; checksums en verificatiequeries slagen. |
| 4 | P0 | Canonieke migration chain en schema-/RLS-driftcontrole | 1,5–2 dagen | Lege DB migreert deterministisch; 35 public tables/15 views en vectorversies zijn gecontroleerd. |
| 5 | P0 | API/SSE/auth contract vastleggen en contracttests toevoegen | 1,5–2 dagen | Frontendclients valideren tegen echte V3/V4 schema's; user identity komt niet uit default `1`. |
| 6 | P0 | Securityregister + secrets/authz/RLS/rate-limitreview | 2–3 dagen documentatie/review, exclusief fixes | Alle routes/tabellen hebben authn/authz/RLS-besluit; secrets en bekende risico's hebben owner. |
| 7 | P0 | Testgates herstellen en CI invoeren | 1–2 dagen | Backend collection/tests en frontend check/lint/tests draaien in CI met gescheiden live-testprofiel. |
| 8 | P0 | Deployment- en operationsrunbook als code | 1,5–2 dagen | Staging deploy en rollback zijn door een tweede persoon uitgevoerd; backups zijn proefhersteld. |
| 9 | P0 | `KNOWN_ISSUES_AND_RISKS.md` publiceren | 0,5 dag | Actuele blockers, quality baseline, TODO's, impact, workarounds en owners staan centraal. |
| 10 | P1 | Docs opschonen: current versus archive, dubbelen verwijderen | 1–2 dagen | Hoofdindex bevat alleen current docs; historische PRD's/specs zijn expliciet gelabeld. |
| 11 | P1 | Tool-, env-, dependency- en dataprovenancereferenties vernieuwen | 2–3 dagen | Referenties worden door scripts/CI tegen code en manifests gevalideerd. |
| 12 | P2 | Doc QA, glossary, examples en ADR-template | 1 dag | Broken-link/path/age checks draaien automatisch en contributors hebben templates. |

**Indicatie:** circa **14–20 ontwikkelaarsdagen** voor documentatie, validatie en beperkte ondersteunende automatisering; echte security-, typecheck-, deployment- en datamigratiefixes kunnen daar bovenop komen.

## Per-Document Details

### Backend root en guides

#### `MAP-IT_SA/README.md` — status: ingrijpend herzien

- **Issues:** quickstart combineert `make docker-up` (API op 18000, plus externe Supabase-network) met `make run-reload` (tweede API op 8000); Supabase CLI/startlocatie, frontend, dataartifact en smoke test ontbreken. “Production-grade” botst met de eigen auditwaarschuwing en verborgen kwaliteitsresultaten.
- **Fix:** maak dit een korte ingang naar de full-stack quickstart, ondersteunde modes, huidige readiness en canonical docs; verplaats credits/contactgegevens naar ownership/support.

#### `MAP-IT_SA/CLAUDE.md` — status: behouden als AI-agentguide, technisch opschonen

- **Issues:** verwijst naar verwijderde `AgenticManager`, `agentic_manager.py`, `enhanced_rag_service.py`, `intent_detection_service.py`, `app/api/routes.py`, `celery_app.py`, ontbrekende CI-workflow en verkeerd benoemde `DATABASE_SCHEMA_DOCUMENTATION.md`. Door omvang en overlap is het geen veilige onboarding-SSOT.
- **Fix:** beperk tot repositoryconventies en gegenereerde current references; link naar aparte architecture/API/data/test runbooks en voeg een automatische pathcheck toe.

#### `MAP-IT_SA/GEMINI.md` — status: consolideren

- **Issues:** niet in Git, herhaalt README/CLAUDE en noemt geen afwijkende Gemini-specifieke workflow.
- **Fix:** verwijder of commit als dunne verwijzing naar één agent-instructionfile; vermijd parallelle bron van waarheid.

#### `MAP-IT_SA/questions.md` — status: hernoemen en formaliseren

- **Issues:** geen titel, doel, verwachte uitkomst, toolset, datasetversie of teststatus.
- **Fix:** maak `docs/examples/demo_queries.md` of parametriseer als acceptatietests met expected tools/claims.

#### `MAP-IT_SA/docs/README.md` — status: vervangen

- **Issues:** noemt niet-bestaande `docs/implementation/` en vier Erasmus-files; statistieken zijn fout; “no redundancy” is aantoonbaar onjuist; verborgen K8s/Supabase/qualitydocs ontbreken.
- **Fix:** genereer de index uit metadata en label elk document current/proposal/historical/archive met owner en last-reviewed.

#### `MAP-IT_SA/docs/DATABASE_SCHEMA.md` — status: behouden, genereren/verifiëren

- **Issues:** statische recordaantallen zonder snapshotdatum/query; geen kolommen, constraints, RLS, ownership, migrations of ERD. Huidige worktree beweegt document/call/cache-embeddings naar 3072 terwijl de snapshot nog 768 toont.
- **Fix:** genereer objectlijsten uit migrations/live schema, voeg ERD, vectorversiematrix, RLS/publication/RPC-details en reproducible count queries toe.

#### `MAP-IT_SA/docs/guides/DATA_INGESTION.md` — status: P0 herschrijven

- **Issues:** dump niet in clone; Neo4j-ingestbron ontbreekt; genoemde quality flags bestaan niet; geen inputbestanden/CLI-opties, idempotentie, resume, cleanup, provenance of fout herstel.
- **Fix:** bouw een getest runbook per dataset met prerequisites, source artifact, schema version, exact command, expected counts, retry/rollback en verificatie.

#### `MAP-IT_SA/docs/guides/ROADMAP.md` — status: vervangen door evidence-based status

- **Issues:** noemt V3/multi-programme/V4 “production-ready” zonder releasecriteria en terwijl actuele quality reports het tegendeel melden; collaboration “early stage” is niet gekoppeld aan concrete TODO's.
- **Fix:** gebruik een issue-/risicoregister met evidence, owner, statusdatum, scope en links naar tests.

#### `MAP-IT_SA/docs/guides/TOOL_REFERENCE.md` — status: uitbreiden/genereren

- **Issues:** titel/intro claimen 42 tools maar een aanzienlijk deel ontbreekt; parameters wijken soms van registry af en output-/errorcontracten ontbreken.
- **Fix:** genereer toolnamen/inputschemas uit `tool_registry.py` en voeg output, datastore, degraded mode, voorbeelden en testlinks toe.

### Backend PRD's

#### `MAPIT_ADVANCED_CONVERSATION_FLOWS_PRD.md` — status: historisch bewijs, niet current spec

- **Issues:** alle taken zijn afgevinkt, maar genoemd quality-baselinebestand en CI-workflow ontbreken; scenarioaantallen en files zijn niet als releasebewijs gekoppeld.
- **Fix:** archiveer met completiondatum/commit en link naar actueel testrapport; verplaats blijvende requirements naar current acceptance criteria.

#### `MAPIT_APPLICATION_QUALITY_IMPROVEMENT_PRD.md` — status: archiveren

- **Issues:** zegt 34 tools en “all verified”, maar noemt tien niet-bestaande scripts (`run_prd_scenarios.py`, sync/check scripts) en biedt geen reproduceerbaar resultmanifest.
- **Fix:** archiveer als plan; draag open requirements en echte test-ID's over naar een current quality plan.

#### `MAPIT_CONVERSATION_STABILITY_PRD.md` — status: archiveren na reconciliatie

- **Issues:** nuttige checklists, maar verwijst naar ontbrekende `docs/MAPIT_IMPROVEMENT_REPORT.md` en CI; dupliceert `MAPIT_STABILITY_PRD.md` en de architectuurguide.
- **Fix:** bewaar requirements/results in één statuspagina; zet implementatiedetails in de stability architecture.

#### `MAPIT_MULTI_PROGRAMME_EXCELLENCE_PRD.md` — status: actief maar expliciet open labelen

- **Issues:** 70 open checkboxes, niet-bestaand `tests/test_multi_programme_scenarios.py`, en `search_digital_europe_templates` is geen geregistreerde tool.
- **Fix:** koppel requirements aan bestaande tests/tools of maak tickets; voeg owner, target release en datasetversion toe.

#### `MAPIT_PRODUCTION_QUALITY_VALIDATION_PRD.md` — status: superseded/archiveren

- **Issues:** gemengde 32 checked/130 open items zonder heldere supersession; overlapt V3-versie en advanced-flow-PRD.
- **Fix:** markeer bovenaan “superseded by …”; bewaar alleen historische traceability.

#### `MAPIT_PRODUCTION_QUALITY_VALIDATION_PRD_V3.md` — status: results scheiden van requirements

- **Issues:** verwijst naar verwijderde `agentic_manager.py`, `replanner.py`, `result_evaluator.py` en oude geneste scenariopaden; finale resultaten staan los van de aparte “NOT PRODUCTION READY”-rapporten.
- **Fix:** zet immutable requirements apart van gegenereerde testruns; link naar één latest run met code/data/modelversie.

#### `MAPIT_QUALITY_PRD.md` — status: historisch evaluatierapport

- **Issues:** afgevinkte prompts mengen offline samples, live scores en implementatieclaims; geen datum/model/dataset per score en geen huidige regressiestatus.
- **Fix:** archiveer de run en migreer prompts naar versioned testcases; rapporteer huidige pass/fail in CI.

#### `MAPIT_STABILITY_PRD.md` — status: verwijderen uit actieve docs

- **Issues:** volledig onafgevinkte oudere duplicaatversie van conversation stability; wekt twijfel of werk wel/niet klaar is.
- **Fix:** archiveer als superseded of verwijder na behoud in Git-history.

#### `MAPIT_V3_RESPONSE_FORMATTER_PRD.md` — status: actief plan actualiseren

- **Issues:** gaat uit van 34 tools, terwijl registry 42 bevat; `tests/test_formatter_integration.py` bestaat niet en formattercoverage is niet automatisch gemeten.
- **Fix:** herbaseer op 42 geregistreerde tools, koppel elke outputvorm aan tests en publiceer echte coverage.

### Backend `docs_tracked`

#### `CONVERSATION_STABILITY_ARCHITECTURE.md` — status: behouden, commands valideren

- **Issues:** sterk ontwerp, maar te veel overlap met CLAUDE/PRD's; health/debugroutes en scenario-runcommands missen een bewezen fresh-clone context en CI is afwezig.
- **Fix:** maak dit de canonical stability design doc en verplaats runcommands naar een getest test-runbook.

#### `DEVELOPER_SETUP.md` — status: P0 herschrijven

- **Issues:** `venv`/`.venv` mismatch; `make run`-betekenis fout; Supabase wordt in de backendroot geïnitialiseerd in plaats van de bestaande frontendconfig te gebruiken; V4-sequentie noemt niet-bestaande `/api/v4/proposals/sessions`-routes; Neo4j copy kan door volumepermissions falen.
- **Fix:** vervang door het gevalideerde full-stack golden path en correcte `/api/v4/proposal-session/...`-routes.

#### `ENHANCED_HORIZON_QUESTIONS_CAPABILITIES.md` — status: bewijs toevoegen of archiveren

- **Issues:** metrics (83,3%, 0,837 s) hebben geen test/datum/commit; `client-centricity.io`-architectuur is niet traceerbaar in huidige code; “all completed” is niet verifieerbaar.
- **Fix:** link claims aan benchmarkresultaten en huidige services, of label als historische oplevernotitie.

#### `EU_PROGRAMME_DATA_DOWNLOADS.md` — status: ombouwen tot provenance manifest

- **Issues:** download/checklistdocument zonder artifactstatus, exacte bestanden, checksums, licenties, ingestmapping of freshness; oudere 2025-links en handmatige stappen domineren.
- **Fix:** maak per dataset een machineleesbaar manifest en documenteer sync owner/SLA en welke bestanden reeds in `newdocs/` zitten.

#### `FRONTEND_INTEGRATION_PROMPT.md` — status: archiveren na extractie

- **Issues:** noemt 20 tools en is een lange bouwprompt, geen onderhouden contract; overlapt toolguide, streaming spec en bestaande Sveltecomponents.
- **Fix:** migreer blijvende UX/eventrequirements naar SvelteKit component- en contractdocs; archiveer prompt.

#### `FRONTEND_STREAMING_V3_SPEC.md` — status: onmiddellijk als historical markeren

- **Issues:** V2-endpoint en verwijderde files zijn “current”; V3 is “pending”; voorbeeld gebruikt V2; fallback bestaat niet. Complete-eventvorm en agent-iterationevents wijken van huidige code af.
- **Fix:** vervang door een schema uit echte emissions in `ConversationManager`/`ToolAwareAgent`, inclusief top-level versus `metadata` fields, ordering, storage errors en reconnect/cancel semantics.

#### `IMPROVEMENT_ROADMAP.md` — status: opsplitsen en archiveren

- **Issues:** status bovenaan zegt dat veel prioriteiten geïmplementeerd zijn, maar “Current State” zegt het tegenovergestelde; 20 tools; veel voorgestelde migrations/files bestaan niet; 2.277 regels zijn onbruikbaar als actueel issuebeheer.
- **Fix:** archiveer ontwerpvoorbeelden en zet resterend werk in een kort, eigenaar-gestuurd register of issue tracker.

#### `MANUAL_TEST_SCENARIOS.md` — status: behouden na executable koppeling

- **Issues:** nuttige vragen maar geen expected outputs/data baseline; twee genoemde enrichscripts ontbreken; performancecriteria zijn niet aan metingen gekoppeld.
- **Fix:** koppel ieder scenario aan test-ID, preconditions, expected tool/contract en runrecord; verwijder ongeldige scripts.

#### `PROPOSAL_WRITER_FRONTEND_INTEGRATION.md` — status: één canonical kopie, Svelte-versie maken

- **Issues:** vrijwel identiek gedupliceerd in frontend; React/Redux-voorbeelden passen niet bij SvelteKit; migration `015_proposal_sessions.sql` ontbreekt, terwijl echte routes grotendeels wel kloppen.
- **Fix:** houd het contract bij backend/OpenAPI, maak in frontend alleen Svelte services/stores/examples en link beide.

#### `QUESTION_CATEGORIES.md` — status: capability matrix actualiseren

- **Issues:** noemt 20 tools, legacy `/api/query*`-routes die niet door FastAPI zijn gemount en features zoals exports/version comparison zonder statusbewijs.
- **Fix:** splits ondersteund/getest/gepland; genereer endpoints/tools en koppel scenario's aan evidence.

#### `README_CELERY.md` — status: verwijderen uit actieve docs

- **Issues:** vrijwel alle operationele claims zijn niet uitvoerbaar door ontbrekende compose, workers, task module, endpoints en testscript.
- **Fix:** archiveer expliciet als abandoned design, of implementeer en test de stack vóór herpublicatie.

#### `SYSTEM_ARCHITECTURE.md` — status: canonical basis, aanvullen

- **Issues:** kern V3/V4 en 42 tools zijn bruikbaar, maar envvoorbeeld gebruikt `SUPABASE_KEY` in plaats van anon/service keys; frontend/auth/deployment/trust boundaries ontbreken; RRF-claim is niet overal even precies; link naar toolguide overschat volledigheid.
- **Fix:** maak dit de geïntegreerde current architecture met versie/owner/reviewdatum en gegenereerde service/route/toolreferenties.

#### `example_questions.md` — status: reduceren of testbaar maken

- **Issues:** honderden vragen zonder classificatie van supported/aspirational, verwachte tools of waarheid; overlap met `questions.md`, manual scenarios en PRD's.
- **Fix:** curateer een kleine onboarding/demo-set en verplaats de rest naar versioned evaluatiedata.

### Frontend root en docs

#### `webapp-ui/README.md` — status: uitbreiden

- **Issues:** geen prerequisites/Node-versie, env setup, Supabase start, backendstart, authflow, teststatus, routes of troubleshootinglinks. Databasecommands verbergen `DATABASE_URL` versus `PRIVATE_DATABASE_URL` mismatch.
- **Fix:** maak README de frontend-ingang naar de unified quickstart en documenteer daadwerkelijk ondersteunde scripts en quality gates.

#### `webapp-ui/CRUSH.md` — status: corrigeren/consolideren

- **Issues:** noemt niet-bestaand `npm run test`; “Svelte 5+ conventions” wordt niet aan current migrationguidance gekoppeld; overlap met README.
- **Fix:** genereer commandlijst uit `package.json`; bewaar alleen agent-specifieke conventies.

#### `webapp-ui/DEPLOYMENT_GUIDE.md` — status: als code reproduceerbaar maken

- **Issues:** hardcoded domein/VPS en `/etc/nginx`-state; geen volledige NGINX-file, DNS/firewall/Supabase service lifecycle, frontend healthcheck, rollback, backups of secret manager. Python 3.9 wijkt af van backend 3.11+. Cookieadvies stimuleert `httpOnly: false`; frontend check faalt.
- **Fix:** versioneer NGINX/PM2 of container manifests, scheid local/staging/prod, pin runtimes en voeg build/check/migrate/deploy/rollback-verificatie toe.

#### `webapp-ui/SESSION_SUMMARY.md` — status: archiveren

- **Issues:** een tijdgebonden incident-/werksessie wordt als onboardingbron gebruikt; codevoorbeeld voor `response.headers.set` met cookieopties is in huidige TS foutief en de fix is domeinspecifiek.
- **Fix:** zet blijvende beslissingen in ADR/runbook en archiveer het sessieverslag.

#### `webapp-ui/TROUBLESHOOTING.md` — status: runbookmatig herschrijven

- **Issues:** adviseert live code te wijzigen voor cookiegedrag, gebruikt één productiedomein, mist veilige diagnosevolgorde/rollback en behandelt symptomen zonder auth/Supabase/backend ownership.
- **Fix:** voeg severity, preconditions, read-only checks, expected output, veilige remediatie, rollback en escalatie toe.

#### `AI_Proposal_Writer_Frontend_Integration_Guide.md` — status: dedupliceren

- **Issues:** vrijwel kopie van backendguide, React-code in Svelte-repo, ontbrekende migration en geen link naar de werkelijk geïmplementeerde `proposalSessionService.ts`/store/proxy.
- **Fix:** vervang door een korte SvelteKit implementatieguide die naar één backendcontract linkt.

#### `proposal_editor_report.md` — status: label als snapshot

- **Issues:** “currently used” lijst bevat niet-bestaande `/api/proposal/generate-template` en `/events`; aanbeveling en implementatiestatus lopen door elkaar; frontendcomponent bevat bovendien een hardcoded `http://localhost:8000` export-URL.
- **Fix:** dateer/commit-label de snapshot of vervang door current component/route/state diagram met known gaps.

#### `CITATION_IMPLEMENTATION_VERIFICATION.md` — status: open testplan formaliseren

- **Issues:** genoemde `citationRenderer.test.ts` en testinfrastructuur ontbreken; checklist is niet uitgevoerd; actual persistence/schema/proxy-flow is niet als contract vastgelegd.
- **Fix:** implementeer of ticket tests, voeg test runner toe en publiceer runresultaat/browsermatrix.

#### `CONVERSATIONAL_AI_ARCHITECTURE.md` — status: archiveren als pre-V3 voorstel

- **Issues:** stelt `conversation_context_service.py`, `rag_service.py`, oude routes en DB-wijzigingen voor die niet de huidige V3 memory/stabilityarchitectuur vormen; alle fasen staan open.
- **Fix:** markeer superseded door backend stability/system architecture en verwijder uit onboarding.

#### `CONVERSATION_IMPLEMENTATION_GUIDE.md` — status: archiveren

- **Issues:** 945 regels copy-paste-Python voor niet-bestaande backendfiles in de frontendrepo; oude `/api/conversation`-contracten en datamodellen.
- **Fix:** bewaar alleen historisch; schrijf geen backendimplementatieguide in frontenddocs.

#### `PROPOSAL_EDITOR_IMPLEMENTATION.md` — status: actualiseren

- **Issues:** claimt comprehensive error handling/mobile/accessibility en mockroutes zonder geautomatiseerd bewijs; linkt naar ontbrekende `API_DOCUMENTATION.md` en `COMPONENT_LIBRARY.md`; future work overlapt inmiddels aanwezige collaborationcode.
- **Fix:** toets claims aan current UI/tests, herstel links en scheid implemented/partial/planned.

#### `RATE_LIMITING.md` — status: securitykritisch corrigeren

- **Issues:** globale bescherming wordt als actief beschreven terwijl de middleware uitstaat; in-memory limiter is niet clusterbestendig en production proxy trust/config ontbreekt.
- **Fix:** documenteer werkelijke dekking per route, enablementflag, trusted proxyketen, Redis/distributed strategie, observability en contracttests.

#### `SUPABASE_LOCAL_REALTIME_SETUP.md` — status: integreren in unified setup

- **Issues:** handmatige SQL is al aanwezig in de grote schema dump maar migration ownership ontbreekt; `supabase/migrations`-pad bestaat niet in de gebruikte `supabase_cli`-structuur.
- **Fix:** zet publicationwijziging in canonical migration chain en maak dit een korte verificatie/troubleshootingsectie.

#### `SUPABASE_REALTIME_SETUP.md` — status: RLS herschrijven en valideren

- **Issues:** voorbeeld-RLS is niet als migration opgenomen en de actual schema dump activeert RLS niet voor `message`, `conversation`, `client` of collaborationtabellen. Tijdelijke permissive policy mist een harde waarschuwing/cleanupcheck.
- **Fix:** laat security-reviewed policies als migrations gelden en documenteer claims, membershipmodel, Realtime replica identity en tests.

#### `project_docs.md` — status: verwijderen of voltooien

- **Issues:** expliciete WIP-stub zonder actuele architectuur, setup of ownership; voegt naast README nauwelijks waarde toe.
- **Fix:** merge geldige conventie in README/databaseguide en archiveer de stub.

#### `project_setup.md` — status: onmiddellijk als legacy/destructive labelen

- **Issues:** adviseert migrations te verwijderen, schema handmatig in Studio uit te voeren, Supabase API-poort publiek open te zetten en een oud PM2-pad te gebruiken; conflicteert met de recentere deploymentguide.
- **Fix:** archiveer; verwijs uitsluitend naar het gevalideerde, geautomatiseerde setup/deploypad.

## Conclusie voor overdracht

De documentatie toont veel domeinkennis en geeft een nieuwe developer nuttige losse bouwstenen, maar functioneert nog niet als overdrachtspakket. De beslissende stap is niet “meer documentatie schrijven”, maar **één aantoonbaar werkende bron van waarheid maken**: fresh-clone setup, versioned data/schema, echt API/authcontract, reproduceerbare quality gates, securityregister en production runbook. Pas wanneer een tweede developer die route zonder impliciete kennis heeft uitgevoerd en de P0-acceptatiecriteria zijn vastgelegd, is MAPIT documentair overdrachtsklaar.
