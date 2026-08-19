# Progress log — Wave 3 P1 documentation improvements

**Date:** 20 July 2026

**Scope:** documentation only in `NEWMAPIT` and `MAP-IT_SA`; no code or
configuration changes

**Final status:** all five requested P1 tasks delivered and checked against repository sources

## Delivered

| Task | Document | Action | Result |
|---|---|---|---|
| Environment matrix | `MAP-IT_SA/docs/ENVIRONMENT_MATRIX.md` | New | 104 variables from backend, frontend, Compose, tests, and CI; required/conditional, secret/public, defaults, environment, and description included. |
| Tool reference | `MAP-IT_SA/docs/guides/TOOL_REFERENCE.md` | Completely rewritten | Exactly 42 registered tools, grouped into the six requested categories, with types, runtime defaults, behavior, and data stores. |
| Dependency matrix | `DEPENDENCY_MATRIX.md` | New | Runtimes, 27 backend runtime packages, 6 dev packages, 4 extractor packages, 52 direct npm packages, infrastructure, and AI models documented. |
| Business handover | `HANDOVER_SUMMARY.md` | New | 567-word management summary with status, five top risks, work performed, next steps, and contact. |
| Documentation index | `MAP-IT_SA/docs/README.md` | Completely rewritten | Project, backend, and relevant frontend docs divided into Current, Historical/PRD, and Archived; `START_HERE.md` is the entry point. |
| Wave log | `WAVE3_PROGRESS_LOG.md` | New | This verification and handover log. |

## Key findings

### Environment

- The frontend uses two different database variables: the SvelteKit runtime reads
  `PRIVATE_DATABASE_URL`, while `drizzle.config.ts` requires `DATABASE_URL` for Drizzle Kit.
  Both must be set server-side; the configuration was deliberately not modified.
- All 35 statically recognizable `os.getenv`/`os.environ` names from the readable Python source are in the
  matrix. All 72 unique names from both `.env.example` files and Compose, and all 14
  statically recognizable frontend names, are also covered.
- Code, example files, and Compose differ in areas including vector search, embedding model,
  database pool, and Redis host. These differences are explicitly stated rather than artificially
  harmonized.
- `MAX_CONTEXT_*`, `LLM_TIMEOUT_SECONDS`, retrieval timeouts, call enrichment tuning, and the four
  existing test API URL aliases are included.
- `MAP-IT_SA/neo4j/import` was unreadable due to OS permissions. This is a container data volume;
  all Python source accessible through the project file list was checked.

### Tools

- `ToolRegistryMixin._init_tools()` contains exactly 42 unique registered tools. The reference contains
  exactly the same 42 unique tool rows: none are missing and no extras are presented as available tools.
- The distribution is Statistics 7, Comparison 1, Document 5, Call 10, Project 11, and Proposal support 8.
- Registry text and executor differ in several places. Four registered parameters are
  ignored, multiple executor parameters are not described to the LLM, `search_documents`
  uses runtime default 15, and two project search tools retrieve at least 100 candidates.
- `search_digital_europe_templates_execute()` exists, but is not registered or routed
  and was therefore correctly not included as a 43rd tool.

### Dependencies and runtimes

- The handover baseline is Python 3.11+, Node 20+, PostgreSQL 15+ with pgvector, Neo4j 5.x, and Redis
  7.x. Known drift is documented: CI uses Python 3.11, the backend container 3.13, and an old
  deployment guide still mentions 3.9.
- All direct Python requirements and all 52 direct npm dependencies are in the matrix. The
  manifest/lockfile versions were compared automatically per row; there were no discrepancies.
- Python is not fully locked, the Supabase CLI is not pinned by the repository, container tags
  are not immutable, and PostgreSQL 15 versus the optional pg16 service requires a deliberate
  upgrade decision.
- Central and hardcoded Gemini, OpenAI, OpenRouter, embedding, and rerank models were inventoried
  separately. Model aliases such as `preview` and `latest` are marked as reproducibility risks.

### Handover and document lifecycle

- The original 37/100 from `DOCUMENTATION_REVIEW.md` is correctly identified in the handover text as
  a baseline measurement, not as a remeasured score after the three waves.
- The product status remains **not production-ready**: the final quality baseline remains 23/42
  scenarios (54.8%), and documentation does not resolve the open product, data, security, and operations risks.
- The old index claim “no redundancy” was removed. PRDs, old roadmaps, V2/Celery documents, and
  implementation snapshots are no longer collectively presented as the current single source of truth.
- All eight files with an existing archive banner are listed under Archived in the index.

## Verification performed

- Context from `DOCUMENTATION_PROGRESS_LOG.md`, `WAVE2_PROGRESS_LOG.md`,
  `DOCUMENTATION_REVIEW.md`, and `KNOWN_ISSUES_AND_RISKS.md` was reused and relevant claims
  reconciled with the current wave.
- Tool names were extracted from the registry via an independent AST check and compared with the 42 tool rows in the
  guide: no missing or extra names.
- Environment names were independently compared against Python AST, both environment examples, Compose, and
  static frontend environment consumers: no missing static names.
- Direct Python and npm manifests plus npm lock versions were automatically compared with the dependency matrix:
  no missing dependency or version discrepancy.
- All local Markdown links in the five substantive Wave 3 documents and this progress log exist.
- Markdown code fences in the six Wave 3 documents are balanced.
- Trailing whitespace checks on all six documents and `git diff --check` on the tracked
  backend documents passed.
- No commit was made.

## Deliberately not modified

This wave was documentation-only. Existing application code, environment examples,
`docker-compose.yml`, manifests, lockfiles, migrations, CI, and other configuration were not modified by
Wave 3. Pre-existing non-documentation worktree changes were preserved and fall
outside this wave.
