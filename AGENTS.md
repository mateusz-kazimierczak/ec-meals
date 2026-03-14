# EC Meals — CLAUDE.md (Root)

## System Overview
EC Meals is a meal management system for Ernescliff. It consists of three subprojects that work together:

| Subproject | Purpose | Stack |
|---|---|---|
| `ec-meals-app/` | Mobile + web UI for students and admins | React Native + Expo |
| `ec-meals-backend/` | REST API + daily cron job | Next.js 14 + MongoDB |
| `meal-DAGs/` | Airflow workflows for sheets + notifications | Python + Airflow 3 |

## Data Flow

```
User (mobile/web)
    ↕ JWT API calls
ec-meals-backend (Vercel)
    ↕ Mongoose
MongoDB Atlas
    ↑ Airflow DAGs
meal-DAGs (Airflow, runs on local server)
    → Google Sheets (daily meal summary)
    → Resend email (meal notifications)
    → Expo push notifications (mobile alerts)
    → BigQuery (audit log, written by backend cron)
```

## How the Three Pieces Connect

1. **Backend** is the source of truth. It owns the MongoDB database, authenticates users, and runs the 8:30 AM daily cron job (via Vercel Cron) that advances meal schedules.

2. **App** talks exclusively to the backend REST API. All state is server-side; the app only caches the auth token locally.

3. **DAGs** run independently on a separate Airflow instance. They also read from the same MongoDB database (read-only for notifications; read-only for sheet generation) and send their own notifications independently of the backend.

## Shared Database (MongoDB)
All three subprojects share the same MongoDB Atlas cluster. Collections:
- `users` — User profiles, meal matrices, preferences, push tokens
- `days` — Daily meal records (created by backend cron at 8:30 AM)
- `diets` — Diet type definitions

Date format in MongoDB: `D/M/YYYY` (e.g., `13/3/2026` — no leading zeros)

## Each Subproject Has Its Own Docs
- `ec-meals-app/CLAUDE.md` — Frontend-specific context for Claude
- `ec-meals-backend/CLAUDE.md` — Backend-specific context for Claude
- `meal-DAGs/CLAUDE.md` — DAG-specific context for Claude

## Timing Dependencies
The daily workflow depends on ordering:
1. **8:30 AM** — Backend cron (`/api/internal/dailyUpdate`) creates today's `Day` document in MongoDB
2. **8:35 AM** — `meal_sheet_update_dag` reads that document and updates Google Sheets
3. **7:30 AM / 12 PM / 7:30 PM** — `meal_notifications` DAG reads user schedules and sends reminders

## Deployment Environments

| Subproject | Production URL | Dev Setup |
|---|---|---|
| App | Netlify (web) + App Store (iOS) | `npm start` in `ec-meals-app/` |
| Backend | `https://ec-meals-backend.vercel.app` | `npm run dev` in `ec-meals-backend/` |
| DAGs | Self-hosted Airflow (local server) | `airflow standalone` in `meal-DAGs/` |

## Secrets & Credentials
- Backend secrets: `.env.local` in `ec-meals-backend/`
- App secrets: `.env.test` / `.env.production` in `ec-meals-app/`
- DAGs secrets: Airflow connections + `/home/mateusz/secrets/sheets_sa.json` + `RESEND_API_KEY` env var
- Firebase credentials (`google-services.json`) are committed to the app repo — do not add further secrets to version control

## Repo Structure
This is a monorepo managed as a Git repo with submodules:
```
meals/
├── ec-meals-app/       (submodule)
├── ec-meals-backend/   (submodule)
├── meal-DAGs/          (directory)
├── CLAUDE.md           (this file)
└── README.md
```
