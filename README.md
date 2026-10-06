# Echelon

Swipe-style opportunity discovery for Virginia Tech students of all majors. Originally built at VTHacks 2026.

Students sign in, upload a resume and add interests. Echelon builds a profile from them and shows
internships, research, jobs and campus opportunities as cards to swipe right to save or left to pass.

## App Preview

<p align="center">
  <img src="docs/images/discover.png" width="30%" />
  <img src="docs/images/discover-match.png" width="30%" />
  <img src="docs/images/discover-pass.png" width="30%" />
</p>

<p align="center">
  <img src="docs/images/opportunity-details.png" width="30%" />
  <img src="docs/images/matches.png" width="30%" />
  <img src="docs/images/profile.png" width="30%" />
</p>

## How it works

```
iOS app (UIKit + SwiftUI, Firebase Auth) --Bearer ID token--> FastAPI backend --> Gemini (resume parsing, ranking)
                                                                              --> Database (Databricks, moving to Postgres)
```

- **iOS** (`ios/`): UIKit + SwiftUI app. Users sign in with Firebase Auth, and every API call sends the Firebase ID token.
- **Backend** (`backend/`): Python 3.13 + FastAPI. Verifies the token, parses resumes with Gemini, and ranks opportunities
  from listings collected by an ingestion pipeline.

The app never talks to Gemini or the database directly.

## Run the backend

Needs Python 3.13 and [uv](https://docs.astral.sh/uv/).

```bash
cp .env.example .env        # fill in your own keys; see the env-var table in AGENTS.md
cd backend
uv sync
uv run uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

Check `http://localhost:8000/health` returns `{"status": "ok"}`. Run the tests with `uv run pytest` (no `.env` needed).

## Run the iOS app

Needs a Mac with Xcode.

1. Open `ios/EchelonTestRun/Echelon.xcodeproj` and pick the `EchelonTestRun` scheme.
2. Set `baseURL` in `ios/EchelonTestRun/Echelon/Services/APIService.swift` to your backend:
   `http://localhost:8000` for the Simulator, or your machine's LAN IP for a physical iPhone.
3. Build and run (`Cmd + R`).

## Docs

- [AGENTS.md](AGENTS.md): setup details, environment variables, architecture and project rules (for people and coding agents).
- [docs/API_CONTRACTS.md](docs/API_CONTRACTS.md): the iOS ↔ backend API contract.

Never commit `.env`, keys or service-account files. This repo is public.
