# Bekende problemen en risico's

**Statusdatum:** 20 juli 2026  
**Scope:** `MAP-IT_SA` (backend) en `webapp-ui` (frontend)  
**Releasebesluit:** **NIET PRODUCTIERIJP**  
**Eigenaar register:** TBD

Dit is het centrale, eerlijke statusregister voor overdracht. Een item is pas gesloten nadat de oplossing in code of infrastructuur is aangebracht en reproduceerbaar is geverifieerd. Een afgevinkt PRD-item geldt niet als bewijs. Zie ook het [securityregister](SECURITY_REGISTER.md), het [databaseschema](MAP-IT_SA/docs/DATABASE_SCHEMA.md) en de [onboardingguide](START_HERE.md).

## Quality baseline

De nieuwste aanwezige finale kwaliteitsrapportage is van **23 januari 2026**; er is geen nieuwer releasebewijs in de repository.

| Bron | Resultaat | Betekenis |
|---|---:|---|
| `MAP-IT_SA/tests/quality/results/FINAL_VERIFICATION_SUMMARY.md` | 23/42 scenario's geslaagd; **54,8%**; gemiddelde kwaliteit 82,0/100; factual 97,9% | Expliciete status: **NOT PRODUCTION READY**; vereist was 80% passrate. |
| `MAP-IT_SA/tests/quality/results/PRODUCTION_READINESS_SUMMARY.md` | 25/41 gevonden scenario's geslaagd; **61,0%**; gemiddelde kwaliteit 81,4/100 | Eerdere run/samenvatting; eveneens expliciet **NOT READY**. |
| `MAP-IT_SA/tests/quality/results/production_readiness_report.json` | Zelfde 61,0%-run, gegenereerd 23 januari 2026 15:53 | Machineleesbare bron bij de eerdere samenvatting. |
| Documentatiereview van 20 juli 2026 | `make test`/`pytest -q` stopt tijdens collectie op `neo4j/import`; `npm run check` rapporteert 64 fouten en 76 waarschuwingen; `npm run lint` meldt 119 bestanden | De huidige algemene quality gates zijn niet groen. De frontend heeft geen `test`-script. |

De twee scenario-passrates zijn geen alternatieve succescriteria: ze zijn verschillende snapshots van dezelfde validatiecampagne. De finale, later gegenereerde 54,8%-rapportage is leidend. Geen van beide bewijst productiegeschiktheid.

## Betekenis van severity en status

- **critical:** reële kans op data-isolatieverlies, secret exposure, niet-reproduceerbare kernfunctionaliteit of onverantwoorde productiegang.
- **high:** blokkeert betrouwbare overdracht, releasevalidatie of herstel; kan een groot deel van gebruikers raken.
- **medium:** belangrijke beperking of schuld met een afgebakende workaround.
- **low:** beperkte impact of vooral onderhoudslast.
- **open:** niet opgelost; **in-progress:** aantoonbaar in uitvoering; **accepted:** expliciet geaccepteerd met eigenaar en herbeoordelingsdatum.

## Known issues

| ID | Beschrijving en bewijs | Severity | Impact | Workaround | Owner | Status |
|---|---|---|---|---|---|---|
| QUAL-001 | Finale scenario-passrate is 54,8%, onder de vereiste 80%; 19 van 42 scenario's falen. | critical | Antwoorden en multi-turnflows zijn niet voldoende gevalideerd voor productie. | Alleen gecontroleerde demo/staging; controleer kritieke antwoorden handmatig. | TBD | open |
| QUAL-002 | Backendtests worden door `pyproject.toml` uit `tests` én `.` verzameld; collectie raakt de niet-leesbare bindmount `neo4j/import`. | high | `make test` levert geen betrouwbare regressie-uitspraak. | Herstel permissies of sluit runtimevolumes in pytest-config uit voordat resultaten als gate gelden. | TBD | open |
| QUAL-003 | `npm run check` heeft volgens de review 64 fouten/76 waarschuwingen; lint heeft 119 formattingafwijkingen. | high | Frontendbuild en runtimegedrag zijn niet betrouwbaar vrijgegeven. | Geen productie-uitrol; voer `npm run check`, `npm run lint` en `npm run build` als harde gates uit. | TBD | open |
| QUAL-004 | `webapp-ui/package.json` bevat geen `test`-script en geen van beide repo's bevat een CI-workflow. | high | Geen automatische contract-, browser- of regressiebescherming. | Handmatige smoke tests uit [START_HERE](START_HERE.md); dit is geen gelijkwaardige vervanging. | TBD | open |
| DATA-001 | De genoemde PostgreSQL-dump zit niet in een verse clone: `/neo4j_backup/` is genegeerd en alleen een lokale dump is aangetroffen. | critical | De representatieve kernset kan niet zelfstandig worden opgebouwd. | Vraag een versie, checksum en overdrachtskanaal bij de data-eigenaar aan, of voer raw ingest uit volgens [DATA_INGESTION](MAP-IT_SA/docs/guides/DATA_INGESTION.md). | TBD | open |
| DATA-002 | `scripts/ingest_cordis_to_neo4j.py` verwacht standaard `data/cordis`, maar `/data/` is genegeerd en het bronartifact ontbreekt. | high | Neo4j kan vanuit een clone niet worden gevuld. | Lever CORDIS-artifacts buiten Git met manifest/checksum; start niet met een lege map zonder provenance. | TBD | open |
| DATA-003 | Schema-snapshot heeft 768-dimensionale vectors voor `funding_calls`, `document_chunks` en `embedding_cache`, terwijl config/code 3072 en migratie `002` voorschrijven; bestaande waarden worden daarbij verwijderd. | critical | Vectorinsert/search kan dimension errors geven; migratie veroorzaakt bewuste her-embedding. | Zet vector search uit totdat schema en data dezelfde versie hebben; maak vooraf backup en herembed daarna alles. | TBD | open |
| DATA-004 | Er is geen canonieke, door tooling toegepaste migration chain: volledige dumps, `001`, `002`, `008`, `009` en Drizzle-schema bestaan naast elkaar. | high | Schema drift en onvolledige deployments zijn waarschijnlijk. | Volg tijdelijk de ledger in [DATABASE_SCHEMA](MAP-IT_SA/docs/DATABASE_SCHEMA.md) en leg per omgeving toegepaste checksums vast. | TBD | open |
| API-001 | V3/V4 FastAPI-routes passen de aanwezige `get_current_user` dependency niet toe; V3-requestmodellen gebruiken standaard `user_id=1`. | critical | Een directe backendcaller kan identiteit kiezen/delen; tenantisolatie is niet afgedwongen. | Backend niet publiek exposen; alleen gecontroleerde frontendproxy toelaten. Dit is defense-in-depth, geen definitieve fix. | TBD | open |
| API-002 | SSE-schema's stonden alleen in een verouderde V2/V3-spec en events hebben niet overal dezelfde top-levelvelden. | high | Frontend kan events missen of verkeerd afhandelen. | Gebruik [API_CONTRACT](API_CONTRACT.md) als huidige code-snapshot; voeg contracttests toe. | TBD | open |
| SEC-001 | Een Mapbox-token staat hardcoded in frontendbron. | medium | Tokenmisbruik/quota- en kostenrisico; rotatie vereist een codefix. | Beperk token op origin/scopes bij Mapbox en roteer; details in [SECURITY_REGISTER](SECURITY_REGISTER.md). | TBD | open |
| SEC-002 | Backend-JWT-validatie heeft een fallback-secret wanneer configuratie ontbreekt. | critical | Tokens kunnen vervalst worden als de fallback actief is. | Zet een uniek sterk `JWT_SECRET_KEY`, blokkeer startup bij placeholder/fallback en roteer bestaande tokens. | TBD | open |
| SEC-003 | Auth access cookies worden met `httpOnly: false` gezet. | high | XSS kan tokens uitlezen; groter accountovernamerisico. | Beperk sessieduur en CSP; structurele fix is server-only HttpOnly-cookiebeheer. | TBD | open |
| SEC-004 | `PUBLIC_SUPABASE_SERVICE_KEY` suggereert een publieke service-rolekey en wordt in `.env.example` zo gevuld. | critical | Service-role exposure omzeilt RLS en kan de hele database compromitteren. | Gebruik uitsluitend server-private `SUPABASE_SERVICE_ROLE_KEY`; scan builds/logs en roteer bij mogelijke exposure. | TBD | open |
| SEC-005 | Globale rate limiting in `webapp-ui/src/hooks.server.ts` is uitgecommentarieerd. | high | Login/API/LLM-routes zijn vatbaarder voor misbruik en kostendoorgroei. | Tijdelijk beperken in NGINX/firewall; app-level, gedeelde limiter blijft nodig. | TBD | open |
| SEC-006 | Collaborationroutes bevatten drie expliciete TODO's voor membershipcontrole. | critical | Mogelijke toegang tot gesprekken van andere samenwerkingen. | Collaboration niet voor onvertrouwde multi-userproductie activeren. | TBD | open |
| SEC-007 | In de schema-snapshot hebben alleen `document_chunks` en `horizon_documents` public-table-RLS; o.a. conversation/collaboration/message hebben geen RLS. | critical | Directe Supabase-toegang kan gegevens buiten de bedoelde gebruiker/tenant tonen of wijzigen. | Beperk anon/authenticated grants, gebruik server-side service key en implementeer geteste policies vóór productie. | TBD | open |
| DEP-001 | Backend-Compose gebruikt `--reload`, bindmount de bron en vereist het externe netwerk `supabase_network_supabase_cli`. | high | Dit is een developmentstack, geen immutable of zelfstandig productieartifact. | Gebruik het expliciete VPS-pad in [DEPLOYMENT_RUNBOOK](DEPLOYMENT_RUNBOOK.md); stel Compose-API niet publiek beschikbaar. | TBD | open |
| DEP-002 | K8s-manifests starten `app.services.celery_app`, dat niet bestaat, refereren `/metrics` dat niet bestaat, gebruiken `latest`, en bevatten geen frontenddeployment. | high | `kubectl apply -k` levert geen complete werkende productieomgeving. | Kubernetes is niet ondersteund; gebruik de gekozen VPS-topologie. | TBD | open |
| OPS-001 | Backups, restore drills, alerts, SLO's en on-call zijn niet geïmplementeerd of toegewezen. | high | Data- en hersteltijd zijn niet beheerst. | Voer de voorgestelde procedures uit [OPERATIONS_RUNBOOK](OPERATIONS_RUNBOOK.md) in en registreer elke restoretest. | TBD | open |
| OPS-002 | Er draait geen werkende scheduler: Celery worker/beat ontbreken; `process-alerts` is een handmatig endpoint en incremental embeddings zijn alleen voor cron/Celery ontworpen. | high | Call alerts, synchronisatie en embeddings worden niet gegarandeerd ververst. | Handmatig uitvoeren met runlog; configureer pas daarna een beheerde scheduler met locking/alerting. | TBD | open |
| FEAT-001 | Collaboration- en consortiumschermen tonen expliciet dat functionaliteit tijdelijk is uitgeschakeld. | medium | Productscope is kleiner dan sommige roadmapclaims. | Behandel als niet-beschikbaar en communiceer dit aan gebruikers. | TBD | open |
| FEAT-002 | Saved-search e-mailnotificaties hebben een TODO voor e-mailintegratie. | medium | Alerts worden niet per e-mail bezorgd. | Resultaten handmatig ophalen; beloof geen e-mailalerts. | TBD | open |

## Known limits

| Gebied | Huidige grens |
|---|---|
| Vector search | De actieve embeddingconfig is `gemini-embedding-001` met 3072 dimensies, maar de schema-snapshot is deels 768. pgvector-indexering boven 2000 dimensies heeft bovendien versie/indexkeuzes nodig; migratie `002` kiest voorlopig exact/sequential search. |
| Programmavolwassenheid | Horizon Europe heeft de rijkste project-, call-, graph- en scenarioondersteuning. Erasmus+ en Creative Europe hebben specifieke tools en scenario's. LIFE, Digital Europe en EU4Health zijn wel in routing/kennis aanwezig maar hebben geen vergelijkbare scenario-dekking in de finale 42-scenariorun; behandel ze als beperkt/experimenteel. |
| Datareproduceerbaarheid | Volledige projectdata, CORDIS raw files en versie-/checksummanifest ontbreken in Git. Een leeg schema is reproduceerbaar; een representatieve dataset niet. |
| Tooling | `tool_registry.py` registreert 42 tools, maar de bestaande toolguide beschrijft slechts een subset. Registratie bewijst niet dat een tool voor iedere dataset of elk programma gevuld en getest is. |
| Supabase API | Lokale `config.toml` begrenst REST-responses op 1000 rijen. Clientcode moet pagineren. |
| Uploads | Backenddefault is 50 MiB; frontendconfig noemt 10 MiB. De laagste limiet is effectief totdat ze gelijkgetrokken zijn. |
| Externe providers | Gemini/OpenAI/OpenRouter, Google embeddings en EU Portal vallen buiten de beschikbaarheidscontrole van MAPIT. Time-outs, quota, modelwijzigingen en kosten kunnen antwoorden degraderen. |
| Redis | Memory/cache kan degraderen naar lokale/in-memory fallbacks. Die zijn niet gedeeld tussen processen en gaan bij restart verloren. |
| Schaalbaarheid | Lokale Compose draait één API-proces en single-node Redis/Neo4j. Er zijn geen betrouwbare cluster-/loadtests of actieve distributed rate limits. |
| Analytics | Analytics-endpoints en healthmetrics bestaan, maar zijn niet geauthenticeerd en `/api/health/metrics` is JSON, geen Prometheus exposition format. Er is geen dashboard of alertmanagerconfig. |

## Technische schuld en uitgeschakelde onderdelen

| Item | Bewijs/locatie | Gevolg |
|---|---|---|
| Globale limiter uitgeschakeld | `webapp-ui/src/hooks.server.ts` | Alleen eventuele route-/proxy-/NGINX-limieten beschermen verkeer. |
| Membershipchecks ontbreken | `webapp-ui/src/routes/api/conversations/+server.ts`, `.../[conversation_id]/+server.ts`, `api/chat/websocket/+server.ts` | Collaboration-isolatie is niet aantoonbaar. |
| E-mailintegratie ontbreekt | `MAP-IT_SA/app/services/alerts/saved_search_service.py` | Saved-searchalerts kunnen niet als e-mailproduct worden beschouwd. |
| Celery-ontwerp zonder implementatie | `MAP-IT_SA/k8s/deployment.yaml`; gearchiveerde Celery-guide | Worker, beat en periodieke verwerking starten niet. |
| Collaboration/consortium UI uit | Twee dashboardpagina's in `webapp-ui/src/routes/(protected)` | Navigatie kan aanwezige maar onbeschikbare features tonen. |
| Analytics kan stil uitschakelen | `conversation_analytics_service.py`, `tool_analytics_service.py` | Ontbrekende DB-tabellen/connectie leveren gaten in observability. |
| Geen releaseautomatisering | Geen CI-workflows; geen changelog/releasetagbeleid in scope | Build-, schema- en docdrift worden niet automatisch gestopt. |
| Devcontainer als deployclaim | `docker-compose.yml` gebruikt reload/bindmount/default Neo4j-wachtwoord | Config is geschikt voor lokaal gebruik, niet als productiehardening. |

`pass` in abstracte basisklassen of exception-cleanup is niet automatisch een defect en is daarom niet als afzonderlijk issue opgenomen. Bovenstaande schuld bestaat uit concrete open TODO's, ontbrekende entrypoints of aantoonbaar uitgeschakelde features.

## Tegenstrijdigheden in documentatie

| Tegenstrijdige claim | Canonieke waarheid per 20 juli 2026 |
|---|---|
| README/roadmap noemt onderdelen “production-ready”. | De finale quality baseline zegt **NOT PRODUCTION READY**, 54,8%. Dit register is leidend voor readiness. |
| Oude setup zegt `source .venv/bin/activate`. | `scripts/setup.sh` maakt `venv`; gebruik `source venv/bin/activate`. |
| Oude setup suggereert dat `make run` Compose start. | `make run` start lokale Uvicorn op 8000; `make docker-up` start Compose en publiceert de API op 18000; `make run-reload` is lokale Uvicorn met reload op 8000. |
| Setup impliceert dat backend-Compose zelfstandig start. | Compose verwacht het externe netwerk `supabase_network_supabase_cli`; start Supabase vanuit `webapp-ui/supabase_cli` eerst. |
| Oude V4-doc noemt `/api/v4/proposals/sessions`. | De echte prefix is `/api/v4/proposal-session`; zie [API_CONTRACT](API_CONTRACT.md). |
| Streamingdocument behandelt V2 als current en V3 als pending. | Alleen V3/V4 is gemount; de oude streamingdoc is gearchiveerd. |
| Celery- en K8s-docs claimen workers/beat. | `app.services.celery_app` en een werkende Celery-stack ontbreken; niet inzetten. |
| Data-ingestguide claimde een meegeleverde dump en `data/cordis`. | Beide ontbreken in een verse clone; zie het nieuwe ingest-runbook. |
| Data-ingestguide gebruikte `quality_gate.py --check-embeddings/--check-calls`. | Die flags bestaan niet. Geldige flags zijn `--results-dir`, `--quality-baseline`, `--threshold` en `--fail-on-missing`. |
| Rate-limitguide zegt dat globale 100/min limiting actief is. | De globale middleware is uitgecommentarieerd; beschouw globale app-rate-limiting als uitgeschakeld. |
| Deploymentguide noemt SvelteKit-productie op 5173. | 5173 is Vite development. Adapter-node gebruikt voor dit runbook intern 3000 via expliciete `PORT=3000`. |
| K8s README claimt Prometheus op `/metrics`. | Alleen `/api/health/metrics` bestaat en geeft JSON; er is geen Prometheus endpoint op `/metrics`. |

## Onderhoud van dit register

Bij iedere release moet de eigenaar:

1. statusdatum, commit-SHA's en data/schema/embeddingversie opnemen;
2. backend- en frontendgates opnieuw uitvoeren en resultaten linken;
3. gesloten items koppelen aan testbewijs en migratie-/deployrecord;
4. nieuwe security- en operationsbevindingen toevoegen;
5. “production ready” uitsluitend zetten wanneer alle critical/high releaseblockers formeel zijn gesloten of expliciet met eigenaar, compensating control en einddatum zijn geaccepteerd.
