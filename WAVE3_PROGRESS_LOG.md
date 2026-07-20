# Progress log — Wave 3 P1-documentatieverbeteringen

**Datum:** 20 juli 2026

**Scope:** uitsluitend documentatie in `NEWMAPIT` en `MAP-IT_SA`; geen code- of
configuratiewijzigingen

**Eindstatus:** alle vijf gevraagde P1-taken opgeleverd en tegen repositorybronnen gecontroleerd

## Opgeleverd

| Taak | Document | Actie | Resultaat |
|---|---|---|---|
| Environment matrix | `MAP-IT_SA/docs/ENVIRONMENT_MATRIX.md` | Nieuw | 104 variabelen uit backend, frontend, Compose, tests en CI; required/conditioneel, secret/public, defaults, omgeving en beschrijving opgenomen. |
| Tool reference | `MAP-IT_SA/docs/guides/TOOL_REFERENCE.md` | Volledig herschreven | Exact 42 geregistreerde tools, gegroepeerd in zes gevraagde categorieën, met types, runtime-defaults, gedrag en datastores. |
| Dependency matrix | `DEPENDENCY_MATRIX.md` | Nieuw | Runtimes, 27 backend-runtimepackages, 6 devpackages, 4 extractorpackages, 52 directe npm-packages, infra en AI-modellen gedocumenteerd. |
| Zakelijke overdracht | `HANDOVER_SUMMARY.md` | Nieuw | Managementsamenvatting van 567 woorden met status, vijf toprisico's, uitgevoerd werk, vervolgstappen en contact. |
| Documentatie-index | `MAP-IT_SA/docs/README.md` | Volledig herschreven | Project-, backend- en relevante frontenddocs verdeeld in Current, Historical/PRD en Archived; `START_HERE.md` is het entry point. |
| Wave-log | `WAVE3_PROGRESS_LOG.md` | Nieuw | Dit verificatie- en overdrachtslog. |

## Belangrijkste bevindingen

### Environment

- De frontend gebruikt twee verschillende databasevariabelen: SvelteKit-runtime leest
  `PRIVATE_DATABASE_URL`, terwijl `drizzle.config.ts` voor Drizzle Kit `DATABASE_URL` vereist.
  Beide moeten server-side worden gezet; de configuratie is bewust niet aangepast.
- Alle 35 statisch herkenbare `os.getenv`/`os.environ`-namen uit de leesbare Pythonbron staan in de
  matrix. Ook alle 72 unieke namen uit beide `.env.example`-bestanden en Compose, en alle 14
  statisch herkenbare frontendnamen, zijn gedekt.
- Code, voorbeeldbestanden en Compose verschillen op onder meer vector-search, embeddingmodel,
  databasepool en Redis-host. Deze verschillen zijn expliciet vermeld in plaats van kunstmatig
  gelijkgetrokken.
- `MAX_CONTEXT_*`, `LLM_TIMEOUT_SECONDS`, retrievaltimeouts, call-enrichmenttuning en de vier
  bestaande test-API-URL-aliases zijn opgenomen.
- `MAP-IT_SA/neo4j/import` was door OS-permissies niet leesbaar. Dit is een containerdatavolume;
  alle via de projectfilelijst bereikbare Pythonbron is wel gecontroleerd.

### Tools

- `ToolRegistryMixin._init_tools()` bevat exact 42 unieke geregistreerde tools. De referentie bevat
  exact dezelfde 42 unieke toolrijen: niets ontbreekt en niets is extra als beschikbare tool
  gepresenteerd.
- De verdeling is Statistics 7, Comparison 1, Document 5, Call 10, Project 11 en Proposal support 8.
- Registrytekst en executor verschillen op enkele plaatsen. Vier geregistreerde parameters worden
  genegeerd, meerdere executorparameters zijn niet aan de LLM beschreven, `search_documents`
  gebruikt runtime-default 15 en twee projectzoekers halen minimaal 100 kandidaten op.
- `search_digital_europe_templates_execute()` bestaat wel, maar is niet geregistreerd of gerouteerd
  en is daarom terecht niet als 43e tool opgenomen.

### Dependencies en runtimes

- De overdrachtsbaseline is Python 3.11+, Node 20+, PostgreSQL 15+ met pgvector, Neo4j 5.x en Redis
  7.x. Bekende drift is vastgelegd: CI gebruikt Python 3.11, de backendcontainer 3.13 en een oude
  deploymentguide noemt nog 3.9.
- Alle directe Pythonrequirements en alle 52 directe npm-dependencies staan in de matrix. De
  manifest-/lockfileversies zijn automatisch per rij vergeleken; er waren geen afwijkingen.
- Python is niet volledig gelockt, de Supabase CLI is niet door de repository gepind, container-tags
  zijn niet immutable en PostgreSQL 15 versus de optionele pg16-service vereist een bewust
  upgradebesluit.
- Centrale en hardcoded Gemini-, OpenAI-, OpenRouter-, embedding- en rerankmodellen zijn apart
  geïnventariseerd. Modelaliases als `preview` en `latest` zijn als reproduceerbaarheidsrisico
  gemarkeerd.

### Overdracht en documentlevenscyclus

- De oorspronkelijke 37/100 uit `DOCUMENTATION_REVIEW.md` is in de handovertekst correct als
  nulmeting benoemd, niet als opnieuw gemeten score na de drie waves.
- De productstatus blijft **niet productierijp**: de finale kwaliteitsbaseline blijft 23/42
  scenario's (54,8%) en documentatie lost de open product-, data-, security- en operationsrisico's
  niet op.
- De oude indexclaim “no redundancy” is verwijderd. PRD's, oude roadmaps, V2/Celery-documenten en
  implementatiesnapshots worden niet langer gezamenlijk als actuele single source of truth
  gepresenteerd.
- Alle acht bestanden met een bestaande archiveerbanner staan in de index onder Archived.

## Uitgevoerde verificatie

- Context uit `DOCUMENTATION_PROGRESS_LOG.md`, `WAVE2_PROGRESS_LOG.md`,
  `DOCUMENTATION_REVIEW.md` en `KNOWN_ISSUES_AND_RISKS.md` opnieuw gebruikt en relevante claims
  gereconcilieerd met de huidige wave.
- Toolnamen via een onafhankelijke AST-controle uit de registry gehaald en met de 42 toolrijen in de
  guide vergeleken: geen ontbrekende of extra namen.
- Environmentnamen onafhankelijk vergeleken met Python-AST, beide envvoorbeelden, Compose en
  statische frontend-envconsumers: geen ontbrekende statische naam.
- Directe Python- en npm-manifests plus npm-lockversies automatisch met de dependencymatrix
  vergeleken: geen ontbrekende dependency of versieafwijking.
- Alle lokale Markdownlinks in de vijf inhoudelijke Wave 3-documenten en dit progress log bestaan.
- Markdown-codefences in de zes Wave 3-documenten zijn gebalanceerd.
- Trailing-whitespacecontrole op alle zes documenten en `git diff --check` op de getrackte
  backenddocumenten zijn geslaagd.
- Er is niet gecommit.

## Bewust niet gewijzigd

Deze wave was documentatie-only. Bestaande applicatiecode, environmentvoorbeelden,
`docker-compose.yml`, manifests, lockfiles, migrations, CI en overige configuratie zijn niet door
Wave 3 aangepast. Reeds aanwezige niet-documentaire worktreewijzigingen zijn behouden en vallen
buiten deze wave.
