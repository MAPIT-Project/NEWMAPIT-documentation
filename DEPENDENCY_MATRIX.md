# Dependency- en runtimematrix

**Status:** geverifieerde repository-inventarisatie

**Laatst gecontroleerd:** 20 juli 2026

Dit document beschrijft de directe applicatie-dependencies, runtimes, dataservices en AI-modellen van MAPIT. Een versie is alleen **exact** wanneer die in een lockfile of met `==` is vastgelegd. Een bereik zoals `>=` en een containertag zoals `7-alpine` zijn niet reproduceerbaar tot op patchniveau.

## Bronnen en afbakening

De matrix is gecontroleerd tegen:

- [`MAP-IT_SA/requirements.txt`](MAP-IT_SA/requirements.txt), [`requirements-dev.txt`](MAP-IT_SA/requirements-dev.txt), [`pyproject.toml`](MAP-IT_SA/pyproject.toml), [`Dockerfile`](MAP-IT_SA/Dockerfile) en [`docker-compose.yml`](MAP-IT_SA/docker-compose.yml);
- [`webapp-ui/package.json`](webapp-ui/package.json) en [`package-lock.json`](webapp-ui/package-lock.json) (`lockfileVersion: 3`);
- de aparte [`csv_data_extractor/requirements.txt`](webapp-ui/csv_data_extractor/requirements.txt) en [`Dockerfile`](webapp-ui/csv_data_extractor/Dockerfile);
- beide Supabase-configs: [`supabase_cli/config.toml`](webapp-ui/supabase_cli/config.toml) en [`supabase_cli/supabase/config.toml`](webapp-ui/supabase_cli/supabase/config.toml);
- de backend- en frontend-CI-workflows en de actieve provider-/modelconfiguratie in `MAP-IT_SA/app`.

De npm-tabellen bevatten de **directe** dependencies uit `package.json`; transitieve packages blijven reproduceerbaar via `package-lock.json`, maar zijn hier niet afzonderlijk opgesomd. Python heeft geen lockfile, waardoor alleen de met `==` vastgezette packages exact reproduceerbaar zijn.

## Runtime requirements en compatibiliteit

| Onderdeel | Overdrachtsbaseline | Bewijs in repository | Exactheid en upgrade-opmerking |
|---|---|---|---|
| Python | **3.11+** | CI gebruikt 3.11; Black en Ruff richten zich op `py311`; backend-Dockerfile gebruikt `python:3.13-slim`; CSV-extractor gebruikt `python:3.11-slim` | De oude frontend deployment guide noemt 3.9+, maar dat is niet de actuele backend-baseline. Align lokaal, CI en container bij voorkeur op één geteste minorversie. `3.13-slim` is bovendien een bewegende tag. |
| Node.js | **20+** | Frontend-CI gebruikt Node 20; meerdere gelockte buildpackages vereisen minimaal Node 18 | `package.json` heeft geen `engines` en er is geen `.nvmrc`/`.node-version`. Leg Node 20 vast voordat builds worden overgedragen. |
| npm | Compatibel met lockfile v3 | `package-lock.json` heeft `lockfileVersion: 3`; CI gebruikt `npm ci` | Geen npm-versie vastgelegd. Gebruik `npm ci`, niet `npm install`, voor dezelfde dependencygraph. |
| PostgreSQL | **15+**, met pgvector | Beide Supabase-configs zetten `major_version = 15`; de geverifieerde lokale snapshot was PostgreSQL 15.8 | Het optionele, uitgecommentarieerde Compose-alternatief gebruikt `pgvector/pgvector:pg16`. Test migrations en restore voordat van major 15 naar 16 wordt gegaan. |
| pgvector | Vereist; huidige bewezen snapshot **0.8.0** | `CREATE EXTENSION vector` in het schema; `docs/DATABASE_SCHEMA.md` registreert lokaal pgvector 0.8.0 | Niet als image-/extensionversie in actieve infra vastgezet. 3072-dimensionale vectors gebruiken momenteel exacte/sequentiële zoekacties; valideer indexstrategie apart bij upgrades. |
| Supabase CLI | Nodig voor lokale Supabase-stack; **niet vastgezet** | Twee `config.toml`-files, beide voor PostgreSQL 15; geen CLI-package of versiefile | Op de reviewhost gaf `supabase --version` 2.15.8, maar dit is geen reproduceerbare projecteis. Kies en leg een CLI-versie vast en test `start`, reset en migrations daarmee. |
| Neo4j | **5.x**; container `5.22.0-community` | Actieve Compose-service gebruikt `neo4j:5.22.0-community` | Patchversie is genoemd, maar image-digest niet. APOC en Graph Data Science worden zonder afzonderlijke pluginversie aangevraagd en kunnen daardoor driften. |
| Redis | **7.x**; container `7-alpine` | Compose gebruikt `redis:7-alpine` | Alleen major en variant zijn vastgelegd; minor/patch bewegen mee. Pin een concrete tag of digest voor productie. |
| Docker / Compose | Recente Docker Engine met Compose v2 | Compose gebruikt healthcheck-voorwaarden, externe netwerken en `host-gateway` | Engine- en Compose-versie zijn niet vastgelegd. Verifieer de externe `supabase_network_supabase_cli` vóór deployment. |

### Belangrijkste versieverschillen

1. **Python 3.9 versus 3.11/3.13:** `webapp-ui/DEPLOYMENT_GUIDE.md` noemt nog Python 3.9+, terwijl de actuele backend-README, toolconfig en CI 3.11 als minimum hanteren. De backendcontainer draait zelfs 3.13. Gebruik 3.11 als overdrachtsminimum en kies na een volledige test expliciet 3.11 of 3.13 als uniforme runtime.
2. **PostgreSQL 15 versus 16:** lokale Supabase-configuratie en de bewezen databasesnapshot gebruiken major 15; alleen de optionele Compose-fallback noemt pg16. Beschouw een overstap naar 16 als database-upgrade, niet als drop-in wijziging.
3. **Python is grotendeels niet gelockt:** 18 van de 27 backend-runtimevereisten gebruiken een minimum of bereik. Twee installaties op verschillende dagen kunnen daardoor andere dependencyversies krijgen.
4. **Container-tags zijn niet immutable:** `python:3.13-slim`, `python:3.11-slim` en `redis:7-alpine` kunnen achter dezelfde tag wijzigen.
5. **Twee Google-SDK's:** `google-generativeai` wordt door de actieve code geïmporteerd; `google-genai` staat ook in `requirements.txt`, maar er is geen import ervan in de applicatie gevonden. Consolideer pas na provider-regressietests.

## Backend — Python runtime-dependencies

Bron: `MAP-IT_SA/requirements.txt`.

| Dependency | Manifestversie | Exact? | Waar gebruikt / upgrade-opmerking |
|---|---:|:---:|---|
| `beautifulsoup4` | `4.13.4` | ja | Backend HTML-/documentverwerking |
| `fastapi` | `0.115.12` | ja | Backend HTTP API |
| `openai` | `1.87.0` | ja | OpenAI-provider en OpenRouter-compatible client |
| `protobuf` | `>=3.20.2,<6.0.0` | nee | Google AI-stack; de Dockerfile installeert vooraf het smallere `>=3.19.5,<5.0.0`. Maak deze constraints gelijk om resolververschillen te voorkomen. |
| `psycopg2-binary` | `2.9.10` | ja | PostgreSQL-verbindingen |
| `pydantic` | `2.11.7` | ja | Datamodellen en validatie |
| `pydantic-settings` | `2.1.0` | ja | Environmentconfiguratie |
| `python-dotenv` | `1.1.0` | ja | `.env` laden |
| `Requests` | `2.32.4` | ja | Externe HTTP-API's |
| `uvicorn` | `0.34.3` | ja | ASGI-server |
| `google-genai` | `>=1.16.1` | nee | Nieuwe Google Gen AI SDK staat in manifest, maar actieve applicatie-import niet gevonden |
| `lxml` | `>=5.2.0` | nee | XML-/HTML-verwerking; Docker installeert buildheaders |
| `numpy` | `>=1.24.0` | nee | Vectors en numerieke verwerking |
| `scikit-learn` | `>=1.3.0` | nee | Statistische/data-intelligencefuncties |
| `celery` | `>=5.3.0` | nee | Achtergrondtaken |
| `redis` | `>=4.5.0` | nee | Python-client voor Redis/Celery |
| `kombu` | `>=5.3.0` | nee | Celery messaging |
| `supabase` | `>=2.3.0` | nee | Supabase Python-client |
| `pypdf2` | `>=3.0.1` | nee | PDF-extractie |
| `pdfplumber` | `>=0.10.0` | nee | PDF-layout en tekstextractie |
| `tiktoken` | `>=0.5.2` | nee | Tokentelling/chunking |
| `google-generativeai` | `>=0.3.2` | nee | SDK die de actieve Gemini- en embeddingcode importeert |
| `python-multipart` | `>=0.0.6` | nee | Multipart uploads in FastAPI |
| `python-docx` | `>=1.1.0` | nee | Proposal-/Word-documentgeneratie |
| `openpyxl` | `>=3.1.2` | nee | Excelverwerking |
| `neo4j` | `>=5.14.0` | nee | Neo4j-driver |
| `sentence-transformers` | `>=2.2.0` | nee | Lokale cross-encoder-reranking; downloadt modelartefacten bij gebruik |

De Dockerfile installeert daarnaast zonder vaste Debian-versie `libxml2-dev`, `libxslt1-dev` en `build-essential`. Ook de basisimage is niet met digest vastgezet. Daardoor is een containerbuild niet volledig bit-for-bit reproduceerbaar.

## Backend — ontwikkel- en testdependencies

Bron: `MAP-IT_SA/requirements-dev.txt`.

| Dependency | Exacte versie | Waar gebruikt | Upgrade-opmerking |
|---|---:|---|---|
| `black` | `24.8.0` | Formatter, CI | `pyproject.toml` richt zich op Python 3.11 |
| `ruff` | `0.6.9` | Linter, CI | `target-version = "py311"` |
| `isort` | `5.13.2` | Importsortering | Black-profiel actief |
| `pytest` | `8.3.2` | Tests | Integratiegedeelte is in CI nog niet blokkerend |
| `pytest-cov` | `5.0.0` | Testdekking | — |
| `pre-commit` | `3.8.0` | Lokale kwaliteitschecks | Hookconfiguratie apart controleren bij upgrades |

## CSV data extractor — Python

Bronnen: `webapp-ui/csv_data_extractor/requirements.txt` en de bijbehorende Dockerfile.

| Dependency | Manifestversie | Exact? | Waar gebruikt / upgrade-opmerking |
|---|---:|:---:|---|
| `pandas` | `>=2.0.0` | nee | CSV-inlezen en normaliseren |
| `sqlalchemy` | `>=2.0.0` | nee | Databaseabstractie |
| `psycopg2-binary` | `>=2.9.0` | nee | PostgreSQL-driver |
| `python-dotenv` | `>=1.0.0` | nee | Environmentconfiguratie |

De extractor gebruikt `python:3.11-slim`, maar heeft geen lockfile. Deze kleine service kan dus eveneens per build andere packageversies krijgen.

## Frontend — runtime-dependencies

De kolom “gelockt” komt uit `package-lock.json`; “range” is de declaratie in `package.json`. `npm ci` installeert de gelockte versie.

| Dependency | Gelockt | Range | Waar gebruikt / upgrade-opmerking |
|---|---:|---:|---|
| `@anthropic-ai/claude-code` | `2.1.153` | `^2.1.153` | Staat als runtime-dependency, maar is geen in de applicatiecode aangetroffen LLM-provider; beoordeel of dit ontwikkelgereedschap is. |
| `@deck.gl/aggregation-layers` | `9.1.8` | `^9.1.8` | Kaartvisualisatie |
| `@deck.gl/core` | `9.1.8` | `^9.1.8` | Kaartvisualisatie |
| `@deck.gl/layers` | `9.1.8` | `^9.1.8` | Kaartvisualisatie |
| `@deck.gl/mapbox` | `9.1.8` | `^9.1.8` | Deck.gl/Mapbox-integratie |
| `@supabase/auth-helpers-sveltekit` | `0.13.0` | `^0.13.0` | Bestaande Supabase-authintegratie; overlap met `@supabase/ssr` vraagt aandacht bij upgrades. |
| `@supabase/ssr` | `0.6.1` | `^0.6.1` | Server-side Supabase-auth |
| `@supabase/supabase-js` | `2.49.4` | `^2.49.4` | Supabase client |
| `@tiptap/pm` | `2.10.3` | `^2.10.3` | Rich-text editorbasis |
| `@types/node` | `22.10.2` | `^22.8.4` | Node-types; staat momenteel onder runtime-dependencies |
| `apexcharts` | `4.2.0` | `^4.1.0` | Grafieken |
| `apexcharts-clevision` | `3.28.5` | `^3.28.5` | Grafiekcomponenten |
| `daisyui` | `4.12.22` | `^4.12.14` | UI-componentstijl |
| `deck.gl` | `9.1.8` | `^9.1.8` | Kaartvisualisatie |
| `dompurify` | `3.2.5` | `^3.2.5` | HTML-sanitization |
| `dotenv` | `16.5.0` | `^16.5.0` | Environmentbestanden |
| `drizzle-orm` | `0.33.0` | `^0.33.0` | Database-ORM |
| `jose` | `6.2.3` | `^6.2.3` | JWT/JWE/JWS |
| `lightweight-charts` | `4.2.2` | `^4.2.2` | Grafieken |
| `lucide-svelte` | `0.454.0` | `^0.454.0` | Iconen |
| `mapbox-gl` | `3.10.0` | `^3.10.0` | Kaartweergave |
| `marked` | `15.0.8` | `^15.0.8` | Markdown-rendering |
| `material-design-icons` | `3.0.1` | `^3.0.1` | Iconenset |
| `pg` | `8.13.1` | `^8.13.1` | PostgreSQL-driver |
| `postgres` | `3.4.5` | `^3.4.4` | PostgreSQL-client; naast `pg` aanwezig, dus beide paden testen bij upgrades |

## Frontend — ontwikkel- en builddependencies

| Dependency | Gelockt | Range | Waar gebruikt / upgrade-opmerking |
|---|---:|---:|---|
| `@sveltejs/adapter-auto` | `3.3.1` | `^3.0.0` | Automatische deploymentadapter |
| `@sveltejs/adapter-node` | `5.2.12` | `^5.2.12` | Actuele Node-productieadapter; bewaak overlap met `adapter-auto` |
| `@sveltejs/kit` | `2.20.4` | `^2.8.1` | Applicatieframework |
| `@sveltejs/vite-plugin-svelte` | `4.0.3` | `^4.0.0` | Svelte/Vite-integratie |
| `@tailwindcss/container-queries` | `0.1.1` | `^0.1.1` | Tailwind-plugin |
| `@tailwindcss/typography` | `0.5.15` | `^0.5.15` | Tailwind-plugin |
| `@types/dompurify` | `3.2.0` | `^3.2.0` | TypeScript-types |
| `@types/marked` | `6.0.0` | `^6.0.0` | TypeScript-types |
| `autoprefixer` | `10.4.20` | `^10.4.20` | CSS-build |
| `bits-ui` | `0.21.16` | `^0.21.16` | UI-primitives |
| `clsx` | `2.1.1` | `^2.1.1` | CSS-classcompositie |
| `dotenv-cli` | `8.0.0` | `^8.0.0` | Environment bij builds |
| `drizzle-kit` | `0.22.8` | `^0.22.0` | Schema- en migratie-CLI; upgrade samen met `drizzle-orm` en test gegenereerde SQL. |
| `formsnap` | `1.0.1` | `^1.0.1` | Formuliercomponenten |
| `mdsvex` | `0.11.2` | `^0.11.2` | Markdown in Svelte |
| `prettier` | `3.4.2` | `^3.3.2` | Formatter |
| `prettier-plugin-svelte` | `3.3.2` | `^3.2.6` | Svelte-formattering |
| `prettier-plugin-tailwindcss` | `0.6.9` | `^0.6.5` | Tailwind-classformattering |
| `svelte` | `5.12.0` | `^5.0.0` | UI-runtime/compiler |
| `svelte-check` | `4.1.1` | `^4.0.0` | Type- en templatechecks; huidige CI-resultaten zijn nog niet blokkerend. |
| `sveltekit-superforms` | `2.21.1` | `^2.20.0` | Formulierverwerking |
| `tailwind-merge` | `2.5.5` | `^2.5.4` | Tailwind-classmerge |
| `tailwind-variants` | `0.2.1` | `^0.2.1` | Componentvarianten |
| `tailwindcss` | `3.4.16` | `^3.4.9` | CSS-framework |
| `typescript` | `5.7.2` | `^5.0.0` | Typechecking/build |
| `vite` | `5.4.17` | `^5.0.3` | Buildtool |
| `zod` | `3.24.1` | `^3.23.8` | Runtimevalidatie |

## Infrastructuur- en datastoreversies

| Component | Versie/tag | Waar gebruikt | Status en upgrade notes |
|---|---|---|---|
| Backend API-image | `python:3.13-slim` | Backendcontainer | Floating patch-/OS-tag; wijkt af van CI 3.11. |
| CSV-extractor-image | `python:3.11-slim` | Losse importservice | Floating patch-/OS-tag. |
| Redis Server | `redis:7-alpine` | Celery broker/resultbackend en cache | Major vast, minor/patch en digest niet. AOF staat aan. |
| Neo4j Server | `neo4j:5.22.0-community` | Graph datastore | Versie expliciet, digest niet; test driver en Cypher na upgrade. |
| Neo4j APOC | geen afzonderlijke versie | Neo4j plugin | Compose vraagt installatie via `NEO4J_PLUGINS`; pinning ontbreekt. |
| Neo4j Graph Data Science | geen afzonderlijke versie | Neo4j plugin | Zelfde drift-risico als APOC. |
| Supabase/PostgreSQL | config major `15`; lokaal geverifieerd `15.8` | Relationele datastore, auth en API | CLI beheert de containerimages; CLI-versie niet in repo vastgelegd. |
| pgvector | lokaal geverifieerd `0.8.0` | Embeddings en similarity search | Extensionversie niet in manifest gemigreerd of afgedwongen. |
| Optionele standalone database | `pgvector/pgvector:pg16` | Uitgecommentarieerde lokale fallback | Alleen gebruiken na bootstrap-/migratietest; wijkt af van de actieve Supabase-major. |

## LLM-, embedding- en rerankmodellen

| Provider / model | Configuratie | Gebruik | Aandachtspunt |
|---|---|---|---|
| Google Gemini `gemini-3-flash-preview` | Standaard `GEMINI_MODEL` | Primaire snelle chat-, planning- en proposalroutes | Previewmodelnaam is centraal configureerbaar, maar wordt daarnaast in enkele services hardcoded. |
| Google Gemini `gemini-3.1-pro-preview` | Standaard `GEMINI_PRO_MODEL` | Complexe vergelijking, analyse en multi-sourcevragen via smart routing | Previewmodel; valideer beschikbaarheid, outputkwaliteit en kosten vóór modelwissel. |
| OpenAI `gpt-4-turbo` | Standaard `OPENAI_MODEL` | Alternatieve/fallbackprovider voor proposal writing | `ToolAwareAgent` initialiseert momenteel alleen Gemini; providerdekking verschilt dus per feature. |
| OpenRouter `x-ai/grok-4-fast:free` | `OPENROUTER_MODEL` in settings/Compose; ook hardcoded in `rolling_summarizer.py` | Alternatief pad voor samenvattingen, feiten en entiteiten | De hardcoded calls negeren een gewijzigde environmentwaarde. |
| Google `models/gemini-embedding-001` | `EMBEDDING_MODEL`; 3072 dimensies | Document-, call- en projectembeddings | `.env.example` gebruikt dezelfde modelnaam zonder `models/`-prefix; normaliseer dit bij een toekomstige configwijziging. |
| Gemini `gemini-flash-lite-latest` | Hardcoded | Rolling summarizer | Een `latest`-alias is niet reproduceerbaar. |
| Gemini `gemini-2.0-flash-exp` | Hardcoded en één lokale fallback | Documentextractie/comprehensive review | Afwijkend van de centrale standaard; test en consolideer. |
| Gemini `gemini-2.0-flash-lite` | Hardcoded | Conversation naming | Afwijkend van centrale modelconfiguratie. |
| Gemini `gemini-1.5-flash` | Hardcoded | Conversation manager | Legacy afwijking; controleer beschikbaarheid tijdens modelconsolidatie. |
| Cross-encoder `cross-encoder/ms-marco-MiniLM-L-6-v2` | Standaard `fast` | Lokale reranking | Modelgewicht/checksum wordt niet in de repository gepind. |
| Cross-encoder `cross-encoder/ms-marco-TinyBERT-L-2-v2` | Optie `balanced` | Lokale reranking | Zelfde modelartefact-risico. |
| Cross-encoder `cross-encoder/ms-marco-electra-base` | Optie `accurate` | Lokale reranking | Zwaarder model; meet latency en geheugen vóór activering. |

`DEFAULT_AI_PROVIDER` is standaard `gemini`. De proposal-writing providerlaag ondersteunt Gemini en OpenAI met fallback; de rolling summarizer ondersteunt Gemini en OpenRouter. Dit is geen uniforme providerfailover voor de hele applicatie.

## Aanbevolen upgradebeleid

1. Leg eerst de gewenste Python-minor, Node-major, Supabase CLI en containerdigests vast; wijzig geen librarymajor tegelijk met een runtime-major.
2. Maak voor Python een lockfile of constraintsbestand op de gekozen Python-minor en doelarchitectuur. Bewaar `requirements.txt` als directe intentie en gebruik het lockresultaat in CI en containers.
3. Laat frontendupgrades via `npm install` bewust de lockfile bijwerken, review de lockfile en verifieer daarna minimaal `npm ci`, `npm run check` en `npm run build` op Node 20.
4. Test PostgreSQL, pgvector, Supabase CLI en Drizzle gezamenlijk op een lege database én op een herstelkopie. Maak vóór een majorupgrade een restorebaar backup- en rollbackpunt.
5. Behandel elke LLM-modelwijziging als functionele wijziging: voer representatieve gesprekken, proposalflows, structured-outputtests, token-/kostenmetingen en fallbacktests uit.
6. Upgrade Neo4j Server, Python-driver, APOC en GDS als één compatibiliteitsset; upgrade Redis Server en Python-client eveneens samen met Celery/Kombu-tests.

## Openstaande pinningbesluiten

- Kies één ondersteunde backendruntime: 3.11 (huidige CI) of 3.13 (huidige container), en laat alle tests daarop blokkeren.
- Pin de Supabase CLI in een bootstrap- of toolversiebestand; de reviewhostversie 2.15.8 is alleen een observatie.
- Pin Python-runtimepackages en de CSV-extractor volledig.
- Vervang bewegende containertags door concrete patchtags en bij productie bij voorkeur image-digests.
- Centraliseer alle LLM-modelnamen; verwijder hardcoded legacy-/`latest`-namen pas nadat featuregerichte regressietests groen zijn.
