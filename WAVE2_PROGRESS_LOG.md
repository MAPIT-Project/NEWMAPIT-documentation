# Progress log — Wave 2 P0-codefixes

**Datum:** 20 juli 2026

**Scope:** backend-CI en testcollectie, frontend-CI en checkbaseline, en migratiedocumentatie

**Eindstatus:** alle gevraagde Wave 2-artifacts opgeleverd; bestaande format-, typecheck- en integratietestschuld bewust niet breed herschreven

## Opgeleverd

| Onderdeel | Wijziging | Resultaat |
|---|---|---|
| Backend-CI | `MAP-IT_SA/.github/workflows/ci.yml` toegevoegd | Push naar `main` en pull requests; Python 3.11; `make install-dev`; blokkerende Ruff- en Black-checks; niet-blokkerende tests met altijd geüploade JUnit-artifact. De workflow benoemt PostgreSQL/Supabase/pgvector, Redis, Neo4j, backend en providercredentials als mogelijke testdependencies. |
| Pytest-collectie | `MAP-IT_SA/pyproject.toml` beperkt van `testpaths = ["tests", "."]` naar `testpaths = ["tests"]` | Pytest loopt niet meer recursief runtime-/volumegebieden zoals het ontoegankelijke `neo4j/import/` binnen. Er zijn geen tests verwijderd of geskipt. |
| Backend-testdocumentatie | `MAP-IT_SA/tests/README.md` toegevoegd | Reproductiecommando's, externe services/env-vars en de tijdelijke niet-blokkerende CI-testpolicy vastgelegd. |
| Frontend-testscript | Alleen `"test": "npm run check"` aan `webapp-ui/package.json` toegevoegd | Alle bestaande scripts en `package-lock.json` behouden. |
| Frontend-CI | `webapp-ui/.github/workflows/ci.yml` toegevoegd | Push naar `main` en pull requests; Node 20; `npm ci`; niet-blokkerende `npm run check`; blokkerende productiebuild. |
| Frontend-baseline | `webapp-ui/CHECK_ISSUES.md` toegevoegd | Alle 64 errors en 76 warnings in 34 bestanden zijn volledig, mutually exclusive gecategoriseerd en per bestand geteld. Geen applicatiebroncode geforceerd aangepast. |
| Migration manifest | `MAP-IT_SA/migration/MIGRATION_LEDGER.md` toegevoegd | Huidige bestanden, SHA-256-checksums, statussen, dependencies, snapshotrelatie, vectortransitie en veilige fresh-databasebeslisroute gedocumenteerd. Geen migration aangemaakt of gewijzigd. |

## Backendverificatie

- Diagnose vóór de fix: `pytest --collect-only` vond 3.575 tests, maar eindigde met `PermissionError` op `neo4j/import/conftest.py` en exitcode 2.
- Na de fix: `pytest --collect-only -q` verzamelt 3.575 tests met exitcode 0; centraal herhaald op 20 juli 2026.
- `ruff check app tests`: geslaagd.
- `tests/validators/test_failure_detector.py`: 53 tests geslaagd.
- De volledige `make test`-run kwam voorbij collectie, maar is na 60 seconden beëindigd. Met `pytest -q -x` was de eerste runtimefailure `tests/quality/test_phase1_scenarios.py::test_programme_discovery`: geen backend op `localhost:8000`, geen Google API-key en daardoor score 45 onder de drempel 80. Een door deze test tijdelijk gewijzigd resultaatbestand is exact hersteld.
- Een afzonderlijke historische workflowtest verwacht nog het niet-bestaande `.github/workflows/conversation-tests.yml`: 53 tests slaagden, 1 faalde en 18 werden geskipt in de gecombineerde gerichte run. Die test is niet aangepast, omdat Wave 2 expliciet `ci.yml` vraagt en het bedoelde conversation-workflowcontract niet veilig kan worden afgeleid.
- `black 24.8.0 --check app tests`: verwachte baselinefailure; 135 bestaande bestanden zouden worden geformatteerd en 237 blijven ongewijzigd. Er is geen brede formatrewrite uitgevoerd. De nieuwe CI-formatgate zal dus rood blijven totdat dit als aparte, reviewbare formattingwave wordt opgelost.
- Letterlijk lokaal `make lint` wordt daarnaast beïnvloed door reeds aanwezige untracked Python-scripts/tests en de ontoegankelijke Neo4j-volumemap. De gevraagde workflow gebruikt wel letterlijk `make lint`; in een schone checkout bestaan lokale volume- en niet-gecommitteerde bestanden niet, tenzij ze later bewust worden toegevoegd.
- Beide workflowbestanden zijn als YAML gevalideerd; `package.json` is als JSON gevalideerd.

## Frontendverificatie

- `npm ci`: geslaagd; `package-lock.json` bleef ongewijzigd.
- `npm test`: loopt nu `svelte-check` en eindigt met de gedocumenteerde baseline van 64 errors en 76 warnings in 34 bestanden (exitcode 1).
- `npm run build`: geslaagd met bestaande waarschuwingen, waaronder accessibility/CSS/Svelte-migratiemeldingen en een bundle groter dan 500 kB.
- `npm ci` rapporteerde 35 bestaande auditbevindingen: 3 low, 16 moderate, 15 high en 1 critical. Er is geen automatische dependency-upgrade uitgevoerd vanwege het regressierisico.
- Er zijn in deze wave nul frontendbroncodefixes uitgevoerd; de toegestane limiet van maximaal vijf is dus niet overschreden.

## Migratiebevindingen

- De huidige SQL-map is geen lineaire, reproduceerbare keten. Exacte applied-statussen zijn niet aantoonbaar en staan daarom op `unknown`.
- Git-historie bewijst dat 003–007 bestonden en op 13 mei 2026 in commit `bb90f94` zijn verwijderd. Er waren twee verschillende migraties met nummer 005; ook latere 010–022-bestanden zijn in die cleanup verwijderd.
- `schema.sql` is een brede Supabase/PostgreSQL-snapshot, geen incrementele opvolger van `001`. Het bestand bevat bovendien vier kale `pg_dump:`-diagnoseregels en is niet blind als verse bootstrap uitvoerbaar.
- Het huidige, reeds vóór Wave 2 aanwezige untracked `002_upgrade_embeddings_to_3072.sql` regelt de destructieve overgang voor drie embeddingkolommen van 768 naar 3072 dimensies, maar past niet direct op `001` omdat vereiste modelkolommen daar ontbreken.
- De zes in de ledger opgenomen SHA-256-checksums zijn opnieuw berekend en komen overeen. Geen SQL-bestand is gewijzigd.

## Bewust open gebleven

1. Backend-CI zal op de huidige Black-baseline falen; automatisch 135 bestanden herformatteren valt buiten deze voorzichtige overdrachtswave.
2. De volledige backendtestsuite vereist tiering en reproduceerbare fixtures/services voordat tests blokkerend kunnen worden.
3. De frontendcheck blijft niet-blokkerend totdat de 64 gecategoriseerde errors gefaseerd zijn opgelost; de build is wel groen.
4. De dependency-auditbevindingen vereisen een aparte upgrade- en regressietestronde.
5. Er is nog geen uitvoerbare authoritative fresh-databaseketen. Eerst moet één schema-authority worden gekozen en in CI op een lege compatibele Supabase-instance worden getest.

## Worktreehygiëne

Er is niet gecommit. Reeds vóór Wave 2 aanwezige wijzigingen, dumps, scripts en losse testbestanden in beide repositories zijn behouden en niet inhoudelijk aangepast. Deze Wave 2-wijzigingen moeten bij de uiteindelijke commit expliciet van die bestaande worktree-inhoud worden onderscheiden.
