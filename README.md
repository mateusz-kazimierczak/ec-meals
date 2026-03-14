# EC Meals

Meal management system for Ernescliff. Students select their weekly meals; admins manage rosters and dietary requirements; the kitchen gets a daily Google Sheet summary; everyone gets reminders at meal times.

## Projects

| Directory | What It Is |
|---|---|
| [`ec-meals-app/`](./ec-meals-app/README.md) | React Native + Expo mobile/web app |
| [`ec-meals-backend/`](./ec-meals-backend/README.md) | Next.js 14 REST API + daily cron job |
| [`meal-DAGs/`](./meal-DAGs/README.md) | Apache Airflow workflows (sheets + notifications) |

## Architecture

```
┌─────────────────────────────────┐
│        EC Meals App             │
│   (React Native / Web)          │
│   Netlify + App Store           │
└────────────┬────────────────────┘
             │ JWT API calls
┌────────────▼────────────────────┐
│       EC Meals Backend          │
│       (Next.js on Vercel)       │
│   REST API + 8:30 AM cron       │
└────────────┬────────────────────┘
             │ Mongoose
┌────────────▼────────────────────┐
│         MongoDB Atlas           │
│   users / days / diets          │
└────────────┬────────────────────┘
             │ Airflow reads
┌────────────▼────────────────────┐
│          Meal DAGs              │
│     (Airflow, self-hosted)      │
├─────────────────────────────────┤
│  → Google Sheets (8:35 AM)      │
│  → Email via Resend             │
│  → Push via Expo (3x daily)     │
│  → BigQuery audit log           │
└─────────────────────────────────┘
```

## Daily Automated Workflow

| Time (Toronto) | What Happens |
|---|---|
| 8:30 AM | Backend cron advances all user meal schedules and creates today's meal record |
| 8:35 AM | Airflow reads today's record and updates the monthly Google Sheet |
| 7:30 AM | Airflow sends morning meal reminders (today's meals) |
| 12:00 PM | Airflow sends noon reminders (tomorrow's meals) |
| 7:30 PM | Airflow sends evening reminders (next-day packed meals) |

## Quick Start

Each subproject has its own README with setup instructions. In general:

### App (web)
```bash
cd ec-meals-app
npm install
npm run web
```

### Backend
```bash
cd ec-meals-backend
npm install
# Create .env.local (see ec-meals-backend/README.md)
npm run dev
```

### DAGs
```bash
cd meal-DAGs
uv sync
airflow standalone
```

## Shared Database

All three subprojects connect to the same MongoDB Atlas cluster. The backend owns writes; DAGs are read-only. Key collections:

- **`users`** — Profiles, 7-day meal matrices, notification preferences, push tokens
- **`days`** — Daily meal snapshots (created by backend cron)
- **`diets`** — Dietary restriction types

## Repo Structure

This repo uses Git submodules for `ec-meals-app` and `ec-meals-backend`:

```bash
# Clone with submodules
git clone --recurse-submodules <repo-url>

# Update submodules after pulling
git submodule update --init --recursive
```
