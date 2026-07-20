# Operations- en incidentrunbook

**Statusdatum:** 20 juli 2026  
**Topologie:** het VPS-pad uit [DEPLOYMENT_RUNBOOK](DEPLOYMENT_RUNBOOK.md)  
**Operations owner / on-call:** TBD  
**Tijdzone voor runlogs:** UTC

Dit document beschrijft de dagelijkse operatie en stelt initiële SLO-, backup- en DR-doelen voor. De repository bevat nog geen dashboards, alertmanager, scheduler, on-callrooster of bewijs van restore drills. Alles wat hieronder als “voorgesteld” is gemarkeerd, is dus een in te voeren control en geen bestaande garantie.

## 1. Dagelijkse startcheck

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

Lees bij de twee laatste calls ook de JSON-body. De full-healthroute retourneert niet automatisch HTTP 503 bij een `unhealthy` body; de frontendstatus meldt `connected` zodra HTTP bereikbaar is, ook wanneer `backend_status` degraded/unhealthy is.

Controleer vervolgens:

- fouten/restarts sinds de vorige dienst;
- backupjobs en checksum/retentie;
- schijf, inode, RAM/swap en containerstatus;
- verlopen/bijna verlopen TLS-certificaten en providerquota;
- dataversheid en mislukte ingest-/embeddingrunlogs;
- open critical/high incidenten in [KNOWN_ISSUES_AND_RISKS](KNOWN_ISSUES_AND_RISKS.md).

## 2. Health endpoints

| Endpoint | Werkelijke controle | Statussemantiek | Gebruik |
|---|---|---|---|
| `GET /api/health/live` | Alleen FastAPI-proces/event loop antwoordt. | Altijd 200 met `alive: true` zolang handler draait. | Liveness, iedere 30 s. |
| `GET /api/health/ready` | PostgreSQL; Gemini alleen als een Gemini-key is gezet. | 200 wanneer ready, 503 bij een falende kritieke check. | Load-balancer/readiness, iedere 30 s. |
| `GET /api/health` en `/api/health/` | PostgreSQL, Neo4j, Redis, Gemini, embeddingservice en Supabase parallel. | Body `healthy`, `degraded` of `unhealthy`; handler zet zelf geen 503. Redis/embedding zijn niet-kritiek, PostgreSQL en geconfigureerde Gemini wel. | Synthetische dependencycheck, iedere 2 min. |
| `GET /api/health/dependencies` | Dezelfde dependencies plus critical/fallbacklabels en configuratiesamenvatting. | JSON, geen beloofde non-200 bij degradatie. | Diagnose, niet publiek exposen. |
| `GET /api/health/metrics` | PostgreSQL, Neo4j en Redisstatus/latency. | JSON met 1/0,5/0, **geen Prometheus exposition format**. | Interne JSON-scrape/exporter. |
| `GET /api/analytics/health` | In-memory analyticsservice, buffer en last flush. | `healthy` of `disabled`; endpoint is momenteel niet geauthenticeerd. | Interne diagnose; blokkeer extern. |
| `GET /api/alerts/health` | Saved-searchservice. | Eenvoudige servicehealth. | Interne diagnose; blokkeer extern. |
| `GET /api/v4/proposal-session/health` | Bedoeld voor session manager, proposal agent en database. | In bron gedeclareerd, maar wordt waarschijnlijk door eerdere `GET /{session_id}` gematcht en kan 404 opleveren. | Niet als probe gebruiken tot routevolgorde is hersteld en getest. |
| `GET /api/backend-status` op SvelteKit | SvelteKit haalt FastAPI `/api/health` op. | 200 `connected` bij bereikbare response; 503 alleen bij fetchfout. | End-to-end proxybereikbaarheid, JSON-body beoordelen. |

OpenAPI is beschikbaar op backend `/docs`, `/redoc` en `/openapi.json`, maar moet in productie achter beheertoegang of uitgeschakeld worden.

## 3. Logs en diagnose

De applicaties configureren standaard logging naar stdout/stderr; er is geen applicatie-`FileHandler` of centrale logcollector in de code.

| Component | Locatie/commando | Opmerking |
|---|---|---|
| FastAPI/systemd | `journalctl -u mapit-api --since -1h` | Voeg `-f` toe voor follow; journaldretentie is OS-configuratie. |
| SvelteKit/PM2 | `pm2 logs mapit-ui --lines 200` | PM2-bestanden staan standaard onder de home van de PM2-user in `.pm2/logs/`; gebruik het commando als canonieke toegang. |
| Redis/Neo4j | `docker compose --env-file /etc/mapit/backend.env -p mapit logs --since 1h redis neo4j` | Neo4j heeft daarnaast een named `neo4j_logs`-volume; beheer via container/Compose. |
| NGINX | `/var/log/nginx/access.log`, `/var/log/nginx/error.log` | Zorg voor logrotate en maskering van tokens/querydata. |
| Supabase managed | Projectdashboard/log explorer van de provider | Toegang en retentie buiten deze repo beheren. |
| Deploy/ingest/backup | Nog te maken centraal runlog | Noteer commandoversie, SHA, data-artifactchecksum, start/einde, counts en operator. |

Veilige triage:

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

Log geen JWT's, cookies, service-rolekeys, volledige prompts met persoonsgegevens of database-URL's. Een aangetroffen secret geldt als securityincident: toegang intrekken/roteren en bereik bepalen.

## 4. Metrics en analytics

Aanwezige telemetry:

- `/api/analytics/overview?days=30`: usage, performance, content, proposals en feedback uit PostgreSQL;
- `/api/analytics/tools?hours=24`: toolperformance, trage tools, combinaties en process-local sessiemetrics;
- `/api/analytics/trends`, `/queries/popular`: tijdreeksen en populaire queries;
- V3 `/api/v3/conversation/analytics/summary`, `/low-confidence`, `/search`, `/export`;
- `tool_usage_analytics`, `tool_combination_analytics`, `conversation_analytics` en `response_feedback` in PostgreSQL;
- request-ID en timing via backendmiddleware;
- JSON dependencylatencies via `/api/health/metrics`.

Beperkingen:

- analytics kan bij databasefouten stil als `disabled` initialiseren;
- toolanalytics buffert tot 50 events of circa 60 seconden in process memory; een crash kan bufferdata verliezen;
- process-local sessiemetrics worden niet tussen Uvicornworkers gedeeld;
- analytics- en exportendpoints hebben geen backend-auth; publiceer ze niet;
- er is geen echte `/metrics`-route en geen Prometheus/Grafana-config in de repo.

Voorgestelde minimumdashboardpanelen: requestvolume/status/latency, SSE-start en streamduur, active streams, providerfouten/429/timeouts, tool success/latency, PostgreSQL connecties/querylatency, Redis memory/evictions, Neo4j nodes/relationships/querylatency, container/processrestarts, disk/inodes, backupouderdom en data-/embeddingversheid.

## 5. Alerts

Voorgestelde alerts; pas drempels na 30 dagen baseline aan.

| Alert | Initiële drempel | Urgentie |
|---|---|---|
| Liveness faalt | 2 opeenvolgende checks | P1 |
| Readiness faalt | 3 checks / 2 minuten | P1 |
| Full health `unhealthy` | 2 checks; PostgreSQL/Supabase direct | P1 |
| Full health `degraded` | 10 minuten | P2 |
| HTTP 5xx | >2% gedurende 5 minuten en minimaal 20 requests | P1/P2 op impact |
| SSE-startlatency | p95 >5 s gedurende 15 minuten | P2 |
| Provider 429/timeouts | >5% gedurende 10 minuten | P2; P1 als alle providers uitvallen |
| Tool failures | >10% per veelgebruikte tool in 15 minuten | P2 |
| Redis/Neo4j down | >5 minuten | P2 zolang fallback werkt, anders P1 |
| Databaseconnecties | >80% van limiet gedurende 10 minuten | P2 |
| Disk/inodes | >80% warning, >90% critical | P2/P1 |
| Process/container restart | >3 in 10 minuten | P1 |
| Backup | laatste succesvolle PostgreSQL-backup >26 uur; checksum/verify faalt direct | P1 |
| Data freshness | afgesproken dataset-SLA overschreden | P2 |
| TLS | <21 dagen tot expiratie; renewaltest faalt | P2/P1 |

Alerts moeten naar een bemand kanaal met acknowledgement, runbooklink en escalatie. Eigenaar/kanaal zijn momenteel **TBD**; dit is zelf een releaseblocker.

## 6. Voorgestelde SLO's

Deze doelen zijn nog niet gemeten of contractueel toegezegd.

| SLI | Voorgesteld SLO per kalendermaand |
|---|---:|
| Publieke webapp + login beschikbaar | 99,5% |
| FastAPI readiness | 99,5% |
| Succesvolle niet-streaming API-requests (geen 4xx) | 99,0% |
| V3-streams die een eerste SSE-event starten | 95% binnen 5 s; 99% binnen 15 s |
| V3-streams met `complete` zonder `error` | 98% |
| Kritieke databasewrites | 99,9% zonder verlies/duplicatie |
| Datafreshness | volgens de per-dataset-SLA hieronder |

Planned maintenance telt alleen buiten de beschikbaarheids-SLI wanneer vooraf aangekondigd en afzonderlijk gemeten. Externe providerfouten mogen in rapportage apart worden uitgesplitst, maar gebruikersimpact blijft zichtbaar.

## 7. Backups

### Beleid en retentie (voorgesteld)

| Store | Frequentie | Retentie | RPO-doel | Autoriteit |
|---|---|---|---|---|
| PostgreSQL/Supabase | Provider-PITR indien beschikbaar + dagelijkse logische dump | 7 dagelijks, 5 wekelijks, 12 maandelijks | 1 uur met PITR, anders 24 uur | Primair voor app/auth/content/analytics. |
| Neo4j | Dagelijkse offline dump na PostgreSQL-backup | 7 dagelijks, 5 wekelijks | 24 uur | Afgeleide graphdata, maar raw bron ontbreekt nu deels; backup dus noodzakelijk. |
| Redis | Dagelijkse RDB/AOF-kopie voor diagnose; niet als primaire recoverybron | 2 dagen | maximaal 24 uur, maar sessies/cache mogen verloren gaan | Cache/short-term state, niet source of truth. |
| Config/secrets | Versleutelde secret-storeversies bij iedere wijziging | Minimaal 12 maanden audittrail | Laatste geldige versie | Geen plaintext secrets in Git/backupbundel. |

Versleutel backups client-side of in de backupdienst, sla ze buiten de VPS/accountfailure-domain op, beperk toegang, maak SHA-256-checksums en test restores per kwartaal.

### PostgreSQL

Gebruik bij managed Supabase eerst provider-PITR/snapshots. Aanvullende logische dump:

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

Gebruik een dedicated backuprol met voldoende leesrechten; leg vast welke Supabase auth/storageobjecten de providerbackup dekt. Controleer dump met `pg_restore --list <bestand>` en kopieer daarna naar immutable/off-site storage.

### Neo4j

Neo4j Community 5.22 gebruikt een offline dump. Dit veroorzaakt een kort onderhoudsvenster:

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

Als `/backups` niet schrijfbaar is, bepaal eerst de container-UID met `docker compose --env-file /etc/mapit/backend.env -p mapit run --rm --no-deps neo4j id` en geef alleen die UID schrijfrecht. Verander niet willekeurig permissies op `neo4j_data`.

### Redis

Compose zet AOF aan. Maak daarnaast een synchrone RDB-snapshot met `redis-cli --rdb`:

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

Omdat Redis cache en tijdelijke conversation state bevat, is leeg starten doorgaans veiliger dan oude state terugzetten.

## 8. Restore en kwartaaltest

Restore nooit voor het eerst tijdens een incident. Gebruik een geïsoleerde restoreomgeving met dezelfde PostgreSQL/pgvector-, Neo4j- en Redis-majorversies.

### PostgreSQL restoretest

```bash
sha256sum --check /srv/mapit/backups/postgres/SHA256SUMS.latest
pg_restore --no-owner --no-acl --exit-on-error \
  --dbname="$RESTORE_DATABASE_URL" /pad/naar/mapit-UTC.dump
```

Maak het lege doelproject/de doeldatabase via de provider en gebruik daarna `pg_restore --dbname="$RESTORE_DATABASE_URL"`. Controleer tabellen/counts, RLS/publications/functions, vectorafmetingen, login en V3/V4-smoke met de queries uit [DATABASE_SCHEMA](MAP-IT_SA/docs/DATABASE_SCHEMA.md). Voor managed Supabase-auth/storage kan providerrestore nodig zijn naast een public-schema-dump.

### Neo4j restoretest

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

Voer dit op een afzonderlijke restorehost uit, of override de hostpoorten zodat zij niet met productie botsen. Verifieer verbinding, node-/relationshipcounts en representatieve graphquery. Als de graph aantoonbaar volledig afgeleid kan worden, is heringest vanuit geverifieerde raw artifacts een alternatief; momenteel ontbreken sommige artifacts, dus vertrouw daar niet zonder manifest op.

### Redis restoretest

Voorkeursherstel is een **lege Redis**: stop API, maak een kopie van het bestaande volume, start Redis leeg en accepteer cache/sessionverlies. Test een RDB eerst geïsoleerd:

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

Controleer representatieve keys en TTL's en verwijder daarna de tijdelijke container/volume volgens het lokale changeproces. Alleen als tijdelijke state echt moet worden teruggezet: zet FastAPI en productie-Redis uit, maak een tweede kopie van `mapit_redis_data`, leeg uitsluitend dat gevalideerde volume en plaats de gecontroleerde `dump.rdb` zoals hierboven. Dit is een destructieve productiehandeling en vereist expliciete changegoedkeuring; overschrijf het actieve volume nooit tijdens normaal verkeer.

### Restore-acceptatie

- backupchecksum geldig en artifact decryptbaar;
- restorecommand exit 0 en geen onverwachte warnings;
- schema/counts/vectorafmetingen conform databaseguide;
- Supabase login, RLS-negatietest en Realtime werken;
- Neo4j counts/representatieve relaties kloppen;
- V3 chat, persistence en V4 session smoke slagen;
- gemeten RPO/RTO, operator en afwijkingen in restorelog;
- productiebackup blijft onaangeroerd.

## 9. Scheduled jobs en datafreshness

### Werkelijke huidige staat

Er is **geen werkende schedulerconfiguratie** in de repository. Celery worker/beat verwijzen naar een niet-bestaand `app.services.celery_app`. De volgende functies zijn alleen handmatig of on-demand:

| Functie | Huidige trigger | Huidige freshnessgarantie |
|---|---|---|
| Funding-call detail enrichment | On-demand door service/tool; cache-TTL default 3600 s | Cache kan na een uur verversen, maar dit is geen volledige callsync. |
| `sync_topics_from_projects.py` | Handmatig script | Geen schema/cadans vastgelegd. |
| Document- en programme-ingest | Handmatige scripts | Geen automatische freshness-SLA. |
| Project/chunk/call embeddings | Handmatige generate/re-embed scripts | Geen automatische drift- of freshnessgarantie. |
| Saved-searchalerts | Handmatig `POST /api/alerts/process-alerts`; bedoeld voor niet-bestaande Celery task | Geen betrouwbare verwerking; e-mailintegratie heeft TODO. |
| Analytics flush | Tijdens requests, buffer 50 events of circa 60 s | Kan bij procescrash verliezen. |

Publiceer `process-alerts` niet als unauthenticated cron-URL. Voeg eerst service-auth, idempotentie/locking, timeout, retries, metrics en auditlogging toe.

### Voorgestelde cadans na implementatie

| Job | Cadans | Freshness-SLA |
|---|---|---|
| Open/upcoming funding calls sync | Dagelijks 02:00 UTC; extra run na bronrelease | <26 uur |
| Deadline/status normalization | Iedere 6 uur | <8 uur |
| Document/programme guide ingest | Wekelijks en event-driven bij nieuwe guide | <8 dagen, kritieke regelwijziging <48 uur |
| Incremental embeddings | Na elke succesvolle ingest, anders dagelijks | <30 uur na brondatamutatie |
| Neo4j afgeleide graphrebuild/increment | Na projectbatch, minimaal wekelijks | <8 dagen |
| Saved-search daily/weekly | Volgens gebruikersfrequentie | daily <26 uur; weekly <8 dagen |
| Quality scenario sample | Nachtelijk op staging; volledig voor release | Resultaat <24 uur / releasegebonden |

Iedere job schrijft een runrecord met code-SHA, inputversie/checksum, start/einde, inserted/updated/skipped/failed counts en volgende retry. Geen overlap: gebruik distributed lock en unieke run-ID.

## 10. Capacity en grenzen

| Onderdeel | Geconfigureerde/gekende grens | Operationele actie |
|---|---|---|
| PostgreSQL pool | Backenddefault 5–50; analytics extra pools tot 5 per proces | Tel over alle Uvicornworkers; alert voor providerlimiet. |
| Supabase REST | `max_rows=1000` lokaal | Pagineer; geen volledige exports via één REST-call. |
| Neo4j | Compose: heap 512 MiB–2 GiB, page cache 512 MiB, single Community node | Monitor heap/page cache/disk; geen HA. |
| Redis | Single `redis:7-alpine`, AOF, geen auth/TLS op loopback | Alleen intern binden/firewall; monitor memory/evictions. |
| API | VPS-runbook start 2 Uvicornworkers; geen bewezen loadbaseline | Loadtest staging vóór verhogen; deel state via Redis. |
| SSE/NGINX | Proxy read/send timeout voorgesteld 600 s | Alert op hangende streams; test disconnect/cancel. |
| Upload | Frontend 10 MiB, backend 50 MiB, NGINX-runbook 10 MiB | 10 MiB is effectieve ondersteunde limiet. |
| Vector | Doel 3072; snapshot deels 768; geen ANN-index voor >2000 in migratie `002` | Houd model/schema/data gelijk; monitor sequential-scanlatency. |
| Rate limit | App-global uit; runbook stelt NGINX 20 r/s per IP voor | Geen per-user/cost-aware/distributed bescherming; nog securityblocker. |
| Providerquota | Niet in repo vastgelegd | Leg quota/budget per provider vast en alert op 429/kosten. |

## 11. Incident response

| Niveau | Definitie | Eerste reactie | Escalatie |
|---|---|---:|---|
| P0 | Bevestigde secret/service-role exposure, cross-tenant datalek, destructief dataverlies of actieve compromise | 15 min | Incident commander + security + data owner onmiddellijk; autoriteiten/klanten volgens beleid. |
| P1 | Volledige uitval, login/database onbeschikbaar, brede foutieve antwoorden of RPO/RTO in gevaar | 15 min | On-call → service owner na 15 min; management na 30 min. |
| P2 | Degradatie met workaround, één datastore/provider/tool uit, data te oud | 1 uur | Service owner dezelfde werkdag. |
| P3 | Kleine bug/onderhoudsprobleem zonder directe gebruikersimpact | Volgende werkdag | Backlog met owner/target. |

Contactnamen, telefoon, kanaal, DPO/security en management zijn **TBD** en moeten vóór productie buiten dit publieke document in het on-callregister staan.

Standaardprocedure:

1. Acknowledge, benoem incident commander/scribe en noteer UTC-tijd.
2. Classificeer gebruikers-, data-, security- en compliance-impact; bewaar bewijs.
3. Beperk schade: verkeer stoppen, feature uit, provider/key intrekken of vorige release activeren.
4. Diagnoseer read-only met health, logs, release-SHA, schema/data-/providerstatus.
5. Herstel via bekende rollback/restore; vermijd ad-hoc productiedata-edits.
6. Valideer met twee gebruikers, RLS-negatietest en kernsmokes.
7. Communiceer status/cadans; security/privacy via formeel proces.
8. Sluit alleen na monitoringstabiliteit; postmortem binnen 5 werkdagen voor P0/P1.

## 12. Disaster recovery

Voorgestelde doelen:

| Scenario | RPO | RTO |
|---|---:|---:|
| PostgreSQL/Supabase met PITR | 1 uur | 4 uur |
| Zonder PITR | 24 uur | 8 uur |
| Neo4j | 24 uur | 8 uur; sneller als betrouwbare heringest mogelijk is |
| Redis | Best effort / 24 uur | 1 uur; cache/sessionverlies geaccepteerd |
| Volledige VPS-regio/accountfailure | Volgens traagste primaire store | 8 uur na beschikbare nieuwe VPS/DNS |

DR-volgorde:

1. Declareer P1/P0, freeze writes en bepaal laatst consistente recovery point.
2. Provision nieuwe VPS/account volgens deploymentrunbook; herstel secrets via secret store en roteer bij compromise.
3. Herstel Supabase/PostgreSQL eerst; verifieer auth, schema, RLS, publications en counts.
4. Herstel Neo4j uit dezelfde of latere afgeleide snapshot; anders gecontroleerde heringest.
5. Start lege Redis of herstel alleen geverifieerde tijdelijke state.
6. Deploy exact compatibele backend/frontend-SHA's en environmentversie.
7. Voer health, login, tenantnegatie, V3/V4 en datarepresentativiteitchecks uit.
8. Schakel DNS/verkeer om met verlaagde TTL; monitor intensief.
9. Documenteer werkelijk RPO/RTO en reconcile writes die na recovery point verloren gingen.

Voer elk kwartaal een tabletop en minimaal halfjaarlijks een technische restore/regionale rebuild uit. Een DR-doel geldt pas als bewezen nadat een andere operator het runbook binnen RTO heeft uitgevoerd.
