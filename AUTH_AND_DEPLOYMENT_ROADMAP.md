# Echelon: Auth & Deployment Roadmap

Open work to get Echelon onto the App Store. For in-flight backend changes (Postgres, all-majors tracks,
rerank off by default), see "In progress / planned" in [AGENTS.md](AGENTS.md).

## Current auth

The iOS app signs in with Firebase Auth (email/password, phone, and Google via `OAuthProvider`) against the
`echelon-prod` project and sends the ID token as `Authorization: Bearer <token>`. The backend verifies it with the
Firebase Admin SDK (`GET /api/auth/me` checks a token end to end).

## 1. Identity

- [ ] **Sign in with Apple.** App Store Guideline 4.8 requires it because the app offers Google sign-in.
  Add the capability in Xcode and an Apple button to `LoginView` and `SignUpView`.
- [ ] **Phone auth test numbers.** Add test numbers in Firebase console → Authentication → Phone, so QA, the
  Simulator (which falls back to reCAPTCHA) and App Review don't use real SMS. Spark-plan projects get about
  10 SMS a day; move to Blaze before launch if phone sign-in stays.

## 2. Backend deployment

- [ ] Containerize the FastAPI backend and deploy it behind HTTPS on a custom domain.
- [ ] Keep `GOOGLE_API_KEY`, the Firebase service-account credentials and database credentials in the host's
  secret manager, never in the image or repo.
- [ ] Make the iOS `baseURL` (now a hardcoded LAN IP in `APIService.swift`) come from the build configuration,
  with separate debug and release values.
- [ ] Add rate limiting and audit logging to `/api/opportunities/recommendations` and `/api/profile`
  (see `docs/BACKEND_HANDOFF_TODOS.md`).

## 3. App Store release

- [ ] Host a privacy policy and support URL covering accounts, phone numbers and resume analysis.
- [ ] Add `PrivacyInfo.xcprivacy` declaring collected data (email, phone number, user ID, resume contents).
- [ ] TestFlight: archive, upload to App Store Connect, and distribute to campus beta testers.
- [ ] App Review: create a dedicated review account in Firebase and give its credentials only in App Review notes,
  not in the repo.
