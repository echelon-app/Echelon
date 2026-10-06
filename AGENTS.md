# Project Instructions

Single source of truth for every coding agent working in this repo.

## Product

Echelon is swipe-style opportunity discovery for Virginia Tech students of all majors.
Students sign in, upload a resume and enter interests; the backend parses the profile with Gemini,
ranks opportunities for them, and the iOS app shows them as cards to swipe and save.

## Stack and architecture

- iOS: UIKit + SwiftUI, Firebase Auth. Sends the Firebase ID token as `Authorization: Bearer <token>`.
- Backend: Python 3.13, FastAPI, Pydantic, uv, Gemini (`google-genai`), Firebase Admin SDK, Databricks (being replaced by Postgres).

```
iOS (UIKit + SwiftUI, Firebase Auth) --Bearer ID token--> FastAPI --> Gemini
                                                                  --> Databricks SQL warehouse (being replaced)
```

The frontend must never access Gemini or the database directly.

## Rules

- All secrets come from environment variables.
- Never commit `.env`.
- Never fabricate opportunity data, application links, recruiter names, or contact information.
- API responses must use Pydantic models.
- External provider logic belongs in `backend/app/services/`.
- Route files should remain thin. Business logic and provider-specific logic must not live in route handlers.
- Cross-service orchestration belongs in a dedicated application/service layer, not in provider services.
- Write tests for non-trivial logic.
- Do not edit unrelated files.
- Read `docs/API_CONTRACTS.md` before changing response formats. It is the iOS ↔ backend contract.
- Never rename or change an existing environment variable without coordinating the change.
- Never modify another integration's service files unless the current task explicitly requires it.
- Before modifying shared configuration, API contracts, dependencies, models, or app initialization,
  inspect existing implementations and avoid breaking other integrations.

### Public repo

This repo is public; anything pushed is exposed, even if later removed.
- Never commit secrets, `.env`, real user data, resumes, or service-account JSON.
- `ios/EchelonTestRun/Echelon/GoogleService-Info.plist` may only hold the bundle-restricted iOS key for `echelon-prod`.
- Each developer gets their own keys from the `echelon-prod` Google Cloud / Firebase project (ID `echelon-prod-706c0`).
  Keys are never shared in chat or committed.

### Persistence (Databricks → Postgres)

- No new Databricks code.
- New persistence goes behind the same module-level functions in the persistence service
  (`save_student_profile`, `get_active_opportunities`, `save_swipe`, ...) so callers don't change.
- No Databricks-specific SQL (`MERGE`, `USING DELTA`, Unity Catalog names) in new code.
- `scripts/seed_opportunities.py` and `scripts/migrate_databricks_schema.py` are legacy.

## Commands

All backend commands run from `backend/` (where `pyproject.toml` and `uv.lock` live).

```bash
uv sync                                                   # install deps
uv run uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
uv run pytest                                             # all tests (testpaths=tests, pythonpath=.)
uv run pytest tests/test_swipes.py::test_name -v          # one test
uv run python scripts/run_ingestion.py --max-items 25     # scrape -> filter -> Gemini classify -> upsert (live providers)
```

No linter or formatter is configured. The iOS app opens in Xcode via `ios/EchelonTestRun/Echelon.xcodeproj` (scheme `EchelonTestRun`).

## Backend layering (`backend/app/`)

- `api/`: thin routers registered in `main.py`. User-facing routes depend on `api/deps.py:get_current_user`,
  which verifies the Firebase token; the user's ID is `current_user["uid"]` (Firebase UID = student ID everywhere).
  `/health`, `/test/gemini` and `/test/databricks` are unauthenticated.
- `services/`: module-level functions, not classes (except `FirebaseService`). Callers do
  `from app.services import databricks_service` and call `databricks_service.get_student_profile(uid)`.
  - Provider services `gemini_service.py`, `databricks_service.py`, `firebase.py` must not import or instantiate each other.
  - Orchestrators combine providers: `recommendation_service.py`, `echelon_agent_service.py`,
    `ingestion_service.py`, `candidate_retrieval.py`. `eligibility_engine.py` is pure logic.
  - Firebase auth must not contain recommendation logic.
- `models/`: Pydantic domain models (`StudentProfile`, `Opportunity`, `CareerPreferences`, `Swipe`, recommendation types). `schemas/`: request/response wrappers.
- `ingestion/`: source adapters (`BaseSourceAdapter.fetch_opportunities()`; `virginia_tech.py`, `simplify_jobs.py`, `web_scraper.py`)
  and quality filters (`filters.py`, `strict_tech_only=True` by default), driven by `ingestion_service.run_ingestion_pipeline`.
- `core/config.py` is the single `Settings` source; `app/config.py` re-exports it. Settings load the **repo-root** `.env`.
- Unused stubs: top-level `/ingestion`, `/src`, and `backend/src`.

## Recommendation pipeline (`GET /api/opportunities/recommendations`)

`recommendation_service.get_recommendations`:
1. Load `StudentProfile` and `CareerPreferences`. Missing profile → `StudentProfileNotFoundError` (404); missing preferences → defaults.
2. `candidate_retrieval.get_top_candidates(limit=30)`: active opportunities, minus expired deadlines and `INELIGIBLE`
   ones, scored on career-track affinity, skill overlap and interest overlap.
3. `gemini_service.rerank_opportunities` reranks and writes explanations (always called today).
4. Re-attach eligibility status and notes, truncate to `limit`.

Agent chat (`POST /api/agent/chat`) loads context, calls `gemini_service.chat_agent`, and persists any
`updated_preferences`; those preferences feed track weighting in retrieval.

## Current persistence: Databricks (being replaced)

All SQL goes through `databricks_service._execute_statement` against a SQL warehouse, authenticated via a
`~/.databrickscfg` profile. Tables (`student_profiles`, `opportunities`, `swipes`, `saved_opportunities`,
`career_preferences`) are Delta tables in `DATABRICKS_CATALOG`.`DATABRICKS_SCHEMA`; opportunities upsert with `MERGE`.
`_row_to_opportunity` maps rows by column name.

## Environment variables

Read by `core/config.py` from the process environment or the repo-root `.env`. `.env.example` mirrors this table. Never commit values.

| Variable | Required? | Purpose | Where to get it |
|---|---|---|---|
| `GOOGLE_API_KEY` | For Gemini features | Gemini API key used by `gemini_service` | `echelon-prod` Cloud console → APIs & Services → Credentials (Gemini-restricted key) |
| `FIREBASE_CREDENTIALS_PATH` | For authenticated routes | Path to a service-account JSON **outside the repo**; unset → Application Default Credentials | Firebase console → Project settings → Service accounts → Generate new private key (`firebase-adminsdk-fbsvc@echelon-prod-706c0...`) |
| `DATABRICKS_CONFIG_PROFILE` | Legacy, for persistence | Profile name in `~/.databrickscfg` | Databricks workspace (not in `echelon-prod`) |
| `DATABRICKS_WAREHOUSE_ID` | Legacy, for persistence | SQL warehouse ID | Databricks workspace |
| `DATABRICKS_CATALOG` | Legacy, for persistence | Unity Catalog name | Databricks workspace |
| `DATABRICKS_SCHEMA` | Legacy, for persistence | Schema name | Databricks workspace |
| `GEMINI_API_KEY` | Optional | Fallback when `GOOGLE_API_KEY` is unset; read from the **shell env only** (see Gotchas) | Same key as `GOOGLE_API_KEY` |
| `APP_NAME`, `DEBUG` | Optional | FastAPI title and debug flag (defaults `Echelon Backend`, `false`) | n/a |
| `GEMINI_GENERATIVE_MODEL`, `GEMINI_EMBEDDING_MODEL`, `GEMINI_EMBEDDING_DIMENSION` | Optional, unused | Read into Settings but code hardcodes `gemini-3.8-flash` | n/a |
| `DATABRICKS_HOST`, `DATABRICKS_AI_SEARCH_ENDPOINT`, `DATABRICKS_AI_SEARCH_INDEX` | Optional, unused | Read into Settings, never used (host comes from the profile) | n/a |

## Testing conventions

- Tests never call live providers. Patch at the module path the code looks up, e.g.
  `mocker.patch("app.services.databricks_service.get_opportunity", ...)` (pytest-mock).
- Bypass auth with `app.dependency_overrides[get_current_user] = lambda: {"uid": "..."}`; clear it afterward.
- `conftest.py` provides a `client` fixture (`TestClient(app)`).

## Gotchas

- **Gemini key**: use `GOOGLE_API_KEY`. `gemini_service._get_client` falls back to `os.environ["GEMINI_API_KEY"]`,
  which `.env` does not populate for the server (pydantic-settings doesn't export to `os.environ`). Don't rename either.
- **`.env` leaks into tests**: `tests/test_config.py` expects `GEMINI_API_KEY` and `DATABRICKS_HOST` unset, so leave them out of `.env`.
- **Process-env-only vars**: `GOOGLE_APPLICATION_CREDENTIALS` / `GOOGLE_CLOUD_PROJECT` (used by Firebase ADC) are not read from `.env`; prefer `FIREBASE_CREDENTIALS_PATH`.
- **iOS `baseURL`** is a hardcoded LAN IP in `ios/.../Services/APIService.swift`.
- **iOS calls endpoints the backend lacks**: `POST /api/opportunities/{id}/apply`, `POST /api/chat` (fallback),
  `DELETE /api/profile`, `DELETE /api/students/{id}`.
- iOS mixes UIKit (`Controllers/`, `MainFloatingTabBarController` is the tab shell) and SwiftUI (`Views/`).
  Firebase is configured in `EchelonApp.swift` (`FirebaseApp.configure()` from `GoogleService-Info.plist`).
- `GoogleService-Info.plist` is often locally modified; don't commit changes to it casually.
- Trust code and `docs/API_CONTRACTS.md` over other docs; `docs/BACKEND_HANDOFF_TODOS.md` lists work iOS already expects.

## In progress / planned

- Databricks → a new Postgres database on Openship Cloud, starting empty and filled by re-running ingestion.
- The repo is moving to the public `echelon-app` GitHub org.
- The sample cards (`OpportunityCard.mockDeck` in `DataModels.swift`, the `MatchStore` fallback deck) and the demo login
  (`signInDemo` in `AuthService.swift`) are being removed. Don't build on them.
- Career tracks move from 14 tech-only tracks (`gemini_service.classify_opportunity`) to an all-majors taxonomy.
- Gemini rerank is going off by default. Feeds must work with zero Gemini calls.
- `POST /api/opportunities/{id}/apply` is called by iOS but not implemented yet.
