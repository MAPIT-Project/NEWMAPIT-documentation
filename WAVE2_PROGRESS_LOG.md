# Progress log — Wave 2 P0 code fixes

**Date:** 20 July 2026

**Scope:** backend CI and test collection, frontend CI and check baseline, and migration documentation

**Final status:** all requested Wave 2 artifacts delivered; existing format, typecheck and integration test debt deliberately not broadly rewritten

## Delivered

| Component | Change | Result |
|---|---|---|
| Backend CI | `MAP-IT_SA/.github/workflows/ci.yml` added | Push to `main` and pull requests; Python 3.11; `make install-dev`; blocking Ruff and Black checks; non-blocking tests with an always-uploaded JUnit artifact. The workflow names PostgreSQL/Supabase/pgvector, Redis, Neo4j, backend and provider credentials as possible test dependencies. |
| Pytest collection | `MAP-IT_SA/pyproject.toml` narrowed from `testpaths = ["tests", "."]` to `testpaths = ["tests"]` | Pytest no longer recursively enters runtime/volume areas such as the inaccessible `neo4j/import/`. No tests were removed or skipped. |
| Backend test documentation | `MAP-IT_SA/tests/README.md` added | Reproduction commands, external services/env vars and the temporary non-blocking CI test policy recorded. |
| Frontend test script | Only `"test": "npm run check"` added to `webapp-ui/package.json` | All existing scripts and `package-lock.json` preserved. |
| Frontend CI | `webapp-ui/.github/workflows/ci.yml` added | Push to `main` and pull requests; Node 20; `npm ci`; non-blocking `npm run check`; blocking production build. |
| Frontend baseline | `webapp-ui/CHECK_ISSUES.md` added | All 64 errors and 76 warnings in 34 files fully, mutually exclusively categorised and counted per file. No application source code forcibly changed. |
| Migration manifest | `MAP-IT_SA/migration/MIGRATION_LEDGER.md` added | Current files, SHA-256 checksums, statuses, dependencies, snapshot relationship, vector transition and a safe fresh-database decision path documented. No migration created or changed. |

## Backend verification

- Diagnosis before the fix: `pytest --collect-only` found 3,575 tests, but ended with `PermissionError` on `neo4j/import/conftest.py` and exit code 2.
- After the fix: `pytest --collect-only -q` collects 3,575 tests with exit code 0; repeated centrally on 20 July 2026.
- `ruff check app tests`: passed.
- `tests/validators/test_failure_detector.py`: 53 tests passed.
- The full `make test` run got past collection, but was terminated after 60 seconds. With `pytest -q -x` the first runtime failure was `tests/quality/test_phase1_scenarios.py::test_programme_discovery`: no backend on `localhost:8000`, no Google API key and therefore score 45 below the threshold 80. A results file temporarily modified by this test was exactly restored.
- A separate historical workflow test still expects the non-existent `.github/workflows/conversation-tests.yml`: 53 tests passed, 1 failed and 18 were skipped in the combined targeted run. That test was not modified, because Wave 2 explicitly asks for `ci.yml` and the intended conversation workflow contract cannot be safely inferred.
- `black 24.8.0 --check app tests`: expected baseline failure; 135 existing files would be formatted and 237 remain unchanged. No broad format rewrite was performed. The new CI format gate will therefore stay red until this is resolved as a separate, reviewable formatting wave.
- A literal local `make lint` is additionally affected by already-present untracked Python scripts/tests and the inaccessible Neo4j volume directory. The requested workflow does literally use `make lint`; in a clean checkout local volume and non-committed files do not exist, unless deliberately added later.
- Both workflow files were validated as YAML; `package.json` was validated as JSON.

## Frontend verification

- `npm ci`: succeeded; `package-lock.json` remained unchanged.
- `npm test`: now runs `svelte-check` and ends with the documented baseline of 64 errors and 76 warnings in 34 files (exit code 1).
- `npm run build`: succeeded with existing warnings, including accessibility/CSS/Svelte migration notices and a bundle larger than 500 kB.
- `npm ci` reported 35 existing audit findings: 3 low, 16 moderate, 15 high and 1 critical. No automatic dependency upgrade was performed due to the regression risk.
- Zero frontend source code fixes were performed in this wave; the allowed maximum of five was therefore not exceeded.

## Migration findings

- The current SQL directory is not a linear, reproducible chain. Exact applied statuses are not demonstrable and are therefore set to `unknown`.
- Git history proves that 003–007 existed and were removed on 13 May 2026 in commit `bb90f94`. There were two different migrations with number 005; later 010–022 files were also removed in that cleanup.
- `schema.sql` is a broad Supabase/PostgreSQL snapshot, not an incremental successor to `001`. The file moreover contains four bare `pg_dump:` diagnostic lines and is not blindly executable as a fresh bootstrap.
- The current, already-present-before-Wave-2 untracked `002_upgrade_embeddings_to_3072.sql` handles the destructive transition for three embedding columns from 768 to 3072 dimensions, but does not apply directly to `001` because required model columns are missing there.
- The six SHA-256 checksums recorded in the ledger were recalculated and match. No SQL file was changed.

## Deliberately left open

1. Backend CI will fail on the current Black baseline; automatically reformatting 135 files falls outside this cautious handover wave.
2. The full backend test suite requires tiering and reproducible fixtures/services before tests can be made blocking.
3. The frontend check remains non-blocking until the 64 categorised errors are resolved in phases; the build is green.
4. The dependency audit findings require a separate upgrade and regression test round.
5. There is not yet an executable authoritative fresh-database chain. First one schema authority must be chosen and tested in CI on an empty compatible Supabase instance.

## Worktree hygiene

Nothing was committed. Changes, dumps, scripts and loose test files already present before Wave 2 in both repositories were preserved and not substantively modified. These Wave 2 changes must be explicitly distinguished from that existing worktree content in the final commit.
