# Architecture — Echelon

Architecture is documented in [AGENTS.md](../AGENTS.md) so there is one source of truth:

- **Stack and architecture**: iOS → FastAPI → Gemini and the database, and the Firebase auth flow.
- **Backend layering**: routers, provider services, orchestrators, models and ingestion.
- **Recommendation pipeline**: how `GET /api/opportunities/recommendations` builds a feed.
- **Current persistence**: the Databricks tables, which are being replaced by Postgres.
- **In progress / planned**: changes underway.

Request and response shapes are in [API_CONTRACTS.md](API_CONTRACTS.md).
