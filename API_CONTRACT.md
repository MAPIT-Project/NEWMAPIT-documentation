# MAPIT API-, SSE- en authenticatiecontract

**Status:** actuele codebaseline, met bekende contractbreuken  
**Gecontroleerd op:** 20 juli 2026  
**Scope:** contract tussen `webapp-ui/` en de FastAPI-backend in `MAP-IT_SA/`

Dit document beschrijft wat de huidige code daadwerkelijk doet. Het is geen garantie dat de API veilig voor productie is. Met name authenticatie en gebruikersisolatie worden door de V3- en V4-backendroutes niet afgedwongen. Zie [SECURITY_REGISTER.md](SECURITY_REGISTER.md) en [KNOWN_ISSUES_AND_RISKS.md](KNOWN_ISSUES_AND_RISKS.md).

## 1. Basis-URL's en protocol

| Gebruik | Basis-URL | Toelichting |
|---|---|---|
| FastAPI, lokaal proces | `http://localhost:8000` | Standaard van `app/core/config.py` en `make run`/`make run-reload`. |
| FastAPI, Docker Compose vanaf de host | `http://localhost:18000` | Hostmapping van de backendcontainer; dit is ook de standaardfallback van de SvelteKit-proxies. |
| SvelteKit | `http://localhost:5173` | Browser gebruikt bij voorkeur de same-origin routes onder `/api/...`. |
| OpenAPI | `${BACKEND_BASE}/openapi.json` | Gegenereerd door FastAPI. OpenAPI beschrijft de JSON-routes, maar niet de verschillende SSE-payloads als discriminated union. |

JSON-verzoeken gebruiken `Content-Type: application/json`, behalve de multipart-uploadroutes. Streamingroutes retourneren `text/event-stream`. Alle SSE-frames hebben alleen een `data:`-regel met één JSON-object en eindigen met een lege regel:

```text
data: {"type":"token","content":"tekst"}

```

Er worden geen benoemde SSE `event:`-velden en geen heartbeatframes verzonden. De client moet daarom discrimineren op het JSON-veld `type`.

## 2. Authenticatie en gebruikersidentiteit: huidige staat

### 2.1 Feitelijke flow

```text
Browser
  -> POST /api/auth/login (SvelteKit)
  -> Supabase Auth signInWithPassword
  <- Supabase-sessiecookies
  -> SvelteKit hook: supabase.auth.getUser() valideert de browsergebruiker

V3 streaming:
Browser -> SvelteKit streamingproxy
        -> getSession() controleert dat een sessie aanwezig is
        -> proxy maakt een eigen HS256-JWT met sub="webapp-proxy"
        -> FastAPI ontvangt Authorization-header, maar V3 gebruikt geen authdependency

V3 non-streaming/context en V4:
Browser -> transparante SvelteKit-proxy -> FastAPI
        (geen uniforme sessiecontrole en geen betrouwbare user-mapping)
```

De frontendlogin gebruikt Supabase Auth. `hooks.server.ts` valideert server-side met `supabase.auth.getUser()` en zet `event.locals.user`. Dit beveiligt niet automatisch iedere SvelteKit-API-route; een handler moet `locals.user` of Supabase zelf controleren.

`webapp-ui/src/lib/server/jwt.ts` kan daarnaast een MAPIT-eigen HS256-token maken. Dit is niet hetzelfde contract als een Supabase access token: de backend verwacht onder meer een eigen `type: "access"` claim en verifieert met `JWT_SECRET_KEY`. Een Supabase-token wordt nergens als zodanig door FastAPI gevalideerd.

FastAPI bevat wel `get_current_user`, rol- en tierdependencies in `app/core/auth.py`, maar `app/main.py` registreert geen `AuthMiddleware` en de hieronder beschreven V3/V4-, feedback-, document- en callroutes gebruiken `Depends(get_current_user)` niet. Een meegestuurde bearer-token heeft daar dus momenteel geen autoriserend effect.

### 2.2 User-ID-contract

| Pad | Werkelijke bron van identiteit | Risico/gevolg |
|---|---|---|
| V3 message stream en non-streaming | Queryparameter `user_id`, standaard integer `1` | De caller kan de ID zelf kiezen; bij weglaten wordt opslag/analytics aan gebruiker `1` gekoppeld. De Supabase-user-ID wordt niet gemapt. |
| V3 context GET/DELETE | Alleen `conversation_id` uit het pad | Geen ownershipcheck. |
| V4 create | Queryparameter `user_id`, standaard string `default-user` | De caller bepaalt de eigenaar van een automatisch aangemaakt proposal. |
| Overige V4-routes | Alleen `session_id`/`section_id` uit het pad | Geen controle dat de caller eigenaar of collaborator is. |
| V3 streamingproxy | Supabase-sessie moet bestaan, maar gegenereerde backend-JWT heeft `sub=webapp-proxy` | De gelezen `session.user.id` wordt niet gebruikt. Backend negeert het JWT bovendien. |
| V3 non-streamingproxy en V4 catch-allproxy | Geen uniforme authenticatiecheck | Rechtstreekse en proxied calls kunnen de open backend bereiken. |

**Contractstatus:** er is geen geïmplementeerde, betrouwbare mapping `Supabase auth.users.id -> client.id -> FastAPI user_id`. Behandel alle V3/V4-data daarom als niet tenant-geïsoleerd totdat authn, authz en ownershipchecks in code zijn toegevoegd.

## 3. V3 Conversation API

Prefix: `/api/v3/conversation`  
Bron: `MAP-IT_SA/app/api/conversation_routes_v3.py`

### 3.1 Modellen

`MessageRequest`:

```json
{
  "message": "Welke open Horizon Europe-calls passen bij circulaire bouw?",
  "include_context": true
}
```

`message` is verplicht en is in het Pydantic-model niet op lengte of lege inhoud gevalideerd. `include_context` is optioneel en standaard `true`, maar wordt door de route niet doorgegeven aan `ConversationManager`; de vlag heeft daardoor momenteel geen functioneel effect.

`MessageResponse`:

```json
{
  "response": "...",
  "metadata": {
    "architecture": "simplified_v3",
    "confidence": 0.82,
    "tools_used": ["search_funding_calls"]
  }
}
```

De inhoud van `metadata` is een vrij JSON-object en kan per agentpad verschillen.

### 3.2 Endpoints

| Methode en pad | Input | Succesrespons | Opmerking |
|---|---|---|---|
| `POST /{conversation_id}/message/stream` | Body `MessageRequest`; query `user_id: int = 1` | SSE, HTTP 200 | Slaat user/assistantberichten op en logt analytics. Errors na het starten van de stream zijn SSE-events, geen HTTP-foutstatus. |
| `POST /{conversation_id}/message` | Body `MessageRequest`; query `user_id: int = 1` | `MessageResponse` | Niet-streamend. Onverwachte fout wordt HTTP 500. |
| `GET /{conversation_id}/context` | Geen body | `{conversation_id, message_count, messages, architecture}` | Leest maximaal 10 recente berichten; geen user-/ownershipfilter. |
| `DELETE /{conversation_id}/context` | Geen body | `{success, message, conversation_id}` | Wist context via `clear_conversation`; geen ownershipcheck. |
| `GET /health` | Geen body | `{status, architecture, components, timestamp, errors?}` | Test Redis, database en Gemini. Een Gemini-call van maximaal vijf seconden is onderdeel van deze check. |
| `GET /analytics/summary` | Query `hours: int = 24` | `{success, period_hours, data}` | Geen grenzen op `hours`, geen auth. |
| `GET /analytics/low-confidence` | Query `threshold: float = 0.4`, `limit: int = 50` | `{success, threshold, count, conversations}` | Geen auth. |
| `GET /analytics/search` | Optioneel `query`, `conversation_id`, `min_confidence`, `max_confidence`; `limit=100`, `offset=0` | `{success, count, limit, offset, results}` | Geen Pydantic-grenzen op paginaparameters. |
| `GET /analytics/export` | Optioneel `start_date`, `end_date` in ISO-formaat | `{success, count, start_date, end_date, data}` | Ongeldige datum geeft HTTP 400; geen auth. |

Er bestaat geen `POST /api/v3/conversation` om een gesprek aan te maken. Conversationrecords worden door frontend/Supabase-routes beheerd; de V3-backend verwacht een bestaande numerieke `conversation_id`.

### 3.3 Werkelijk uitgezonden V3-SSE-events

| `type` | Wanneer | Payloadvelden |
|---|---|---|
| `status` | Verwerkingsfase | `stage`: `processing`, `context_loaded`, `agent_starting`, eventueel `synthesizing`, `streaming_response`; `message: string`. |
| `conversation_name` | Alleen als er nog geen recente berichten zijn en naamgeneratie slaagt | `name: string`, `conversation_id: int`. |
| `agent_thinking` | Iedere ReAct-iteratie | `iteration: int`, `thought: string`, `action: string`. Let op: dit exposeert agentredenering aan de client. |
| `tool_call` | Vóór tooluitvoering | `iteration`, `tool_name`, `display_name`, `parameters: object`, `reasoning`. |
| `tool_result` | Na tooluitvoering | `iteration`, `tool_name`, `success`, `result_count`, `result_summary` (maximaal circa 300 tekens), `execution_time_ms`. |
| `token` | Nadat het complete antwoord al is opgebouwd | `content: string`; de manager knipt het antwoord in blokken van 50 tekens. Dit is gesimuleerde outputstreaming, geen provider-tokenstream. |
| `complete` | Normaal einde | Top-level `ai_provider`, `ai_version`, `model_used`, `data_source`, `processing_time`, `projects_found`, `logs`; `metadata` met `architecture`, `confidence`, `tools_used`, `processing_time_ms`. |
| `error` | Agent-, opslag- of routefout | Minimaal `error: string`; soms `stage: "error"`; bij opslagfout ook `response`. Een opslagfout kan worden gevolgd door tokens en `complete`. |

De V3-route heeft `progress` in een interne allowlist, maar geen bekeken producer emitteert dit type. Het interne `agent_complete` van `ToolAwareAgent` wordt door `ConversationManager` verwerkt en niet rechtstreeks naar de V3-client gestuurd.

Voorbeeld:

```bash
curl -N -X POST \
  'http://localhost:8000/api/v3/conversation/123/message/stream?user_id=1' \
  -H 'Content-Type: application/json' \
  -d '{"message":"Zoek open calls voor groene waterstof","include_context":true}'
```

Een bearer-header is voor dit endpoint momenteel niet vereist en verandert de backendidentiteit niet.

## 4. V4 Proposal Session API

Prefix: `/api/v4/proposal-session`  
Bron: `MAP-IT_SA/app/api/proposal_session_routes.py`

De oudere, elders gedocumenteerde prefix `/api/v4/proposals/sessions` bestaat niet.

### 4.1 Requestmodellen

| Model | Velden |
|---|---|
| `CreateSessionRequest` | `proposal_id: int` verplicht; `call_id: string` verplicht; `title: string` verplicht; `proposal_type: string = "RIA"`; `conversation_id: int|null = null`. |
| V4 `MessageRequest` | `message: string` verplicht; `section_context: string|null = null`. |
| `SectionUpdateRequest` | `content: string` verplicht. |
| `SectionStatusRequest` | `status: string` verplicht. Geldige waarden zijn feitelijk `empty`, `draft`, `in_review`, `complete`; iedere onbekende waarde wordt stilzwijgend `draft`. |
| `SectionGenerateRequest` | `guidance: string|null = null`; `word_target: int = 500`; `style: string = "formal"`. Er zijn geen minimum-/maximumgrenzen. |

`SessionResponse` bevat:

```json
{
  "session_id": "uuid",
  "proposal_id": 42,
  "call_id": "HORIZON-CL5-2026-01-D2-01",
  "title": "GREENLOOP",
  "proposal_type": "RIA",
  "status": "active",
  "active_section_id": null,
  "completion_percentage": 0.0,
  "total_words": 0,
  "sections": {
    "excellence": {
      "section_id": "excellence",
      "section_name": "...",
      "content": "",
      "word_count": 0,
      "quality_score": null,
      "status": "empty",
      "suggestions_count": 0,
      "gaps_count": 0,
      "lock_status": "unlocked",
      "is_dirty": false,
      "version": 1,
      "last_updated": null
    }
  },
  "created_at": "2026-07-20T10:00:00+00:00"
}
```

### 4.2 Endpoints

| Methode en pad | Input | Succesrespons |
|---|---|---|
| `POST /create` | Body `CreateSessionRequest`; query `user_id: string = "default-user"` | `SessionResponse`. Als `proposal_id` niet bestaat, wordt een proposal aangemaakt; de uiteindelijke ID kan dus afwijken. |
| `GET /{session_id}` | Geen body | `SessionResponse`; 404 als niet gevonden. |
| `DELETE /{session_id}` | Geen body | `{"success":true,"message":"Session deleted"}`. |
| `POST /{session_id}/message/stream` | Body V4 `MessageRequest` | SSE met agent- en sessiestate-events. |
| `POST /{session_id}/message` | Body V4 `MessageRequest` | `{response, sections_affected, tools_used, confidence, metadata:{architecture,iterations,proposal_intent}}`. |
| `PATCH /{session_id}/section/{section_id}` | Body `SectionUpdateRequest` | `{success, section_id, word_count, status, quality_score}`; HTTP 400 bij lock/mislukking. |
| `PATCH /{session_id}/section/{section_id}/status` | Body `SectionStatusRequest` | `{success, section_id, status}`. |
| `POST /{session_id}/section/{section_id}/generate` | Body `SectionGenerateRequest` | SSE; zie apart schema hieronder. |
| `POST /{session_id}/section/{section_id}/evaluate` | Geen body | Toolresultaat als vrij JSON-object; bij succes onder meer `evaluated` en `score`. Er is geen response-model dat dit contract valideert. |
| `POST /{session_id}/section/{section_id}/analyze-gaps` | Geen body | Toolresultaat als vrij JSON-object. Er is geen response-model. |
| `POST /{session_id}/focus/{section_id}` | Geen body | `{success:true, active_section_id}`; 404 bij onbekende sessie/sectie. |
| `GET /health` | Geen body | Beoogd: `{status, architecture, components, errors, timestamp}`. Zie routeconflict hieronder. |

**Geverifieerde routefout:** `GET /{session_id}` staat vóór `GET /health` en accepteert iedere string. Door Starlette/FastAPI-routevolgorde wordt `/api/v4/proposal-session/health` daardoor als sessie-ID `health` behandeld. De healthhandler is via dat pad momenteel effectief onbereikbaar en geeft doorgaans `NOT_FOUND` in plaats van componenthealth.

### 4.3 V4 chat-SSE

`POST /{session_id}/message/stream` kan de volgende werkelijk uitgezonden events doorgeven:

| `type` | Payloadvelden |
|---|---|
| `status` | `stage`, `message`; agentstatus bevat ook `proposal_intent`. Er zijn twee initiële processingstatussen: één uit de route en één uit de agent. |
| `agent_thinking` | `iteration`, `thought`, `action`. |
| `tool_call` | `iteration`, `tool_name`, `display_name`, `parameters`, `reasoning`. |
| `tool_result` | `tool_name`, `success`, `result_count`, `execution_time_ms`. |
| `section_update` | `section_id`, `tool_name`, `content`. De route schrijft de volledige content naar sessiestate terwijl de sectie tijdelijk gelockt is. |
| `agent_complete` | `result` met `answer`, `success`, `sections_affected`, `iterations`, `confidence`, `total_time_ms`, `proposal_intent`. |
| `session_state` | Laatste event met `completion_percentage`, `total_words`, `sections_affected`. |
| `error` | `error: string`. Een onbekende sessie wordt ook zo gemeld binnen een HTTP 200-stream. |

Ondanks de routedocstring emitteert dit chatpad momenteel geen `token`-events: het volledige antwoord staat in `agent_complete.result.answer`. De frontendtypes noemen ook `chunk`, maar de bekeken V4-producent emitteert dat niet.

`POST /{session_id}/section/{section_id}/generate` heeft een afzonderlijk, kleiner eventcontract:

| `type` | Payloadvelden |
|---|---|
| `status` | `message`, `section_id`. |
| `content` | `section_id`, volledige `content`. |
| `complete` | `section_id`, `word_count`. |
| `error` | `error`. |

Voorbeeld create en chat:

```bash
curl -X POST 'http://localhost:8000/api/v4/proposal-session/create?user_id=demo-user' \
  -H 'Content-Type: application/json' \
  -d '{
    "proposal_id": 42,
    "call_id": "HORIZON-CL5-2026-01-D2-01",
    "title": "GREENLOOP",
    "proposal_type": "RIA"
  }'

curl -N -X POST \
  'http://localhost:8000/api/v4/proposal-session/SESSION_UUID/message/stream' \
  -H 'Content-Type: application/json' \
  -d '{"message":"Schrijf een eerste versie van excellence","section_context":"excellence"}'
```

## 5. Overige gemounte kernendpoints

Deze routes zijn in `app/main.py` gemount en gebruiken evenmin de backend-authdependency.

### 5.1 Feedback — prefix `/api/feedback`

| Methode en pad | Input | Respons |
|---|---|---|
| `POST /api/feedback` | JSON: verplichte `conversation_id: string`, `rating: 1..5`, `feedback_type: string`; optioneel `feedback_text`, `message_id`, `query_text`, `response_text`, `tools_used`, `query_type`, `response_time_ms` | `{success, feedback_id?, message?, error?}`. `feedback_type` is alleen beschreven, niet als enum gevalideerd. `response_text` wordt bij opslag tot 500 tekens beperkt. |
| `GET /api/feedback/summary?days=30` | `days` 1..365 | Vrij aggregatieobject. |
| `GET /api/feedback/conversation/{conversation_id}` | Pad-ID | `{conversation_id, feedback, count}`. |
| `GET /api/feedback/suggestions` | Geen | Vrij suggestieobject/array. |
| `GET /api/feedback/health` | Geen | `{status, enabled, message}`. |

### 5.2 Documentingestie — prefix `/api/documents`

| Methode en pad | Input/samenvatting |
|---|---|
| `POST /upload` | `multipart/form-data`: `file` verplicht; optioneel `document_type`, `programme_year`, `metadata` als JSON-string. Ondersteunde extensies en standaardlimiet (50 MB) komen uit backendsettings. Verwerking start als in-process background task. |
| `POST /ingest-url` | Queryparameters `url`, optioneel `document_type`, `programme_year`; optionele JSON-bodyparameter `metadata`. Alleen een vaste lijst EU-domeinen wordt geaccepteerd. |
| `GET /list` | `limit=50` (1..100), `offset=0`, optioneel `status`, `document_type`. |
| `GET /{document_id}` | Optioneel `include_chunks=false`. |
| `GET /{document_id}/status` | Ingestiestatus. |
| `POST /{document_id}/reprocess` | Start background reprocessing. |
| `DELETE /{document_id}` | Verwijdert databaseobject en probeert lokaal bestand te verwijderen. |
| `GET /{document_id}/chunks` | `limit=50` (1..100), `offset=0`. |
| `GET /stats/overview` | Documentstatistieken. |
| `POST /batch-upload` | Multipart `files`; maximaal tien; ongeldige bestanden worden overgeslagen. |

**Routevolgorderisico:** `GET /{document_id}` is vóór `GET /stats/overview` gedeclareerd. Een verzoek aan `/api/documents/stats/overview` heeft twee segmenten en matcht daarom wel de specifieke statsroute; er is hier geen conflict. Uploads zijn echter zonder auth bereikbaar en schrijven naar lokale opslag.

### 5.3 Call-documenten — prefix `/api/call`

| Methode en pad | Input | Respons |
|---|---|---|
| `POST /{call_id}/ask` | Body `{question: string, call_id?: string|null, max_chunks: int = 10}` | `{answer, call_id, chunks_used, sources: string[], cached, processing_time_ms}`. De body-`call_id` wordt niet gebruikt; de padwaarde is leidend. Geen grenzen op `max_chunks`. |
| `GET /{call_id}/documents` | Pad-ID | `{call_id, documents_count, chunks_count, cached, documents}`. |

De askroute gebruikt een hardcoded modelnaam `gemini-2.0-flash-exp` en retourneert 500 wanneer databasepool of Gemini niet kan initialiseren.

### 5.4 Proposal-documenten

Naast V4 bestaat een aparte CRUD-API onder `/api/proposal-document` (`proposal_document_routes.py`) voor live documentediting. De SvelteKit-routes `/api/proposal-document` en `/api/proposal-document/[...path]` proxyen hiernaar. Dit is een tweede proposalcontract naast de V4-sessiestate; IDs en state moeten door de caller consistent worden gehouden. De routes hebben geen FastAPI-authdependency of ownershipfilter.

## 6. SvelteKit-proxymatrix

| Browserpad | Backendpad | Authgedrag |
|---|---|---|
| `/api/proxy/conversation/{id}/message/stream` | `/api/v3/conversation/{id}/message/stream` | Controleert Supabase `getSession()`, maar niet `getUser()`; maakt proxy-JWT; geeft geen `user_id` mee, dus backend gebruikt `1`. |
| `/api/proxy/conversation/{id}/message` | `/api/v3/conversation/{id}/message` | Geen sessiecheck; stuurt alleen JSON Content-Type door; backend gebruikt `user_id=1`. |
| `/api/proxy/conversation/{id}/context` GET | `/api/v3/conversation/{id}/context` GET | Geen sessiecheck. |
| `/api/proxy/conversation/{id}/context` POST | Niet-bestaande backend-`POST .../context` | Contractbreuk; verwacht doorgaans HTTP 405. |
| `/api/proxy/conversation/{id}/reset-context` POST | `/api/v3/conversation/{id}/context` DELETE | Geen eigen sessiecheck. |
| `/api/v4/[...path]` | `/api/v4/{path}` | Transparante catch-all; geen eigen sessie-/ownershipcheck. |

De frontend gebruikt `BACKEND_URL`, met fallback `http://localhost:18000`. Een browserclient hoort deze same-origin proxies te gebruiken en niet de FastAPI-poort publiek bloot te stellen; dat is momenteel echter alleen een deploymentadvies, geen backendbeveiliging.

## 7. Foutcontract

Voor normale JSON-routes vervangen de globale FastAPI-handlers standaard `HTTPException`- en validatiefouten door:

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Request validation failed: 1 error(s)",
  "recoverable": false,
  "suggestion": "Please check your input and ensure all required fields are provided in the correct format.",
  "details": {
    "validation_errors": [
      {"field": "body -> message", "message": "Field required", "type": "missing"}
    ]
  }
}
```

Mogelijke optionele velden zijn `suggestion`, `details` en `request_id`. Iedere HTTP-response krijgt via middleware een `X-Request-ID`-header. Veel routehandlers vangen echter interne exceptions en zetten de ruwe exceptiontekst in `HTTPException(detail=...)`; de globale handler publiceert die vervolgens als `message`. Dit kan implementatiedetails lekken.

SvelteKit-proxies hanteren niet hetzelfde schema: daar komen onder meer `{error: string}`, SvelteKit errorresponses en ongewijzigde backendresponses voor. SSE-fouten gebruiken altijd een event zoals `{"type":"error","error":"..."}` en houden meestal HTTP 200 omdat headers al zijn verzonden.

## 8. Compatibiliteits- en wijzigingsregels

- Voeg geen V3/V4-eventtype of verplicht veld toe zonder frontendtype en parser tegelijk te wijzigen.
- Behoud onbekende SSE-events client-side als forward-compatible: log ze en negeer ze veilig.
- Behandel vrije toolresultaten (`evaluate`, `analyze-gaps`, feedbackaggregaties) als instabiel totdat response-modellen zijn ingevoerd.
- Maak auth/identity eerst expliciet breaking-changewaardig: na de fix moet `user_id` uitsluitend uit een gevalideerde principal komen en niet meer uit een caller-queryparameter.
- Voeg contracttests toe voor de vijf proxypaden in de matrix, met bijzondere aandacht voor SSE, HTTP-fouten en session/ownership denial.

## 9. Geverifieerde bronbestanden

- `MAP-IT_SA/app/main.py`
- `MAP-IT_SA/app/api/conversation_routes_v3.py`
- `MAP-IT_SA/app/api/proposal_session_routes.py`
- `MAP-IT_SA/app/api/feedback_routes.py`
- `MAP-IT_SA/app/api/document_routes.py`
- `MAP-IT_SA/app/api/call_document_routes.py`
- `MAP-IT_SA/app/api/proposal_document_routes.py`
- `MAP-IT_SA/app/api/error_handlers.py`
- `MAP-IT_SA/app/core/auth.py`
- `MAP-IT_SA/app/services/conversation/conversation_manager.py`
- `MAP-IT_SA/app/services/agents/tool_aware_agent.py`
- `MAP-IT_SA/app/services/agents/proposal_aware_agent.py`
- `MAP-IT_SA/app/services/proposal_session/session_state.py`
- `webapp-ui/src/hooks.server.ts`
- `webapp-ui/src/lib/server/jwt.ts`
- `webapp-ui/src/lib/clients/proposalSessionService.ts`
- `webapp-ui/src/routes/api/proxy/conversation/**/+server.ts`
- `webapp-ui/src/routes/api/v4/[...path]/+server.ts`
