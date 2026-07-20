# MAPIT securityregister

**Status:** open securitybaseline — niet geschikt als productie-goedkeuring  
**Peildatum:** 20 juli 2026  
**Owner:** TBD  
**Herbeoordeling:** vóór iedere externe deployment en minimaal maandelijks zolang critical/high-items openstaan

## 1. Managementsamenvatting

MAPIT is in de huidige codebaseline **niet veilig voor publieke productie-exposure**. De belangrijkste oorzaak is niet één los secret, maar het ontbreken van een afgedwongen identity- en autorisatieketen: de browser authenticeert bij Supabase, terwijl de FastAPI V3/V4-routes die identiteit niet valideren en caller-controlled/default user-ID's gebruiken. Daarnaast toont het getrackte Supabase-schema brede grants en Row Level Security (RLS) op slechts twee van 35 publieke tabellen.

Alle critical-items in dit register zijn release blockers. Dit document registreert de toestand; het voert geen codefixes of secretrotaties uit. Zie [API_CONTRACT.md](API_CONTRACT.md) voor het exacte route- en identitycontract en [KNOWN_ISSUES_AND_RISKS.md](KNOWN_ISSUES_AND_RISKS.md) voor het bredere statusregister.

## 2. Severity en status

| Severity | Betekenis |
|---|---|
| **Critical** | Waarschijnlijke ongeautoriseerde toegang, tenantdoorbraak, secretcompromis of brede datamutatie; blokkeert productie. |
| **High** | Ernstige vertrouwelijkheids-, integriteits- of beschikbaarheidsimpact met realistische exploitroute; blokkeert productie tenzij aantoonbaar gemitigeerd. |
| **Medium** | Beperkte of voorwaardelijke impact; oplossen vóór brede uitrol. |
| **Low** | Hardening, hygiene of defense-in-depth. |

Statuswaarden: `open`, `in-progress`, `accepted`, `resolved`. Alle onderstaande bevindingen staan op `open`; owner is `TBD` tenzij toegewezen.

## 3. Basis-threatmodel

### Te beschermen assets

- Supabase identities, access-/refreshtokens en gebruikersprofielen.
- Persoonlijke en collaboration-conversaties, berichten en citations.
- Proposalinhoud, consortiuminformatie, evaluaties, budgetten en gegenereerde documenten.
- Funding-call-, document-, embedding- en analyticsdata; sommige brondata is publiek, afgeleide gebruikersdata niet.
- Database-, Supabase service-role-, Neo4j-, JWT- en AI-providercredentials.
- Beschikbaarheid en kostenquota van Gemini/OpenAI/OpenRouter, database, Redis en Neo4j.
- Auditbaarheid: betrouwbare koppeling van mutaties aan één gevalideerde principal.

### Aanvallers en relevante scenario's

- Een niet-ingelogde internetgebruiker die FastAPI of onbeveiligde SvelteKit-API-routes rechtstreeks aanroept.
- Een geldige gebruiker die IDs raadt en data van een andere gebruiker/collaboration probeert te lezen of wijzigen (IDOR/BOLA).
- Een aanvaller met XSS die een browserleesbaar access token buitmaakt.
- Een kwaadwillende uploader of externe documentbron die parser-, prompt-injection- of resource-exhaustiongedrag triggert.
- Een actor die een bekende fallback/default credential gebruikt of API-quota uitput.
- Een fout geconfigureerde deployment waarin een service-role-key via een `PUBLIC_`/`VITE_` variabele in de browserbundle belandt.

### Securitydoelen

1. Iedere private request heeft één cryptografisch gevalideerde identity.
2. Iedere objectaccess valideert ownership, collaboration membership en rol server-side.
3. RLS biedt database-level defense-in-depth; `anon` krijgt alleen expliciet publieke read-toegang.
4. Secrets komen nooit in Git, logs, frontendbundles of foutresponses.
5. Muterende cookie-authenticated requests zijn beschermd tegen CSRF; browsercode is gehard tegen XSS.
6. Dure en muterende endpoints hebben rate-, size- en concurrency-limieten.

### Niet bewezen door deze review

Er is geen pentest, dependency-CVE-scan, cloudconfigreview, secret-historyscan of live Supabase-policycheck uitgevoerd. Claims over RLS zijn gebaseerd op `webapp-ui/supabase_cli/schema.sql`; verifieer de werkelijke staging- en productiedatabase afzonderlijk. Een schemafile is geen bewijs dat dezelfde toestand live staat.

## 4. Trust boundaries

```text
[Onbetrouwbare browser / internet]
        |
        | HTTPS, Supabase cookies, JSON/SSE
        v
+-------------------------------+
| SvelteKit / webapp-ui         |
| - Supabase SSR client         |
| - auth- en API-routes         |
| - V3/V4 proxies              |
+-------------------------------+
   |              |             |
   | PostgREST/   | HTTP/SSE    | service-role (enkele serverroutes)
   | Auth         v             |
   |       +------------------+ |
   |       | FastAPI          | |
   |       | MAP-IT_SA        | |
   |       +------------------+ |
   |          |     |     |     |
   v          v     v     v     |
[Supabase] [Redis][Neo4j][AI/EU externe API's]
 Auth +      cache  graph  Gemini/OpenAI/
 PostgreSQL              OpenRouter/EU Portal
```

Belangrijkste boundarybreuk: SvelteKit kent de geverifieerde Supabase-user, maar FastAPI ontvangt geen betrouwbaar afgedwongen mapping van die user naar een backendprincipal. De backend is daarom een afzonderlijke trust boundary en mag niet vertrouwen op alleen de aanwezigheid van de proxy.

## 5. Bekende kwetsbaarheden en risico's

| ID | Severity | Bevinding en impact | Geverifieerde locatie | Voorgestelde fix |
|---|---|---|---|---|
| SEC-001 | Medium | **Hardcoded Mapbox public token.** Een Mapbox public token is per ontwerp browserzichtbaar, maar een token in broncode kan niet per omgeving veilig worden beheerd, blijft in Git-historie staan en kan buiten toegestane origins/quota worden misbruikt als scopes/restricties te ruim zijn. | `webapp-ui/src/routes/(protected)/dashboard/consortium/+page.svelte:10-11` en gebruik op regel 253. | Roteer het token; maak een nieuwe least-privilege public token met URL-restricties en quota; laad via een bewust publieke `PUBLIC_MAPBOX_TOKEN`; scan Git-historie. Verwijder nooit een vereiste browser-public key naar een serversecret zonder de Mapboxflow aan te passen. |
| SEC-002 | Critical | **Voorspelbare JWT-fallback in SvelteKit.** Als `JWT_SECRET_KEY` ontbreekt, signeert de proxy met de bekende string `fallback-secret-change-in-production`. Zodra backendauth op deze tokens vertrouwt, kan iedereen geldige HS256-tokens met willekeurige rollen/sub maken. Nu is auth al open; de fallback maakt een latere “auth aan”-wijziging gevaarlijk. | `webapp-ui/src/lib/server/jwt.ts:10-13`. Backendgedrag verschilt: `MAP-IT_SA/app/core/auth.py:65-73` genereert per proces een random secret als de env ontbreekt. | Fail closed bij startup als `JWT_SECRET_KEY` ontbreekt; minimaal 256-bit random secret uit secret manager; identieke, expliciete keyconfig aan beide kanten; key-ID/rotatieplan. Bij voorkeur valideert FastAPI rechtstreeks een Supabase JWT via ondersteunde asymmetric/JWKS-flow in plaats van een proxy-identiteit. |
| SEC-003 | High | **Accesscookies zijn browserleesbaar (`httpOnly:false`).** Een XSS kan het Supabase access token uitlezen en exfiltreren. De loginroute forceert dit zowel in SSR-cookieopties als in handmatige access-tokenlogica. | `webapp-ui/src/routes/api/auth/login/+server.ts:45-53` en `:101-115`. | Maak authcookies `httpOnly:true`, `secure:true`, passend `sameSite`; laat `@supabase/ssr` één consistente cookieflow beheren. Verwijder handmatige tokenheaders/cookies en test login, refresh en logout. Voeg CSP en XSS-tests toe. |
| SEC-004 | Critical | **`PUBLIC_SUPABASE_SERVICE_KEY` is een misleidende publieke variabelenaam.** Het voorbeeld instrueert een service-role-key onder `PUBLIC_`; SvelteKit beschouwt public env als browser-exposeerbaar. De huidige code importeert deze naam niet, maar kopiëren/gebruik ervan zou RLS volledig omzeilen. | `webapp-ui/.env.example:20-21`. Correcte private variant staat op `:27-28` en wordt server-side gebruikt in collaborationroutes. | Verwijder de public variant overal; gebruik uitsluitend `SUPABASE_SERVICE_ROLE_KEY` via `$env/static/private`/secret manager; voeg buildcheck toe die service-role-/secretnamen onder `PUBLIC_`/`VITE_` weigert; roteer als de key ooit in een echte public env of bundle stond. |
| SEC-005 | High | **Globale SvelteKit-rate limiting is uitgeschakeld.** Alle `/api/*`-limiting in de hook is uitgecommentarieerd. Login/register/query/AI-message hebben wel losse in-memory limiters, maar V3/V4 catch-all, CRUD en andere endpoints niet uniform; een horizontale deployment deelt de counters niet. | `webapp-ui/src/hooks.server.ts:8-46`; TODO op regels 10-14. Backend heeft `RateLimitMiddleware`, maar `app/main.py` registreert die niet. | Activeer Redis-backed limiting op edge/proxy én backend; tier per login, AI, ingest en mutatie; vertrouw alleen proxy-IPheaders van bekende reverse proxy; retourneer 429/Retry-After; load-test en monitor rejects. |
| SEC-006 | Critical | **V3/V4 en andere FastAPI-routes zijn zonder auth.** `get_current_user` bestaat, maar wordt niet gebruikt; `AuthMiddleware` is niet gemount. V3 gebruikt default `user_id=1`; V4 create gebruikt `default-user`; IDs zijn caller-controlled. Lezen, schrijven, verwijderen, AI-kosten en analytics-export zijn daardoor rechtstreeks bereikbaar. | `MAP-IT_SA/app/api/conversation_routes_v3.py:41-42,204-207`; `proposal_session_routes.py:187-191`; routers in `app/main.py`; authdependency alleen gedefinieerd in `app/core/auth.py`. | Maak deny-by-default authdependency op alle private routers; valideer tokenissuer, audience, signature, expiry en type; deriveer user uitsluitend uit principal; voeg per object ownership/membershipcheck toe; verwijder user-ID-queryparameters/defaults; netwerk-isoleer FastAPI totdat dit gereed is. |
| SEC-007 | High | **Collaboration membershipchecks ontbreken/inconsistent.** Directe toegang tot collaborationgesprekken wordt op sommige GET/PATCH-paden niet robuust gecontroleerd; chat websocket/messagepad heeft expliciete TODO; DELETE controleert membership wel. Dit creëert IDOR en ongeautoriseerde mutatie. | `webapp-ui/src/routes/api/conversations/[conversation_id]/+server.ts:58-90` en `:169-187`; `webapp-ui/src/routes/api/chat/websocket/+server.ts:48-68`. | Centraliseer `requireCollaborationMember`/`requireRole`; pas toe op read, write, messages, invitations, locks, websocketupgrade en delete; query membership met geverifieerde `client.id`; voeg negatieve tests toe voor non-member, removed member en rolgrenzen. |
| SEC-008 | Critical | **Minimale RLS plus brede grants in het schema.** Van 35 `public`-tabellen hebben alleen `document_chunks` en `horizon_documents` RLS in de snapshot. Tegelijk staan voor veel tabellen `GRANT ALL ... TO anon, authenticated`. Zonder RLS kan PostgREST brede gebruikers-, conversation-, proposal- en analyticsdata blootstellen of laten muteren. | `webapp-ui/supabase_cli/schema.sql:8677-8718` (vier policies/twee tabellen); grants beginnen rond `:13628` en omvatten o.a. `client`, `conversation`, `message`, `proposals`, `proposal_sessions`. | Revoke default `ALL` van `anon`/`authenticated`; definieer per tabel SELECT/INSERT/UPDATE/DELETE-policies met `auth.uid()` en membershiprelaties; forceer RLS waar passend; service role alleen server-side; test policies met anon, owner, member en outsider. Vergelijk live DB met schema. |
| SEC-009 | High | **V3 streamingproxy valideert met `getSession()` en gebruikt niet de echte user.** Server-side `getSession()` vertrouwt sessiedata zonder dezelfde netwerkvalidatie als `getUser()`. De code leest `session.user.id` maar maakt vervolgens altijd een token voor `webapp-proxy`; backend gebruikt alsnog default `1`. | `webapp-ui/src/routes/api/proxy/conversation/[conversation_id]/message/stream/+server.ts:13-27`. | Gebruik de al door de hook gevalideerde `locals.user` of `supabase.auth.getUser()`; geef een gevalideerde usercontext door volgens één backendcontract; laat backend opnieuw cryptografisch valideren en autoriseren. |
| SEC-010 | Critical | **Proxies zijn geen security boundary.** De V3 non-streaming/context/resetproxies en V4 catch-all doen geen uniforme sessie- of ownershipcheck. Direct FastAPI-verkeer kan de proxy geheel omzeilen. | `webapp-ui/src/routes/api/proxy/conversation/**/+server.ts`; `webapp-ui/src/routes/api/v4/[...path]/+server.ts`; `MAP-IT_SA/app/main.py`. | Backendauth is verplicht, onafhankelijk van proxy. Voeg daarnaast same-origin sessiechecks toe; expose FastAPI alleen op private netwerk/loopback achter NGINX; firewall poort 8000/18000 extern; contracttests voor 401/403. |
| SEC-011 | High | **Documentingestie en destructieve routes zijn open.** Upload, URL-ingest, batch-upload, reprocess en delete vereisen geen principal. Dit maakt opslag-/AI-resource-exhaustion, contentinjectie en dataverwijdering mogelijk. | `MAP-IT_SA/app/api/document_routes.py`; `proposal_document_routes.py`; V4 delete/update/generate-routes. | Admin/editorrollen en ownershipchecks; streaming uploadlimiet vóór volledig inlezen; malware/parser sandbox; contenttype-validatie; SSRF-safe allowlist met DNS/IP-hercontrole; quota en auditlog; soft-delete/confirmatie voor destructieve acties. |
| SEC-012 | Medium | **Overbrede CORS-default.** FastAPI gebruikt standaard `CORS_ORIGINS=["*"]`, alle methods/headers en `allow_credentials=True`. Dit is ongeschikt als productiebeleid en browsergedrag rond wildcard+credentials is foutgevoelig. Enkele SvelteKit-proxies voegen eveneens `Access-Control-Allow-Origin: *` toe. | `MAP-IT_SA/app/core/config.py:20`; `app/main.py:42-49`; onder meer V3 streamingproxy regels 53-61. | Expliciete HTTPS-origin allowlist per omgeving; alleen benodigde methods/headers; geen wildcard met credentials; CORS-headers alleen door één laag; automatische preflighttests. CORS is geen vervanging voor auth/CSRF. |
| SEC-013 | High | **Geen expliciete CSRF-verdediging voor muterende cookie-authenticated routes.** Cookies zijn `sameSite:lax`, maar er is geen CSRF-token of consistente Origin/Referer-validatie gevonden. SameSite helpt, maar is geen compleet beleid voor alle browserflows/subdomeinen. | Cookieconfig in loginroute; geen `csrf`-implementatie gevonden in `webapp-ui/src` of `MAP-IT_SA/app`. | Voor mutaties: CSRF-token of strict Origin/Host-check; `SameSite=Lax/Strict` bewust kiezen; GET nooit muterend; CORS beperken; tests voor cross-site form/fetch en sibling-subdomainaanvallen. |
| SEC-014 | Medium | **Ruwe interne foutdetails en agentredenering gaan naar clients.** Veel handlers zetten `str(e)` in HTTP- of SSE-responses. V3/V4 emitteert `thought`, `reasoning`, toolparameters en samenvattingen. Dit kan schema-, prompt-, data- of providerdetails lekken. | `conversation_routes_v3.py`, `proposal_session_routes.py`; globale HTTP-handler publiceert `HTTPException.detail` in `app/api/error_handlers.py:125-165`; agents emitten reasoning. | Extern stabiele foutcodes en generieke berichten; request-ID voor support; details alleen serverlog met redactie. Emit geen chain-of-thought; vervang door korte, veilige status-/toolactiviteit zonder secrets of private context. |
| SEC-015 | High | **Default Neo4j-credential staat in code en envvoorbeeld.** Als een deployment de variabele niet zet, gebruikt de client een bekende passwordwaarde. Compose/config kan deze credential bovendien normaliseren als ontwikkelstandaard. | `MAP-IT_SA/app/core/neo4j_config.py:46-48`; `MAP-IT_SA/.env.example` bij `NEO4J_PASSWORD`. | Geen passworddefault; fail closed buiten expliciete local-devmode; unieke secret per omgeving; Neo4j niet publiek exposen; TLS waar remote; roteer bestaande credential en controleer logs/history. |
| SEC-016 | Medium | **Foutief/handmatig cookie- en headerbeheer in login.** De route roept `response.headers.set('sb-access-token', token, options)` aan alsof dit een cookie-API is; `Headers.set` heeft geen cookieoptiecontract. Dit kan inconsistente sessies en tokenexposure via custom responseheaders veroorzaken naast de SSR-cookieflow. | `webapp-ui/src/routes/api/auth/login/+server.ts:101-131`. | Gebruik uitsluitend SvelteKit `cookies.set`/Supabase SSR `setAll`; geen tokens in gewone responseheaders of responsebody; integratietest dat exact de bedoelde `Set-Cookie`-attributen ontstaan. |

## 6. Authn/authz- en RLS-besluit

### Gewenst canoniek model

1. Supabase Auth is identity provider.
2. SvelteKit valideert met `getUser()` voor UX en same-origin API-gating.
3. FastAPI valideert iedere bearer-token zelf of vertrouwt uitsluitend een cryptografisch/verifieerbaar intern token van een authenticating proxy. Netwerkvertrouwen alleen is onvoldoende.
4. `sub` wordt server-side gekoppeld aan `client.auth_id`; nooit een `user_id` uit query/body vertrouwen.
5. Resourcehandlers laden object plus owner/collaboration en beslissen 404/403 zonder objectdetails te lekken.
6. PostgreSQL RLS herhaalt dezelfde eigendoms-/membershipgrens als defense-in-depth.

### Minimale routepolicy

| Routegroep | Minimumeis |
|---|---|
| Health/liveness | Publiek, maar geen secrets, exceptionteksten of providerdata. Diepe dependencyhealth alleen intern/admin. |
| Funding calls en expliciet publieke documenten | Auth optioneel indien productbesluit; alleen read; RLS-policy expliciet publiek. |
| V3 conversations/messages/context | Auth verplicht; owner of collaboration member; create/update/delete met passende rol. |
| V4 proposal sessions/documenten | Auth verplicht; proposal owner/editor; evalueren/genereren kostengequoteerd en rate-limited. |
| Feedback submit | Auth verplicht of aantoonbaar abuse-bestendig; user/conversationrelatie afleiden. |
| Feedback analytics, conversation analytics/export | Admin/analyst; nooit publiek. |
| Ingestie/reprocess/delete | Admin/data-steward; auditlog en sterke limieten. |
| Collaboration invites/members/locks | Member voor read; owner/admin voor invite/role/delete; membership op iedere websocket/operation. |

### Live RLS-verificatie vóór deployment

Voer dit uit op staging en productie, niet alleen op de dump:

```sql
SELECT schemaname, tablename, rowsecurity, forcerowsecurity
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY tablename;

SELECT schemaname, tablename, policyname, roles, cmd, qual, with_check
FROM pg_policies
WHERE schemaname = 'public'
ORDER BY tablename, policyname;

SELECT grantee, table_name, privilege_type
FROM information_schema.role_table_grants
WHERE table_schema = 'public'
  AND grantee IN ('anon', 'authenticated', 'service_role')
ORDER BY table_name, grantee, privilege_type;
```

Release-acceptatie: iedere public table heeft een gedocumenteerd besluit (`public read`, `user-owned`, `collaboration-scoped`, `admin-only`, `service-only`) en geautomatiseerde negatieve policytests.

## 7. Secretsmanagement

| Secret/credential | Waar nodig | Mag publiek? | Huidige configuratienaam |
|---|---|---|---|
| Supabase anon key | Browser en server-SSR | Ja, met correcte RLS | `PUBLIC_SUPABASE_ANON_KEY` |
| Supabase service-role key | Alleen vertrouwde SvelteKit/backendserver | **Nee** | `SUPABASE_SERVICE_ROLE_KEY` frontend; `SUPABASE_SERVICE_KEY` backend |
| JWT signing key | SvelteKit en backend zolang eigen HS256-flow bestaat | **Nee** | `JWT_SECRET_KEY` |
| PostgreSQL URL/password | Backend/SvelteKit server en migrationjob | **Nee** | `DATABASE_URL`, `PRIVATE_DATABASE_URL` |
| Neo4j password | Backend | **Nee** | `NEO4J_PASSWORD` |
| AI-providerkeys | Backend of specifieke serverroute | **Nee** | `GEMINI_API_KEY`, `GOOGLE_EMBEDDING_API_KEY`, `OPENAI_API_KEY`, `OPENROUTER_API_KEY`, frontendvoorbeeld `GOOGLE_API_KEY` |
| Static backend API key | Backend/client indien gebruikt | **Nee** | `STATIC_API_KEY` |
| Mapbox public token | Browser | Ja, maar beperkt op origin/scope/quota | maak `PUBLIC_MAPBOX_TOKEN` |

### Opslagbeleid

- Productiesecrets uitsluitend in deployment secret manager of root-readable service-env; nooit in Git, Docker image, PM2 ecosystembestand met wereldleesbare rechten, frontend public env of logs.
- `.env` alleen lokaal, mode `0600`, in `.gitignore`; voorbeelden bevatten uitsluitend duidelijke placeholders en geen werkende defaults.
- Log geen Authorization-, Cookie-, service-role-, database-URL- of providerkeywaarden. Redigeer ook querystrings wanneer die tokens kunnen bevatten.
- Scheid local/staging/production-projecten, databases, Mapbox tokens en AI-keys.
- CI voert secret scanning uit op huidige tree én Git-historie; een gevonden secret wordt eerst geroteerd en daarna uit history behandeld.

### Rotatiebeleid

- Direct roteren bij vermoedelijke exposure, medewerker-/leverancierswissel of public-bundle/Git-opname.
- Service-role, database, Neo4j en AI-keys minimaal per 90 dagen totdat centraal beleid anders voorschrijft; Mapbox public token minimaal jaarlijks en bij misbruik.
- JWT-rotatie ondersteunt overlap via key-ID/twee geldige verificatiekeys; maximale tokenlevensduur begrenst overlap. Trek bestaande sessies in bij incident.
- Iedere rotatie heeft owner, datum, test van nieuwe key en aantoonbare intrekking van de oude key. Geen secretwaarde in ticket/documentatie opnemen.

## 8. Cookies, CORS en CSRF: actuele configuratie

| Onderdeel | Huidige code | Beoordeling |
|---|---|---|
| Login access cookie | `secure:true`, `sameSite:'lax'`, expliciet `httpOnly:false`; handmatig 2 uur en domein `.mapit.work` | XSS-risico; hardcoded productiedomein maakt local/staging inconsistent. |
| Refresh cookie | Handmatig `httpOnly:true`, `secure:true`, `sameSite:'lax'`, 30 dagen, `.mapit.work` | Betere baseline, maar dubbele/handmatige Supabase-cookieflow moet worden verwijderd en getest. |
| Overige Supabase SSR-cookies | `setAll` neemt libraryopties over en forceert `path:'/'` | Attributen zijn niet centraal gedocumenteerd of getest. |
| FastAPI CORS | Origins default `*`, credentials aan, methods/headers `*` | Niet acceptabel voor productie; pin origins. |
| SvelteKit proxy CORS | Sommige routes voegen wildcardheaders toe | Inconsistent en overbodig bij same-origin gebruik. |
| CSRF | Geen token of centrale Origin-check gevonden | High open issue voor cookie-authenticated mutaties. |

Aanbevolen productiepolicy: uitsluitend HTTPS; HSTS bij NGINX; host/origin allowlist voor het echte frontenddomein; authcookies `Secure`, `HttpOnly`, minimaal bereikbare `Domain`/`Path`, bewust `SameSite`; CSRF-token of centrale Origin-validatie op alle mutaties; strikte CSP zonder `unsafe-inline` waar haalbaar.

## 9. Logging, privacy en AI/databoundaries

- De hooks loggen het e-mailadres van de ingelogde gebruiker tweemaal (`webapp-ui/src/hooks.server.ts:71-73`). Beperk productieauthlogs tot pseudonieme user-ID/request-ID en vermijd dubbele PII.
- Conversation analytics kan vraag, antwoord, tools en fouten opslaan. Leg bewaartermijn, rechtsgrond, inzage/verwijdering en toegangsrol vast voordat echte gebruikersdata wordt verwerkt.
- Prompts en proposalcontent verlaten de eigen boundary richting AI-providers. Classificeer data, sluit passende verwerkersvoorwaarden/DPA's, configureer providerretentie en voorkom dat secrets/persoonsdata onnodig in prompts komen.
- Documentingestie en toolresultaten zijn onbetrouwbare content. Markeer ze als data, niet als instructie; voeg prompt-injectiontests en outputvalidatie toe.
- `agent_thinking`/`reasoning` hoort niet in productie naar de browser; toon een veilige activiteitssamenvatting.

## 10. Pre-deployment securitychecklist

Een item is pas gereed met code, geautomatiseerde test en stagingbewijs.

### Release blockers

- [ ] SEC-002: bekende JWT-fallback verwijderd; ontbrekende key laat proces failen.
- [ ] SEC-004: geen service-role-key onder `PUBLIC_`/`VITE_`; mogelijk blootgestelde key geroteerd.
- [ ] SEC-006/010: FastAPI private routes geven zonder/ongeldig token 401 en zonder objectrecht 403/404; publieke backendpoort is gefirewalld.
- [ ] V3/V4 user identity komt uitsluitend uit gevalideerde principal; defaults `1` en `default-user` zijn verwijderd uit het securitycontract.
- [ ] SEC-007: collaboration membership/rol wordt op ieder read/write/websocketpad gecontroleerd; negatieve tests slagen.
- [ ] SEC-008: live RLS/grants geïnventariseerd en per tabel gecorrigeerd; anon/outsidertests slagen.
- [ ] SEC-011: ingestie, AI-mutaties en deletes zijn rolbeveiligd, geaudit en gelimiteerd.
- [ ] Alle werkende hardcoded/default credentials zijn geroteerd; secret scan van tree en historie is schoon.

### Web- en platformhardening

- [ ] Access- en refreshcookieflow gebruikt één Supabase SSR-mechanisme; `HttpOnly`, `Secure`, `SameSite`, Domain, Path en expiratie zijn getest.
- [ ] CSRF-verdediging en expliciete CORS-originallowlist zijn actief en met cross-origin negatieve tests bewezen.
- [ ] CSP, HSTS, `X-Content-Type-Options`, `Referrer-Policy` en framebeleid staan in NGINX/SvelteKit en zijn getest.
- [ ] Globale en endpoint-specifieke rate limits werken gedeeld via Redis/edge; 429 en alerts zijn getest.
- [ ] Uploadtype, streaming size limit, parserisolatie, SSRF/DNS-rebindingbescherming en opslagquota zijn getest.
- [ ] FastAPI `/docs`/`openapi.json`, debug-/testendpoints en diepe healthdetails zijn intern of admin-only in productie.
- [ ] Database, Redis en Neo4j luisteren alleen op private interfaces/netwerken; firewalls en TLS zijn beoordeeld.

### Assurance en operatie

- [ ] Dependency-, container- en IaC-scans hebben geen ongeaccepteerde critical/high-bevindingen.
- [ ] Menselijke code-securityreview en externe pentest omvatten auth bypass, IDOR/BOLA, RLS, CSRF, XSS, upload/SSRF en SSE.
- [ ] Auditlogs registreren auth-, admin-, ingestie-, proposal- en deleteacties met request-ID, zonder secrets/PII-overlogging.
- [ ] Incidentrespons bevat contactowner, token/key-revocation, Supabase session revocation, forensische logretentie en meldingsbesluit.
- [ ] Backup/restore is getest en backups zijn encrypted, toegangsbeperkt en aan retentiebeleid gebonden.
- [ ] Privacy-/AI-providerreview, dataretentie en verwijderprocedure zijn goedgekeurd.

## 11. Acceptatiecriteria voor “security ready”

MAPIT mag pas als security-ready worden gelabeld wanneer:

1. geen critical of ongeaccepteerde high-bevinding openstaat;
2. een onafhankelijke reviewer authn/authz/RLS end-to-end in staging heeft getest;
3. de FastAPI-route-inventaris automatisch faalt wanneer een private route zonder policy/dependency wordt toegevoegd;
4. live schema/grants/policies tegen versiebeheer worden gedrift-checkt;
5. secretsrotatie, logging, alerts en incidentrespons operationeel getest zijn;
6. de resultaten gedateerd en door een menselijke securityowner zijn ondertekend.

## 12. Geverifieerde bronnen

- `DOCUMENTATION_REVIEW.md`
- `MAP-IT_SA/app/main.py`
- `MAP-IT_SA/app/core/auth.py`
- `MAP-IT_SA/app/core/config.py`
- `MAP-IT_SA/app/core/neo4j_config.py`
- `MAP-IT_SA/app/core/rate_limit.py`
- `MAP-IT_SA/app/api/conversation_routes_v3.py`
- `MAP-IT_SA/app/api/proposal_session_routes.py`
- `MAP-IT_SA/app/api/document_routes.py`
- `MAP-IT_SA/app/api/proposal_document_routes.py`
- `MAP-IT_SA/app/api/error_handlers.py`
- `MAP-IT_SA/.env.example`
- `webapp-ui/src/hooks.server.ts`
- `webapp-ui/src/lib/server/jwt.ts`
- `webapp-ui/src/lib/server/rateLimit.ts`
- `webapp-ui/src/routes/api/auth/login/+server.ts`
- `webapp-ui/src/routes/api/proxy/conversation/**/+server.ts`
- `webapp-ui/src/routes/api/v4/[...path]/+server.ts`
- `webapp-ui/src/routes/api/conversations/[conversation_id]/+server.ts`
- `webapp-ui/src/routes/api/chat/websocket/+server.ts`
- `webapp-ui/src/routes/(protected)/dashboard/consortium/+page.svelte`
- `webapp-ui/.env.example`
- `webapp-ui/supabase_cli/schema.sql`
