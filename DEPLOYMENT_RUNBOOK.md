# Deploymentrunbook

**Status:** kandidaat-runbook; productie-uitrol geblokkeerd door open P0's  
**Laatste verificatie tegen repository:** 20 juli 2026  
**Gekozen topologie:** één Ubuntu/Debian-VPS, NGINX, SvelteKit adapter-node onder PM2, FastAPI onder systemd, Redis en Neo4j in Docker, externe/beheerde Supabase  
**Eigenaar:** TBD

Dit is het enige ondersteunde deploymentpad in de huidige documentatie. De bestaande Kubernetes-assets zijn **niet inzetbaar**: ze starten een niet-bestaande `app.services.celery_app`, gebruiken een niet-bestaand `/metrics`-endpoint, taggen `latest` en deployen geen frontend. `MAP-IT_SA/docker-compose.yml` is evenmin een productie-API-manifest: het gebruikt `--reload`, een source-bindmount en een extern lokaal Supabase-network.

> **Releaseblokkade:** de laatste quality baseline is 54,8% en “NOT PRODUCTION READY”; frontendchecks en backendtestcollectie zijn niet groen en critical securityissues staan open. Gebruik dit runbook nu voor een afgeschermde stagingomgeving. Productie vereist dat de releasecriteria in [KNOWN_ISSUES_AND_RISKS](KNOWN_ISSUES_AND_RISKS.md) en [SECURITY_REGISTER](SECURITY_REGISTER.md) zijn gesloten of formeel geaccepteerd.

## Topologie

```text
Internet :443
    |
    v
NGINX (TLS, rate limit, alleen 22/80/443 publiek)
    |
    v
SvelteKit adapter-node, PM2, 127.0.0.1:3000
    |                         |
    | server-side proxy      +--> managed Supabase Auth/REST/Realtime (TLS)
    v
FastAPI/systemd, 127.0.0.1:8000
    |              |              |
    v              v              v
PostgreSQL/     Redis Docker   Neo4j Docker
Supabase TLS    127.0.0.1:16379 127.0.0.1:17687
```

De browser gebruikt de publieke Supabase URL en anon key. De service-rolekey blijft alleen server-side. FastAPI is niet publiek bereikbaar zolang backend-auth/autorisatie niet wordt afgedwongen.

## 1. Prerequisites

### Server en accounts

De volgende sizing is een **startpunt, niet benchmarkonderbouwd**:

| Onderdeel | Minimum voor staging/kleine productie | Opmerking |
|---|---:|---|
| VPS | 4 vCPU, 8 GiB RAM, 80 GiB SSD | Neo4j is in Compose begrensd op 2 GiB heap plus 512 MiB page cache; Supabase draait extern. |
| OS | Actuele ondersteunde Ubuntu LTS of Debian stable, x86_64 | Installeer security-updates automatisch. |
| Python | 3.11.x | `pyproject.toml` richt formatter/linter op py311; Dockerfile gebruikt 3.13 maar dit pad gebruikt geen API-container. |
| Node | 22.x LTS + npm | Sluit aan op `@types/node` 22; de repo definieert geen `engines`, dus pin de gekozen runtime in serverbeheer. |
| Docker | Engine met Compose v2 | Alleen Redis 7-alpine en Neo4j 5.22.0-community uit Compose. |
| NGINX/PM2 | Actuele distributieversie / vastgelegde PM2-versie | Leg werkelijk geïnstalleerde versies vast in het deployrecord. |
| Supabase/PostgreSQL | Beheerd Supabase-project met pgvector en schema | De huidige repository kan een leeg productieproject nog niet veilig migreren; zie hieronder. |

Maak een niet-root serviceaccount, bijvoorbeeld `mapit`, en deze layout:

```text
/srv/mapit/releases/<release-id>/MAP-IT_SA
/srv/mapit/releases/<release-id>/webapp-ui
/srv/mapit/current/backend -> release/backend
/srv/mapit/current/frontend -> release/frontend
/srv/mapit/backups
/etc/mapit/backend.env
/etc/mapit/frontend.env
```

Rechten: releasebestanden eigendom van `mapit`, `/etc/mapit/*.env` root-owned met groep `mapit` en mode `0640`, backupmap niet web-accessible. Sta in de firewall alleen SSH, HTTP en HTTPS toe. Poorten 3000, 8000, 16379, 17474 en 17687 mogen niet extern openstaan. Beperk SSH op beheer-IP en gebruik keys, geen wachtwoordlogin.

### DNS en TLS

1. Maak een A/AAAA-record voor het gekozen domein naar de VPS.
2. Installeer NGINX en Certbot.
3. Plaats eerst de HTTP-config, vraag daarna een certificaat aan en activeer de TLS-config.
4. Controleer automatische renewal met `certbot renew --dry-run`.

### Externe afhankelijkheden

- Een bestaand, geback-upt Supabase-project met database-URL, project-URL, anon key en service-rolekey.
- Minimaal één werkende AI-providerkey; de default is Gemini.
- Google/Gemini embeddingkey als vector search actief is.
- Een Mapbox public token met toegestane origins en minimale scopes.
- DNS/TLS en een secret store of root-only environmentfiles.

## 2. Release-identiteit en preflight

Deploy nooit een zwevende branch of `latest`. Leg voor beide repositories commit-SHA, releasetijd, operator, database-schemahash en dataversie vast.

```bash
git -C MAP-IT_SA rev-parse HEAD
git -C webapp-ui rev-parse HEAD
sha256sum webapp-ui/supabase_cli/schema.sql MAP-IT_SA/migration/*.sql
```

Harde pre-deployment gates:

```bash
cd MAP-IT_SA
source venv/bin/activate
make test
make lint

cd ../webapp-ui
npm run check
npm run lint
npm run build
```

De baseline van 20 juli 2026 voldoet hier niet aan. **Stop** bij een fout; gebruik `npm run host` niet als deploygate, omdat het shellscript na de buildstap PM2 alsnog kan starten.

Controleer daarnaast:

- geen secret of echte `.env` in Git/buildartifact;
- alle critical/high blockers zijn gesloten of schriftelijk geaccepteerd;
- backup en restoretest zijn recenter dan het afgesproken interval;
- schema, code en embeddings gebruiken dezelfde dimensie;
- staging smoke-, login-, chat-, V4- en rollbacktest zijn geslaagd.

## 3. Environmentconfiguratie

### Backend `/etc/mapit/backend.env`

| Variabele | Vereist | Productiewaarde/regel |
|---|---|---|
| `DATABASE_URL` | ja | TLS-PostgreSQL-URL van Supabase; gebruik een server/private connectiestring. |
| `SUPABASE_URL` | ja | `https://<project>.supabase.co`. |
| `SUPABASE_ANON_KEY` | aanbevolen | Publieke anon key; geen service role. |
| `SUPABASE_SERVICE_KEY` | ja | Service-rolekey, uitsluitend in dit private bestand. |
| `JWT_SECRET_KEY` | ja | Uniek random geheim, gelijk aan frontend; minimaal 32 random bytes. |
| `GEMINI_API_KEY`, `OPENAI_API_KEY`, `OPENROUTER_API_KEY` | minstens één | Alleen geconfigureerde providers vullen. |
| `DEFAULT_AI_PROVIDER` | ja | `gemini`, `openai` of `openrouter`; defaultcode is `gemini`. |
| `GOOGLE_EMBEDDING_API_KEY` | bij vectors | Key voor `gemini-embedding-001`; valt in code terug op Gemini-key. |
| `EMBEDDING_MODEL` | bij vectors | `models/gemini-embedding-001`. |
| `EMBEDDING_DIMENSION` | bij vectors | `3072`; database moet eerst zijn gemigreerd en heringebed. |
| `VECTOR_SEARCH_ENABLED` | ja | `false` zolang schema/data niet volledig 3072 zijn; anders `true`. |
| `REDIS_URL` | ja | `redis://127.0.0.1:16379/0`. |
| `REDIS_HOST`, `REDIS_PORT` | ja | `127.0.0.1`, `16379`; short-term-memory leest deze apart. |
| `NEO4J_URI`, `NEO4J_USER`, `NEO4J_PASSWORD` | ja | `bolt://127.0.0.1:17687`, `neo4j`, uniek geheim. |
| `CORS_ORIGINS` | ja | JSON-lijst met exact het HTTPS-domein, bijvoorbeeld `["https://mapit.example"]`; nooit `*` met credentials. |
| `DEBUG` | ja | `false`. |

Kopieer geen placeholderwaarde uit `.env.example`. De Compose-default `mapit_secure_password_2024` is niet geschikt voor productie.

### Frontend `/etc/mapit/frontend.env`

| Variabele | Vereist | Productiewaarde/regel |
|---|---|---|
| `PRIVATE_DATABASE_URL` | ja | Private TLS-database-URL voor Drizzle/serverroutes. |
| `PUBLIC_SUPABASE_URL`, `VITE_SUPABASE_URL` | ja | Publieke HTTPS-project-URL. |
| `PUBLIC_SUPABASE_ANON_KEY`, `VITE_SUPABASE_ANON_KEY` | ja | Alleen de anon key. |
| `SUPABASE_SERVICE_ROLE_KEY` | alleen adminroutes | Server-private service role; nooit als `PUBLIC_*` of `VITE_*`. |
| `BACKEND_URL` | ja | `http://127.0.0.1:8000`. |
| `JWT_SECRET_KEY` | ja | Exact hetzelfde sterke geheim als backend. |
| `MAPBOX_API_KEY` | voor kaart | Origin-beperkte public token; verwijder/roteer eerst de hardcoded token. |
| `PUBLIC_APP_URL` | ja | Canonieke HTTPS-origin. |
| `PUBLIC_APP_ENV` | ja | `production`. |
| `PUBLIC_ENABLE_DEBUG_TOOLS` | ja | `false`. |
| `PORT`, `HOST`, `ORIGIN` | ja | `3000`, `127.0.0.1`, `https://<domein>`. |

Definieer **geen** `PUBLIC_SUPABASE_SERVICE_KEY`. Alles met `PUBLIC_` of `VITE_` kan in browsercode/buildoutput terechtkomen. Statische publieke SvelteKit-variabelen moeten al tijdens `npm run build` aanwezig zijn; een wijziging vereist dus rebuild.

## 4. Databasevoorbereiding en migrations

De repository heeft nog geen bewezen productie-migration chain. De volledige `webapp-ui/supabase_cli/schema.sql` is een dump inclusief Supabase-systeemobjecten; voer die niet blind uit op een beheerd project. Backend `001_init_schema.sql` dekt niet alle frontendtabellen, en `008`/`009` plus Drizzle bestaan daarnaast. Zie de actuele ledger in [DATABASE_SCHEMA](MAP-IT_SA/docs/DATABASE_SCHEMA.md).

Daarom gelden twee paden:

1. **Bestaand gecontroleerd productieproject:** maak een backup, vergelijk objecten/dimensies met de queries uit `DATABASE_SCHEMA.md`, voer uitsluitend gereviewde, nog niet toegepaste migrations uit en registreer checksum/tijd/operator.
2. **Leeg project:** **stop de deployment** totdat een schone stagingrehearsal de canonieke chain volledig heeft bewezen. Een dump in Supabase Studio plakken is geen geaccepteerde productieprocedure.

De 768→3072-migratie is destructief voor drie vectorkolommen. Alleen na backup, onderhoudsvenster en een bewezen re-embeddingplan:

```bash
psql "$DATABASE_URL" -v ON_ERROR_STOP=1 \
  -f MAP-IT_SA/migration/002_upgrade_embeddings_to_3072.sql
```

Daarna moeten funding calls, document chunks en embedding cache opnieuw worden opgebouwd. Zet tot de verificatie `VECTOR_SEARCH_ENABLED=false`. Voer count-, dimensie- en RLS/publicationchecks uit `DATABASE_SCHEMA.md` uit. Data bootstrap staat in [DATA_INGESTION](MAP-IT_SA/docs/guides/DATA_INGESTION.md).

## 5. Build

### Backend

```bash
cd /srv/mapit/releases/<release-id>/MAP-IT_SA
python3.11 -m venv venv
source venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python -c "from app.main import app; print(app.title)"
```

`requirements-dev.txt` hoort in de pre-release build/testomgeving, niet noodzakelijk op de productieserver.

### Frontend

```bash
cd /srv/mapit/releases/<release-id>/webapp-ui
set -a
. /etc/mapit/frontend.env
set +a
npm ci
npm run check
npm run lint
npm run build
test -f build/index.js
```

Gebruik dezelfde lockfile en Node-major als in staging. Bewaar buildlog en checksums bij het release-record.

## 6. Services installeren

### Redis en Neo4j

Start alleen deze twee Compose-services met een vaste projectnaam. Daardoor wordt het externe Supabase-network voor de development-API niet gebruikt en blijven volumes over releases heen hetzelfde.

```bash
cd /srv/mapit/current/backend
docker compose --env-file /etc/mapit/backend.env -p mapit up -d redis neo4j
docker compose --env-file /etc/mapit/backend.env -p mapit ps
docker compose --env-file /etc/mapit/backend.env -p mapit exec redis redis-cli ping
```

Controleer Neo4j via Bolt of intern HTTP:

```bash
curl --fail http://127.0.0.1:17474/
```

### FastAPI onder systemd

Maak `/etc/systemd/system/mapit-api.service`:

```ini
[Unit]
Description=MAPIT FastAPI
After=network-online.target docker.service
Wants=network-online.target
Requires=docker.service

[Service]
Type=simple
User=mapit
Group=mapit
WorkingDirectory=/srv/mapit/current/backend
EnvironmentFile=/etc/mapit/backend.env
ExecStart=/srv/mapit/current/backend/venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000 --workers 2
Restart=on-failure
RestartSec=5
TimeoutStopSec=45
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/srv/mapit/current/backend/uploads

[Install]
WantedBy=multi-user.target
```

Maak de uploadmap schrijfbaar voor `mapit`, laad daarna:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mapit-api
sudo systemctl status mapit-api --no-pager
curl --fail http://127.0.0.1:8000/api/health/live
curl --fail http://127.0.0.1:8000/api/health/ready
```

Begin met twee workers en meet geheugen/locks. Redis moet beschikbaar zijn voor gedeelde state; lokale fallbacks zijn niet geschikt voor consistente multi-workerstate.

### SvelteKit onder PM2

```bash
cd /srv/mapit/current/frontend
set -a
. /etc/mapit/frontend.env
set +a
pm2 start build/index.js --name mapit-ui --time --update-env
pm2 save
pm2 startup
curl --fail http://127.0.0.1:3000/api/backend-status
```

Voer het door `pm2 startup` getoonde rootcommando eenmalig uit. Gebruik niet poort 5173: dat is de Vite-devserver; deze productiebuild luistert expliciet op 3000.

## 7. NGINX

Plaats in `/etc/nginx/conf.d/mapit-limits.conf` (de `http`-context):

```nginx
limit_req_zone $binary_remote_addr zone=mapit_app:10m rate=20r/s;
```

Plaats in `/etc/nginx/sites-available/mapit.conf`, vervang domein en certificaatpaden en symlink naar `sites-enabled`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name mapit.example;
    location /.well-known/acme-challenge/ { root /var/www/html; }
    location / { return 301 https://$host$request_uri; }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name mapit.example;

    ssl_certificate /etc/letsencrypt/live/mapit.example/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mapit.example/privkey.pem;

    client_max_body_size 10m;
    limit_req zone=mapit_app burst=40 nodelay;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options DENY always;
    add_header Referrer-Policy strict-origin-when-cross-origin always;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
    }
}
```

Deze topologie proxy't Supabase niet via de VPS; browser en server gebruiken de beheerde HTTPS-URL. Dat vermijdt de oude, incomplete `/supabase/`-config en houdt service-roleverkeer server-side.

```bash
sudo nginx -t
sudo systemctl reload nginx
curl --fail https://mapit.example/api/backend-status
```

## 8. Deployvolgorde

1. Kondig maintenance/release aan en blokkeer schemawijzigende writes indien een migration nodig is.
2. Noteer huidige release-SHA's en maak database-/Neo4jbackups volgens [OPERATIONS_RUNBOOK](OPERATIONS_RUNBOOK.md).
3. Clone/fetch beide repositories naar een nieuwe, immutable release-directory en checkout exacte SHA's.
4. Bouw backendvenv en frontend zoals hierboven; alle gates moeten slagen.
5. Verifieer schema en voer alleen goedgekeurde pending migrations uit; voer post-migration queries uit.
6. Start/verifieer Redis en Neo4j.
7. Wissel de `current/backend`- en `current/frontend`-symlinks atomair naar de nieuwe release.
8. `sudo systemctl restart mapit-api`; wacht op `/api/health/ready`.
9. Laad `/etc/mapit/frontend.env`, start of `pm2 restart mapit-ui --update-env`; wacht op `/api/backend-status`.
10. Test NGINX-config en reload.
11. Voer de volledige post-deploychecklist uit; zet maintenance pas daarna uit.

## 9. Health- en smokechecks

```bash
curl --fail http://127.0.0.1:8000/api/health/live
curl --fail http://127.0.0.1:8000/api/health/ready
curl --fail http://127.0.0.1:8000/api/health
curl --fail http://127.0.0.1:3000/api/backend-status
curl --fail https://mapit.example/api/backend-status
```

`/api/health/live` bewijst alleen dat het proces antwoordt. `/ready` controleert PostgreSQL en, indien geconfigureerd, Gemini. De volledige healthcheck controleert ook Neo4j, Redis, embeddings en Supabase. Controleer de JSON-status; `curl --fail` faalt niet wanneer de full-healthbody `unhealthy` bevat, omdat dat endpoint zelf geen 503 zet. Gebruik V4 `/api/v4/proposal-session/health` nog niet als probe: de eerdere `/{session_id}`-route overschaduwt dit pad.

Voer daarna handmatig uit:

- login, refresh en logout in een schone browser;
- aanmaken van een persoonlijke conversatie;
- één V3 SSE-chatvraag met zichtbare status/tool/answer/complete-flow;
- herladen en teruglezen van gesprek;
- V4 proposal session aanmaken, chatten en één sectiehandeling;
- controleren dat een tweede testgebruiker geen data van de eerste ziet;
- browserconsole op mixed content, CORS- en cookieproblemen;
- logs op secrets, tracebacks en herstartloops.

## 10. Rollback

Applicatierollback en databaserollback zijn aparte besluiten.

### Applicatie zonder schemawijziging

1. Stop verkeer of zet maintenance aan.
2. Zet beide `current`-symlinks terug naar de vorige release-ID.
3. Herlaad de vorige frontend-environment indien die verschilde.
4. Restart backend en frontend:

```bash
sudo systemctl restart mapit-api
cd /srv/mapit/current/frontend
set -a
. /etc/mapit/frontend.env
set +a
pm2 restart mapit-ui --update-env
```

5. Voer alle health- en smokechecks uit; leg reden/tijd/operator vast.

### Met schema- of datamigratie

- Voer geen automatische “down” uit: niet alle migrations hebben rollback en `002` verwijdert embeddings.
- Kies vooraf één van: backward-compatible approllback, expliciet gereviewde down-migration, of volledige PostgreSQL-restore.
- Bij `002`: een approllback naar 768 vereist databasebackup terugzetten of 768-kolommen en embeddings opnieuw opbouwen. Zet vector search uit tijdens herstel.
- Herstel PostgreSQL eerst, daarna Neo4j-afgeleide data; Redis mag als cache worden geleegd. Zie het operationsrunbook.

## 11. Post-deploychecklist

- [ ] Release-ID, beide SHA's, operator, datum en change ticket geregistreerd.
- [ ] Alle build/test/lint/check-gates groen en logs bewaard.
- [ ] Databasebackup en restorebewijs bekend; migration checksums geregistreerd.
- [ ] Vectorafmetingen en embeddingmodel consistent; counts binnen verwachte bandbreedte.
- [ ] `live`, `ready`, full health en frontend backend-status gecontroleerd; V4-functionaliteit via een echte sessionsmoke getest.
- [ ] TLS-chain, renewal, DNS en firewall gecontroleerd; interne poorten niet publiek.
- [ ] Login/logout, V3 SSE, persistence en V4-smoke geslaagd.
- [ ] RLS/tenant-negatiecheck met tweede gebruiker geslaagd.
- [ ] PM2/systemd/Docker/NGINX stabiel zonder restartloop of nieuwe errors.
- [ ] Monitoring en alerts ontvangen telemetry; synthetische check werkt.
- [ ] Rollbackdoel bestaat en is getest op staging.
- [ ] Bekende issues bijgewerkt en stakeholders over resterende beperkingen geïnformeerd.

## Canonieke vervolginformatie

- [Onboarding en lokale quickstart](START_HERE.md)
- [Backend developer setup](MAP-IT_SA/docs_tracked/DEVELOPER_SETUP.md)
- [Database en migration ledger](MAP-IT_SA/docs/DATABASE_SCHEMA.md)
- [Data bootstrap](MAP-IT_SA/docs/guides/DATA_INGESTION.md)
- [API/auth/SSE-contract](API_CONTRACT.md)
- [Securityregister](SECURITY_REGISTER.md)
- [Operations, backup en incident response](OPERATIONS_RUNBOOK.md)
