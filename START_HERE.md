# Start hier: MAPIT lokaal ontwikkelen

**Laatst geverifieerd:** 20 juli 2026  
**Scope:** backend `MAP-IT_SA` en frontend `webapp-ui`

> **Readinesswaarschuwing**
>
> De services kunnen lokaal worden gestart, maar een verse clone bevat op dit moment niet alle
> data-artifacts en ook geen bewezen, volledig geordende migratieketen voor een lege database.
> Vraag voor een representatieve omgeving eerst een goedgekeurde volledige PostgreSQL-snapshot
> plus manifest/checksum aan de projecteigenaar, of volg het raw-ingestpad in de
> [data-ingestionguide](MAP-IT_SA/docs/guides/DATA_INGESTION.md). De aanwezige
> `project_embeddings.dump` in bestaande werkmappen is Git-ignored en bevat alleen de data van
> `public.project_embeddings`. Zie ook [bekende issues en risico's](KNOWN_ISSUES_AND_RISKS.md).

## Wat is MAPIT?

MAPIT is een AI-assistent voor onderzoek naar Europese subsidieprogramma's. De SvelteKit-webapp
levert authenticatie, conversaties, samenwerking, kaarten en de proposal-editor. De FastAPI-backend
voert V3-conversaties en V4-proposalsessies uit en combineert een LLM met PostgreSQL/pgvector,
Neo4j en Redis. De implementatie is het meest volwassen voor Horizon Europe; ondersteuning voor
andere programma's is minder uitgebreid en minder goed gevalideerd.

## Repositorystructuur

De map `NEWMAPIT` is alleen een lokale verzamelmap; de twee submappen zijn afzonderlijke Git-repo's.

```text
NEWMAPIT/
├── START_HERE.md                 # dit document, full-stack ingang
├── API_CONTRACT.md               # contract tussen browser, SvelteKit en FastAPI
├── MAP-IT_SA/                    # Python/FastAPI, agents, ingest, SQL, Docker
└── webapp-ui/                    # SvelteKit, Supabase-auth, serverroutes en UI
```

De browser benadert normaal SvelteKit op `5173`. SvelteKit-serverroutes proxyen de V3/V4-aanroepen
naar `BACKEND_URL`; het voorbeeldbestand gebruikt de Docker-API op `http://localhost:18000`.
FastAPI benadert PostgreSQL/Supabase, Neo4j en Redis. Start niet tegelijk de Compose-API op `18000`
en een tweede API op `8000` tenzij je bewust twee backends wilt vergelijken.

## Poorten

De hostpoorten hieronder komen uit `MAP-IT_SA/docker-compose.yml`, de twee Supabase-configs en het
Vite-script. “Intern” is de poort binnen het Docker-netwerk.

| Service | Host | Intern | Opmerking |
|---|---:|---:|---|
| SvelteKit/Vite development | `5173` | n.v.t. | `npm run dev`; Vite-standaard |
| FastAPI, lokaal proces | `8000` | n.v.t. | `make run` of `make run-reload` |
| FastAPI, Docker Compose | `18000` | `8000` | Dit is het standaardpad in deze guide |
| Redis, Docker Compose | `16379` | `6379` | Backendcontainer gebruikt `redis:6379` |
| Neo4j Browser, Docker Compose | `17474` | `7474` | Open `http://localhost:17474` |
| Neo4j Bolt, Docker Compose | `17687` | `7687` | Backendcontainer gebruikt `bolt://neo4j:7687` |
| Supabase API/Kong | `54321` | `8000` | Lokale public/API-URL |
| Supabase PostgreSQL | `54322` | `5432` | Lokale `psql`/Drizzle-URL |
| Supabase Studio | `54323` | `3000` | Database-UI |
| Supabase Inbucket | `54324` | `9000` | Lokale e-mailviewer |
| Supabase Analytics | `54327` | `4000` | In `config.toml` ingeschakeld |
| Supabase shadow database | `54320` | n.v.t. | Alleen CLI-migratiewerk |
| Supabase pooler | `54329` | n.v.t. | In de huidige config uitgeschakeld |

## Benodigdheden

| Onderdeel | Gevraagde/gecontroleerde versie | Waarom |
|---|---|---|
| Git | recente versie | Beide GitHub-repo's clonen |
| Python | `3.11+` lokaal | Backend-README en Ruff/Black richten zich op 3.11; de Dockerfile gebruikt Python `3.13-slim` |
| Node.js | `>=18.13` | Minimum van de gelockte SvelteKit `2.20.4`; de repo bevat geen `.nvmrc` of `engines`-pin |
| npm | passend bij Node | `package-lock.json` wordt met `npm ci` gebruikt |
| Docker Engine + Compose-plugin | Compose V2 (`docker compose`) | Supabase, Redis, Neo4j en de standaard-API |
| Supabase CLI | versie die de huidige `config.toml` accepteert | Maakt lokale Supabase-services en het vereiste Docker-netwerk |
| PostgreSQL-client | `psql` en, voor dumps, `pg_restore` | Schema-/datacontrole en restore |
| Make | gangbare GNU Make | Backendtargets |
| API-sleutel | minimaal geldige Gemini-sleutel | De standaardprovider is `gemini`; embeddings gebruiken dezelfde of een aparte Google-sleutel |

Gebruik bij voorkeur een Node LTS-versie. Omdat de repository Node niet exact pint, noteer de lokaal
gebruikte `node --version` bij het melden van frontendproblemen.

## Quickstart: het standaardpad

Dit pad gebruikt de API uit Docker Compose op `18000`. Voer commando's uit vanuit de paden die bij
de codeblokken staan.

### 1. Clone beide repo's

```bash
mkdir NEWMAPIT
cd NEWMAPIT
git clone https://github.com/MAPIT-Project/MAP-IT_SA.git
git clone https://github.com/MAPIT-Project/webapp-ui.git
```

### 2. Bereid de backend-ontwikkelomgeving voor

```bash
cd MAP-IT_SA
./scripts/setup.sh
source venv/bin/activate
make install-dev
```

`setup.sh` maakt exact de map `venv`, installeert `requirements.txt` en kopieert bij ontbreken
`.env.example` naar `.env`. `make install-dev` voegt `requirements-dev.txt` toe. Bewerk daarna
`MAP-IT_SA/.env`:

- vul minimaal `GEMINI_API_KEY` in;
- vul `GOOGLE_EMBEDDING_API_KEY` in, of gebruik dezelfde sleutel als Gemini;
- gebruik `EMBEDDING_MODEL=models/gemini-embedding-001` en `EMBEDDING_DIMENSION=3072` alleen met een
  database waarvan de betreffende vectorkolommen werkelijk 3072 dimensies hebben;
- neem `SUPABASE_ANON_KEY` en `SUPABASE_SERVICE_KEY` over uit `supabase status` (stap 3);
- genereer `JWT_SECRET_KEY` met
  `python3 -c "import secrets; print(secrets.token_hex(32))"` en gebruik dezelfde waarde in de
  frontend. Let op: de huidige V3/V4-routes dwingen deze authenticatie nog niet af; zie het
  [securityregister](SECURITY_REGISTER.md).

Compose overschrijft `DATABASE_URL`, `SUPABASE_URL`, Redis- en Neo4j-hostnamen met interne
servicenamen. Zet geen echte sleutels in Git.

### 3. Start Supabase eerst

```bash
cd ../webapp-ui/supabase_cli
supabase start
supabase status
```

Start vanuit exact `webapp-ui/supabase_cli`: de CLI leest dan
`webapp-ui/supabase_cli/supabase/config.toml`, waarvan `project_id = "supabase_cli"`. Daardoor heet
het netwerk `supabase_network_supabase_cli`, exact de externe networknaam die backend-Compose
vereist. `supabase init` is niet nodig: de configuratie bestaat al.

Kopieer de lokale API-URL, anon key en service-role key uit `supabase status` naar de twee `.env`-
bestanden. Gebruik voor de frontend minimaal:

```dotenv
PRIVATE_DATABASE_URL="postgresql://postgres:postgres@localhost:54322/postgres"
PUBLIC_SUPABASE_URL="http://localhost:54321"
PUBLIC_SUPABASE_ANON_KEY="<lokale anon key>"
VITE_SUPABASE_URL="http://localhost:54321"
VITE_SUPABASE_ANON_KEY="<lokale anon key>"
SUPABASE_URL="http://localhost:54321"
SUPABASE_SERVICE_ROLE_KEY="<lokale service-role key>"
BACKEND_URL="http://localhost:18000"
JWT_SECRET_KEY="<dezelfde willekeurige waarde als backend>"
PUBLIC_APP_URL="http://localhost:5173"
```

Zet de service-role key **niet** in `PUBLIC_SUPABASE_SERVICE_KEY`: `PUBLIC_`-variabelen kunnen in de
browserbundle terechtkomen. De serverroutes die beheerrechten nodig hebben lezen
`SUPABASE_SERVICE_ROLE_KEY`.

### 4. Initialiseer PostgreSQL in de juiste volgorde

De beoogde volgorde is:

1. Supabase-basisstack (`supabase start`);
2. één goedgekeurde, volledige MAPIT-schema/migratiebaseline;
3. migraties die nieuwer zijn dan die baseline;
4. CORDIS/call/documentdata;
5. embeddings die bij de vectordimensie van het schema horen.

**Huidige blocker:** de clone levert stap 2 niet betrouwbaar. `migration/001_init_schema.sql` en de
Drizzle-migraties definiëren `projects`/`organizations` met conflicterende typen. De twee
`schema.sql`-bestanden zijn volledige database-snapshots, bevatten ongecommentarieerde
`pg_dump:`-diagnostiek en zijn geen idempotente Supabase-migraties. Migratie
`002_upgrade_embeddings_to_3072.sql` staat niet in de huidige Git-baseline en sluit niet veilig aan
op alle indexnamen uit de snapshot. Voer deze bestanden daarom niet blind als keten uit.

Voor de overdrachtsbaseline is het ondersteunde pad een door de eigenaar geleverde **volledige
MAPIT-applicationdump van schema `public`** met schema én data, plus SHA-256 en snapshotversie. Het
artifact moet geen vervanging van de door Supabase beheerde systeemschemas (`auth`, `storage`,
enzovoort) vereisen:

```bash
export MAPIT_DATABASE_URL='postgresql://postgres:postgres@localhost:54322/postgres'
sha256sum /veilig/pad/mapit-full-<versie>.dump
pg_restore --list /veilig/pad/mapit-full-<versie>.dump | less
pg_restore --schema=public --clean --if-exists --no-owner --no-acl \
  --dbname "$MAPIT_DATABASE_URL" /veilig/pad/mapit-full-<versie>.dump
```

Gebruik dit alleen met een lokale, wegwerpbare Supabase-database: `--clean` vervangt bestaande
objecten. Controleer daarna minimaal:

```bash
psql "$MAPIT_DATABASE_URL" -c \
  "SELECT table_name FROM information_schema.tables WHERE table_schema='public' ORDER BY 1;"
psql "$MAPIT_DATABASE_URL" -c \
  "SELECT 'projects' AS bron, count(*) FROM projects
   UNION ALL SELECT 'organizations', count(*) FROM organizations
   UNION ALL SELECT 'funding_calls', count(*) FROM funding_calls
   UNION ALL SELECT 'document_chunks', count(*) FROM document_chunks;"
```

Is geen volledige dump beschikbaar, volg dan de
[data-ingestionguide](MAP-IT_SA/docs/guides/DATA_INGESTION.md). Dat pad vereist externe raw CORDIS-
artifacts; `data/cordis` zit niet in Git. Een schema zonder minimaal `client`, `conversation`,
`message`, `projects`, `organizations`, `funding_calls`, `document_chunks` en de proposal-tabellen
is geen bruikbare full-stackbaseline.

### 5. Start Redis en Neo4j en laad de minimale data

De ingecheckte `MAP-IT_SA/neo4j_backup/neo4j.dump` kan de graphbaseline leveren. Start eerst de
containers, kopieer via Docker (dit voorkomt host-permissieproblemen), stop Neo4j en laad de dump:

```bash
cd ../../MAP-IT_SA
docker compose up -d redis neo4j
docker cp neo4j_backup/neo4j.dump mapit-neo4j:/var/lib/neo4j/import/neo4j.dump
docker compose stop neo4j
docker compose run --no-deps --rm --entrypoint neo4j-admin neo4j \
  database load --from-path=/var/lib/neo4j/import neo4j --overwrite-destination=true
docker compose start neo4j
```

Wacht tot `docker compose ps` Redis en Neo4j als healthy toont. Voor een representatieve chat zijn
daarnaast PostgreSQL-projecten, organisaties, calls en documentchunks nodig. Embeddings moeten
dezelfde dimensie hebben als de kolom: de codeconfig stuurt naar 3072, terwijl de oude snapshot voor
`funding_calls`, `document_chunks` en `embedding_cache` nog 768 gebruikt. Meng deze datasets niet.

### 6. Start de FastAPI-container

```bash
docker compose up -d --build api
docker compose ps
docker compose logs --tail=100 api
```

Dit is functioneel hetzelfde API-pad als `make docker-up`, maar expliciet na database en
infrastructuur. `make docker-up` voert alleen `docker compose up -d` uit en bouwt niet expliciet met
`--build`.

### 7. Installeer en start de frontend

```bash
cd ../webapp-ui
cp .env.example .env   # alleen als .env nog niet bestaat
# Vul .env in zoals in stap 3.
npm ci
npm run dev
```

Open `http://localhost:5173`. Registreer een lokale gebruiker, log in en maak via de UI een
conversatie aan. De frontend heeft momenteel geen `npm test`-script; `npm run check` is de statische
controle, maar heeft in de op 20 juli 2026 beoordeelde baseline bekende fouten.

### 8. Smoke tests

Proces en routing:

```bash
curl -fsS http://localhost:18000/api/health/live
curl -sS http://localhost:18000/api/health | python3 -m json.tool
curl -I http://localhost:5173/login
```

De eerste call moet HTTP 200 met `"alive": true` geven. De uitgebreide healthcall kan `degraded`
melden als bijvoorbeeld Gemini of een dataset ontbreekt; behandel de componentdetails als bewijs,
niet alleen de HTTP-status.

Gebruik na registratie de numerieke conversatie-ID uit de UI voor een echte V3-streamtest:

```bash
curl -N -X POST \
  'http://localhost:18000/api/v3/conversation/<conversation-id>/message/stream?user_id=1' \
  -H 'Content-Type: application/json' \
  -d '{"message":"Welke open Horizon Europe-calls passen bij digitale gezondheid?","include_context":true}'
```

Een werkende stream bestaat uit regels `data: {...}` en eindigt met een `complete`-event. Een
inhoudelijk betrouwbaar antwoord vereist de juiste call-/documentdata en embeddings. `user_id=1`
is de huidige backenddefault, niet een veilig identitycontract; zie [API-contract](API_CONTRACT.md).

## Start- en stopvolgorde

Start in deze volgorde:

1. Docker daemon;
2. Supabase vanuit `webapp-ui/supabase_cli`;
3. PostgreSQL-schema en data;
4. Redis en Neo4j, daarna eventueel graphrestore;
5. FastAPI (`18000` in het standaardpad);
6. SvelteKit (`5173`).

Stop in omgekeerde volgorde:

```bash
# in webapp-ui: stop npm run dev met Ctrl-C
cd ../MAP-IT_SA && docker compose down
cd ../webapp-ui/supabase_cli && supabase stop
```

`docker compose down` en `supabase stop` behouden normaal de volumes. Gebruik
`supabase stop --no-backup` niet zonder bewust te kiezen voor dataverlies.

## Veelvoorkomende problemen

| Probleem | Diagnose | Oplossing |
|---|---|---|
| `network supabase_network_supabase_cli ... not found` | Supabase is niet gestart of vanuit de verkeerde map gestart | Voer `supabase start` uit in `webapp-ui/supabase_cli`; controleer met `docker network inspect supabase_network_supabase_cli` |
| Frontend meldt backend offline | `BACKEND_URL` past niet bij startmodus | Compose-API: `http://localhost:18000`; lokale `make run(-reload)`: `http://localhost:8000`; herstart Vite na `.env`-wijziging |
| `permission denied` op `neo4j/import` of pytest-collectie | Neo4j-container heeft de bindmount UID `7474` gegeven | Gebruik `docker cp` voor restore; herstel voor hosttoegang gericht met `sudo chown -R "$(id -u):$(id -g)" MAP-IT_SA/neo4j/import` en `chmod 755` |
| Health is `degraded`/`unhealthy` | Eén of meer dependencies/sleutels ontbreken | Lees `/api/health` per component en `docker compose logs api neo4j redis`; controleer `.env` zonder waarden te loggen |
| Vectorfout “different dimensions” | 3072-queryvector tegen 768-kolom of omgekeerd | Controleer `format_type(atttypid, atttypmod)` voor de vectorkolom; laad/regeneer embeddings voor exact dezelfde dimensie |
| Login/callback gaat naar poort `3000` | Supabase-config heeft nog `site_url=http://127.0.0.1:3000` | Voor lokale callbackflows: zet site/additional redirect URLs op `http://localhost:5173` in `supabase/config.toml` en herstart Supabase |

## Canonieke documentatie

- [Backend developer setup](MAP-IT_SA/docs_tracked/DEVELOPER_SETUP.md)
- [API-, SSE- en authcontract](API_CONTRACT.md)
- [Data-ingestionrunbook](MAP-IT_SA/docs/guides/DATA_INGESTION.md)
- [Databaseschema](MAP-IT_SA/docs/DATABASE_SCHEMA.md)
- [Systeemarchitectuur](MAP-IT_SA/docs_tracked/SYSTEM_ARCHITECTURE.md)
- [Bekende issues en risico's](KNOWN_ISSUES_AND_RISKS.md)
- [Securityregister](SECURITY_REGISTER.md)
- [Deploymentrunbook](DEPLOYMENT_RUNBOOK.md)
- [Operationsrunbook](OPERATIONS_RUNBOOK.md)

## Checklist voor de eerste werkdag

- [ ] Beide repo's gecloned en remotes gecontroleerd.
- [ ] Python-`venv` aangemaakt; productie- en dev-dependencies geïnstalleerd.
- [ ] Backend- en frontend-`.env` lokaal gevuld zonder secrets te committen.
- [ ] Supabase vanuit de juiste map gestart; URL en keys uit `supabase status` gebruikt.
- [ ] Goedgekeurde databasebaseline of raw artifacts met versie en SHA-256 ontvangen.
- [ ] PostgreSQL-tabellen en kerncounts gecontroleerd.
- [ ] Redis en Neo4j healthy; graphdump indien nodig geladen.
- [ ] FastAPI-liveness op `18000` en frontend-loginpagina op `5173` bereikbaar.
- [ ] Lokale gebruiker en conversatie aangemaakt; één V3 SSE-query uitgevoerd.
- [ ] [Bekende issues](KNOWN_ISSUES_AND_RISKS.md) en [securityrisico's](SECURITY_REGISTER.md) gelezen.
- [ ] Geen `.env`, dumps of API-sleutels in Git-status.
