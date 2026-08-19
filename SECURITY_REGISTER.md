# MAPIT security register

**Status:** open security baseline — not suitable as production approval  
**Reference date:** 20 July 2026  
**Owner:** TBD  
**Reassessment:** before every external deployment and at least monthly while critical/high items remain open

## 1. Executive summary

In the current code baseline, MAPIT is **not secure for public production exposure**. The primary cause is not a single exposed secret, but the absence of an enforced identity and authorization chain: the browser authenticates with Supabase, while the FastAPI V3/V4 routes do not validate that identity and use caller-controlled/default user IDs. In addition, the tracked Supabase schema shows broad grants and Row Level Security (RLS) on only two of 35 public tables.

All critical items in this register are release blockers. This document records the current state; it does not implement code fixes or secret rotations. See [API_CONTRACT.md](API_CONTRACT.md) for the exact route and identity contract and [KNOWN_ISSUES_AND_RISKS.md](KNOWN_ISSUES_AND_RISKS.md) for the broader status register.

## 2. Severity and status

| Severity | Meaning |
|---|---|
| **Critical** | Likely unauthorized access, cross-tenant breach, secret compromise, or broad data mutation; blocks production. |
| **High** | Severe confidentiality, integrity, or availability impact with a realistic exploitation path; blocks production unless demonstrably mitigated. |
| **Medium** | Limited or conditional impact; resolve before broad rollout. |
| **Low** | Hardening, hygiene, or defense-in-depth. |

Status values: `open`, `in-progress`, `accepted`, `resolved`. All findings below have status `open`; owner is `TBD` unless assigned.

## 3. Baseline threat model

### Assets to protect

- Supabase identities, access/refresh tokens, and user profiles.
- Personal and collaboration conversations, messages, and citations.
- Proposal content, consortium information, evaluations, budgets, and generated documents.
- Funding call, document, embedding, and analytics data; some source data is public, derived user data is not.
- Database, Supabase service-role, Neo4j, JWT, and AI provider credentials.
- Availability and cost quotas for Gemini/OpenAI/OpenRouter, database, Redis, and Neo4j.
- Auditability: reliable attribution of mutations to a single validated principal.

### Attackers and relevant scenarios

- An unauthenticated internet user who calls FastAPI or unsecured SvelteKit API routes directly.
- A valid user who guesses IDs and attempts to read or modify another user's or collaboration's data (IDOR/BOLA).
- An attacker with XSS who captures a browser-readable access token.
- A malicious uploader or external document source that triggers parser, prompt injection, or resource exhaustion behavior.
- An actor who uses a known fallback/default credential or exhausts API quotas.
- A misconfigured deployment in which a service-role key ends up in the browser bundle through a `PUBLIC_`/`VITE_` variable.

### Security objectives

1. Every private request has one cryptographically validated identity.
2. Every object access validates ownership, collaboration membership, and role server-side.
3. RLS provides database-level defense-in-depth; `anon` receives only explicitly public read access.
4. Secrets never appear in Git, logs, frontend bundles, or error responses.
5. Mutating cookie-authenticated requests are protected against CSRF; browser code is hardened against XSS.
6. Expensive and mutating endpoints have rate, size, and concurrency limits.

### Not established by this review

No penetration test, dependency CVE scan, cloud configuration review, secret history scan, or live Supabase policy check was performed. Claims about RLS are based on `webapp-ui/supabase_cli/schema.sql`; verify the actual staging and production databases separately. A schema file is not evidence that the same state exists in the live environment.

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

Primary boundary failure: SvelteKit knows the verified Supabase user, but FastAPI does not receive a reliably enforced mapping of that user to a backend principal. The backend is therefore a separate trust boundary and must not rely solely on the presence of the proxy.

## 5. Known vulnerabilities and risks

| ID | Severity | Finding and impact | Verified location | Proposed fix |
|---|---|---|---|---|
| SEC-001 | Medium | **Hardcoded Mapbox public token.** A Mapbox public token is browser-visible by design, but a token in source code cannot be securely managed per environment, remains in Git history, and can be misused outside permitted origins/quotas if scopes/restrictions are too broad. | `webapp-ui/src/routes/(protected)/dashboard/consortium/+page.svelte:10-11` and usage on line 253. | Rotate the token; create a new least-privilege public token with URL restrictions and quotas; load it through an intentionally public `PUBLIC_MAPBOX_TOKEN`; scan Git history. Never move a required browser-public key to a server secret without adapting the Mapbox flow. |
| SEC-002 | Critical | **Predictable JWT fallback in SvelteKit.** If `JWT_SECRET_KEY` is missing, the proxy signs with the known string `fallback-secret-change-in-production`. As soon as backend authentication trusts these tokens, anyone can create valid HS256 tokens with arbitrary roles/sub values. Authentication is already open now; the fallback makes a later “enable auth” change dangerous. | `webapp-ui/src/lib/server/jwt.ts:10-13`. Backend behavior differs: `MAP-IT_SA/app/core/auth.py:65-73` generates a random secret per process if the env variable is missing. | Fail closed at startup if `JWT_SECRET_KEY` is missing; use a minimum 256-bit random secret from a secret manager; use identical, explicit key configuration on both sides; establish a key ID/rotation plan. Preferably, FastAPI validates a Supabase JWT directly through a supported asymmetric/JWKS flow instead of a proxy identity. |
| SEC-003 | High | **Access cookies are browser-readable (`httpOnly:false`).** An XSS vulnerability can read and exfiltrate the Supabase access token. The login route enforces this in both SSR cookie options and manual access-token logic. | `webapp-ui/src/routes/api/auth/login/+server.ts:45-53` and `:101-115`. | Set authentication cookies to `httpOnly:true`, `secure:true`, with an appropriate `sameSite`; let `@supabase/ssr` manage one consistent cookie flow. Remove manual token headers/cookies and test login, refresh, and logout. Add CSP and XSS tests. |
| SEC-004 | Critical | **`PUBLIC_SUPABASE_SERVICE_KEY` is a misleading public variable name.** The example instructs users to place a service-role key under `PUBLIC_`; SvelteKit treats public env variables as browser-exposable. The current code does not import this name, but copying/using it would bypass RLS completely. | `webapp-ui/.env.example:20-21`. The correct private variant is at `:27-28` and is used server-side in collaboration routes. | Remove the public variant everywhere; use only `SUPABASE_SERVICE_ROLE_KEY` through `$env/static/private`/secret manager; add a build check that rejects service-role/secret names under `PUBLIC_`/`VITE_`; rotate the key if it ever appeared in an actual public env or bundle. |
| SEC-005 | High | **Global SvelteKit rate limiting is disabled.** All `/api/*` limiting in the hook is commented out. Login/register/query/AI-message do have individual in-memory limiters, but V3/V4 catch-all, CRUD, and other endpoints are not uniformly covered; a horizontally scaled deployment does not share the counters. | `webapp-ui/src/hooks.server.ts:8-46`; TODO on lines 10-14. The backend has `RateLimitMiddleware`, but `app/main.py` does not register it. | Enable Redis-backed limiting at the edge/proxy and backend; set tiers for login, AI, ingest, and mutation; trust proxy IP headers only from a known reverse proxy; return 429/Retry-After; load-test and monitor rejections. |
| SEC-006 | Critical | **V3/V4 and other FastAPI routes have no authentication.** `get_current_user` exists but is not used; `AuthMiddleware` is not mounted. V3 uses default `user_id=1`; V4 create uses `default-user`; IDs are caller-controlled. Reads, writes, deletions, AI costs, and analytics exports are therefore directly accessible. | `MAP-IT_SA/app/api/conversation_routes_v3.py:41-42,204-207`; `proposal_session_routes.py:187-191`; routers in `app/main.py`; auth dependency defined only in `app/core/auth.py`. | Add a deny-by-default authentication dependency to all private routers; validate token issuer, audience, signature, expiry, and type; derive the user exclusively from the principal; add an ownership/membership check for each object; remove user ID query parameters/defaults; network-isolate FastAPI until this is complete. |
| SEC-007 | High | **Collaboration membership checks are missing/inconsistent.** Direct access to collaboration conversations is not robustly checked on some GET/PATCH paths; the chat websocket/message path has an explicit TODO; DELETE does check membership. This creates IDOR and unauthorized mutation. | `webapp-ui/src/routes/api/conversations/[conversation_id]/+server.ts:58-90` and `:169-187`; `webapp-ui/src/routes/api/chat/websocket/+server.ts:48-68`. | Centralize `requireCollaborationMember`/`requireRole`; apply them to reads, writes, messages, invitations, locks, websocket upgrades, and deletes; query membership with the verified `client.id`; add negative tests for non-member, removed member, and role boundaries. |
| SEC-008 | Critical | **Minimal RLS plus broad grants in the schema.** Of 35 `public` tables, only `document_chunks` and `horizon_documents` have RLS in the snapshot. At the same time, many tables have `GRANT ALL ... TO anon, authenticated`. Without RLS, PostgREST can expose broad user, conversation, proposal, and analytics data or allow it to be mutated. | `webapp-ui/supabase_cli/schema.sql:8677-8718` (four policies/two tables); grants start around `:13628` and include `client`, `conversation`, `message`, `proposals`, `proposal_sessions`, among others. | Revoke default `ALL` from `anon`/`authenticated`; define SELECT/INSERT/UPDATE/DELETE policies per table using `auth.uid()` and membership relationships; force RLS where appropriate; keep the service role server-side only; test policies with anon, owner, member, and outsider. Compare the live DB with the schema. |
| SEC-009 | High | **The V3 streaming proxy validates with `getSession()` and does not use the actual user.** Server-side `getSession()` trusts session data without the same network validation as `getUser()`. The code reads `session.user.id` but then always creates a token for `webapp-proxy`; the backend still uses default `1`. | `webapp-ui/src/routes/api/proxy/conversation/[conversation_id]/message/stream/+server.ts:13-27`. | Use the `locals.user` already validated by the hook, or `supabase.auth.getUser()`; pass a validated user context according to a single backend contract; have the backend cryptographically validate and authorize it again. |
| SEC-010 | Critical | **Proxies are not a security boundary.** The V3 non-streaming/context/reset proxies and V4 catch-all do not perform a uniform session or ownership check. Direct FastAPI traffic can bypass the proxy entirely. | `webapp-ui/src/routes/api/proxy/conversation/**/+server.ts`; `webapp-ui/src/routes/api/v4/[...path]/+server.ts`; `MAP-IT_SA/app/main.py`. | Backend authentication is mandatory, independently of the proxy. Also add same-origin session checks; expose FastAPI only on a private network/loopback behind NGINX; firewall ports 8000/18000 externally; add contract tests for 401/403. |
| SEC-011 | High | **Document ingestion and destructive routes are open.** Upload, URL ingest, batch upload, reprocess, and delete require no principal. This enables storage/AI resource exhaustion, content injection, and data deletion. | `MAP-IT_SA/app/api/document_routes.py`; `proposal_document_routes.py`; V4 delete/update/generate routes. | Require admin/editor roles and ownership checks; enforce a streaming upload limit before fully reading the file; sandbox malware/parser processing; validate content types; use an SSRF-safe allowlist with DNS/IP revalidation; add quotas and an audit log; use soft delete/confirmation for destructive actions. |
| SEC-012 | Medium | **Overly broad default CORS configuration.** FastAPI defaults to `CORS_ORIGINS=["*"]`, all methods/headers, and `allow_credentials=True`. This is unsuitable as a production policy, and browser behavior around wildcard+credentials is error-prone. Some SvelteKit proxies also add `Access-Control-Allow-Origin: *`. | `MAP-IT_SA/app/core/config.py:20`; `app/main.py:42-49`; including V3 streaming proxy lines 53-61. | Use an explicit HTTPS origin allowlist per environment; allow only required methods/headers; do not use a wildcard with credentials; have only one layer set CORS headers; add automated preflight tests. CORS is not a substitute for authentication/CSRF. |
| SEC-013 | High | **No explicit CSRF defense for mutating cookie-authenticated routes.** Cookies use `sameSite:lax`, but no CSRF token or consistent Origin/Referer validation was found. SameSite helps, but is not a complete policy for all browser flows/subdomains. | Cookie configuration in the login route; no `csrf` implementation found in `webapp-ui/src` or `MAP-IT_SA/app`. | For mutations: use a CSRF token or strict Origin/Host check; deliberately choose `SameSite=Lax/Strict`; ensure GET never mutates; restrict CORS; test cross-site form/fetch and sibling-subdomain attacks. |
| SEC-014 | Medium | **Raw internal error details and agent reasoning are sent to clients.** Many handlers put `str(e)` in HTTP or SSE responses. V3/V4 emits `thought`, `reasoning`, tool parameters, and summaries. This can leak schema, prompt, data, or provider details. | `conversation_routes_v3.py`, `proposal_session_routes.py`; the global HTTP handler publishes `HTTPException.detail` in `app/api/error_handlers.py:125-165`; agents emit reasoning. | Expose stable error codes and generic messages externally; provide a request ID for support; keep details only in server logs with redaction. Do not emit chain-of-thought; replace it with brief, safe status/tool activity without secrets or private context. |
| SEC-015 | High | **Default Neo4j credential is in code and the env example.** If a deployment does not set the variable, the client uses a known password value. Compose/configuration may also normalize this credential as a development default. | `MAP-IT_SA/app/core/neo4j_config.py:46-48`; `MAP-IT_SA/.env.example` at `NEO4J_PASSWORD`. | Do not provide a password default; fail closed outside an explicit local development mode; use a unique secret per environment; do not expose Neo4j publicly; use TLS when remote; rotate the existing credential and inspect logs/history. |
| SEC-016 | Medium | **Incorrect/manual cookie and header management in login.** The route calls `response.headers.set('sb-access-token', token, options)` as though it were a cookie API; `Headers.set` has no cookie options contract. This can cause inconsistent sessions and token exposure through custom response headers alongside the SSR cookie flow. | `webapp-ui/src/routes/api/auth/login/+server.ts:101-131`. | Use only SvelteKit `cookies.set`/Supabase SSR `setAll`; do not put tokens in ordinary response headers or the response body; add an integration test verifying that exactly the intended `Set-Cookie` attributes are produced. |

## 6. Authentication/authorization and RLS decision

### Desired canonical model

1. Supabase Auth is the identity provider.
2. SvelteKit validates with `getUser()` for UX and same-origin API gating.
3. FastAPI validates every bearer token itself or trusts only a cryptographically verifiable internal token from an authenticating proxy. Network trust alone is insufficient.
4. `sub` is mapped server-side to `client.auth_id`; never trust a `user_id` from a query/body.
5. Resource handlers load the object plus owner/collaboration and decide between 404/403 without leaking object details.
6. PostgreSQL RLS repeats the same ownership/membership boundary as defense-in-depth.

### Minimum route policy

| Route group | Minimum requirement |
|---|---|
| Health/liveness | Public, but no secrets, exception text, or provider data. Deep dependency health only internally/for admins. |
| Funding calls and explicitly public documents | Authentication optional if this is a product decision; read only; RLS policy explicitly public. |
| V3 conversations/messages/context | Authentication required; owner or collaboration member; appropriate role for create/update/delete. |
| V4 proposal sessions/documents | Authentication required; proposal owner/editor; evaluation/generation subject to cost quotas and rate limiting. |
| Feedback submit | Authentication required or demonstrably abuse-resistant; derive the user/conversation relationship. |
| Feedback analytics, conversation analytics/export | Admin/analyst; never public. |
| Ingest/reprocess/delete | Admin/data steward; audit log and strict limits. |
| Collaboration invites/members/locks | Member for read; owner/admin for invite/role/delete; membership checked on every websocket/operation. |

### Live RLS verification before deployment

Run this against staging and production, not just the dump:

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

Release acceptance: every public table has a documented decision (`public read`, `user-owned`, `collaboration-scoped`, `admin-only`, `service-only`) and automated negative policy tests.

## 7. Secrets management

| Secret/credential | Where required | May be public? | Current configuration name |
|---|---|---|---|
| Supabase anon key | Browser and server SSR | Yes, with correct RLS | `PUBLIC_SUPABASE_ANON_KEY` |
| Supabase service-role key | Trusted SvelteKit/backend server only | **No** | `SUPABASE_SERVICE_ROLE_KEY` frontend; `SUPABASE_SERVICE_KEY` backend |
| JWT signing key | SvelteKit and backend while the custom HS256 flow exists | **No** | `JWT_SECRET_KEY` |
| PostgreSQL URL/password | Backend/SvelteKit server and migration job | **No** | `DATABASE_URL`, `PRIVATE_DATABASE_URL` |
| Neo4j password | Backend | **No** | `NEO4J_PASSWORD` |
| AI provider keys | Backend or specific server route | **No** | `GEMINI_API_KEY`, `GOOGLE_EMBEDDING_API_KEY`, `OPENAI_API_KEY`, `OPENROUTER_API_KEY`, frontend example `GOOGLE_API_KEY` |
| Static backend API key | Backend/client if used | **No** | `STATIC_API_KEY` |
| Mapbox public token | Browser | Yes, but restricted by origin/scope/quota | create `PUBLIC_MAPBOX_TOKEN` |

### Storage policy

- Production secrets only in the deployment secret manager or a root-readable service env; never in Git, a Docker image, a PM2 ecosystem file with world-readable permissions, a frontend public env, or logs.
- `.env` only locally, mode `0600`, in `.gitignore`; examples contain only clear placeholders and no working defaults.
- Do not log Authorization, Cookie, service-role, database URL, or provider key values. Also redact query strings when they may contain tokens.
- Separate local/staging/production projects, databases, Mapbox tokens, and AI keys.
- CI performs secret scanning on the current tree and Git history; a discovered secret is rotated first and then addressed in history.

### Rotation policy

- Rotate immediately upon suspected exposure, staff/vendor changes, or inclusion in a public bundle/Git.
- Rotate service-role, database, Neo4j, and AI keys at least every 90 days until central policy specifies otherwise; rotate the Mapbox public token at least annually and upon misuse.
- JWT rotation supports overlap through a key ID/two valid verification keys; the maximum token lifetime limits the overlap. Revoke existing sessions during an incident.
- Every rotation has an owner, date, test of the new key, and demonstrable revocation of the old key. Do not include secret values in tickets/documentation.

## 8. Cookies, CORS, and CSRF: current configuration

| Component | Current code | Assessment |
|---|---|---|
| Login access cookie | `secure:true`, `sameSite:'lax'`, explicitly `httpOnly:false`; manually set to 2 hours and domain `.mapit.work` | XSS risk; hardcoded production domain makes local/staging inconsistent. |
| Refresh cookie | Manually set to `httpOnly:true`, `secure:true`, `sameSite:'lax'`, 30 days, `.mapit.work` | Better baseline, but the duplicate/manual Supabase cookie flow must be removed and tested. |
| Other Supabase SSR cookies | `setAll` carries over library options and enforces `path:'/'` | Attributes are not centrally documented or tested. |
| FastAPI CORS | Origins default to `*`, credentials enabled, methods/headers `*` | Not acceptable for production; pin origins. |
| SvelteKit proxy CORS | Some routes add wildcard headers | Inconsistent and unnecessary for same-origin use. |
| CSRF | No token or central Origin check found | High open issue for cookie-authenticated mutations. |

Recommended production policy: HTTPS only; HSTS at NGINX; a host/origin allowlist for the actual frontend domain; authentication cookies with `Secure`, `HttpOnly`, the narrowest possible `Domain`/`Path`, and a deliberate `SameSite` setting; a CSRF token or central Origin validation on all mutations; a strict CSP without `unsafe-inline` where feasible.

## 9. Logging, privacy, and AI/data boundaries

- The hooks log the signed-in user's email address twice (`webapp-ui/src/hooks.server.ts:71-73`). Limit production authentication logs to pseudonymous user ID/request ID and avoid duplicate PII.
- Conversation analytics can store questions, answers, tools, and errors. Establish a retention period, legal basis, access/deletion rights, and access role before processing real user data.
- Prompts and proposal content leave the organization's boundary for AI providers. Classify the data, establish appropriate processing terms/DPAs, configure provider retention, and prevent secrets/personal data from being included unnecessarily in prompts.
- Document ingestion and tool results are untrusted content. Mark them as data, not instructions; add prompt injection tests and output validation.
- `agent_thinking`/`reasoning` should not be sent to the browser in production; show a safe activity summary.

## 10. Pre-deployment security checklist

An item is complete only with code, an automated test, and staging evidence.

### Release blockers

- [ ] SEC-002: known JWT fallback removed; a missing key causes the process to fail.
- [ ] SEC-004: no service-role key under `PUBLIC_`/`VITE_`; potentially exposed key rotated.
- [ ] SEC-006/010: FastAPI private routes return 401 without a token or with an invalid token, and 403/404 without object access rights; the public backend port is firewalled.
- [ ] V3/V4 user identity comes exclusively from the validated principal; defaults `1` and `default-user` have been removed from the security contract.
- [ ] SEC-007: collaboration membership/role is checked on every read/write/websocket path; negative tests pass.
- [ ] SEC-008: live RLS/grants inventoried and corrected per table; anon/outsider tests pass.
- [ ] SEC-011: ingest, AI mutations, and deletes are role-protected, audited, and limited.
- [ ] All working hardcoded/default credentials have been rotated; secret scan of the tree and history is clean.

### Web and platform hardening

- [ ] The access and refresh cookie flow uses one Supabase SSR mechanism; `HttpOnly`, `Secure`, `SameSite`, Domain, Path, and expiration have been tested.
- [ ] CSRF defense and an explicit CORS origin allowlist are active and demonstrated with negative cross-origin tests.
- [ ] CSP, HSTS, `X-Content-Type-Options`, `Referrer-Policy`, and frame policy are configured in NGINX/SvelteKit and have been tested.
- [ ] Global and endpoint-specific rate limits work across instances through Redis/the edge; 429 responses and alerts have been tested.
- [ ] Upload type, streaming size limit, parser isolation, SSRF/DNS rebinding protection, and storage quotas have been tested.
- [ ] FastAPI `/docs`/`openapi.json`, debug/test endpoints, and deep health details are internal or admin-only in production.
- [ ] Database, Redis, and Neo4j listen only on private interfaces/networks; firewalls and TLS have been assessed.

### Assurance and operations

- [ ] Dependency, container, and IaC scans have no unaccepted critical/high findings.
- [ ] Human code security review and external penetration testing cover authentication bypass, IDOR/BOLA, RLS, CSRF, XSS, upload/SSRF, and SSE.
- [ ] Audit logs record authentication, admin, ingest, proposal, and delete actions with a request ID, without overlogging secrets/PII.
- [ ] Incident response includes a contact owner, token/key revocation, Supabase session revocation, forensic log retention, and a notification decision.
- [ ] Backup/restore has been tested, and backups are encrypted, access-restricted, and subject to a retention policy.
- [ ] Privacy/AI provider review, data retention, and deletion procedure have been approved.

## 11. Acceptance criteria for “security ready”

MAPIT may be labeled security-ready only when:

1. no critical or unaccepted high finding remains open;
2. an independent reviewer has tested authentication/authorization/RLS end-to-end in staging;
3. the FastAPI route inventory automatically fails when a private route without a policy/dependency is added;
4. the live schema/grants/policies are drift-checked against version control;
5. secret rotation, logging, alerts, and incident response have been operationally tested;
6. the results are dated and signed off by a human security owner.

## 12. Verified sources

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
