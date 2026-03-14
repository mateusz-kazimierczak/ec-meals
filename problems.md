# Known Problems & Inconsistencies

Issues identified across the EC Meals system. Ordered roughly by severity.

---

## 1. ~~Duplicate Daily Update System~~

**Resolved.** The `ec-meals-backend/dags/` directory has been removed. The canonical daily update is the Vercel Cron job at `/api/internal/dailyUpdate`.

---

## 2. Vercel Cron Fires 4–5 Hours Early

**Severity: High — partially resolved**

`vercel.json`:
```json
{ "schedule": "30 8 * * *" }
```

Vercel cron schedules run in **UTC**. Toronto is UTC-5 (EST) in winter and UTC-4 (EDT) in summer, so this fires at **3:30 AM Toronto time** in winter and **4:30 AM** in summer — not 8:30 AM as `UPDATE_TIME=0830` and all comments imply.

**Resolved (DAG side):** The new `daily_meals_update` Airflow DAG uses `MultipleCronTriggerTimetable("30 8 * * *", timezone=pendulum.timezone("America/Toronto"))`, which correctly fires at 8:30 AM Toronto time regardless of DST.

**Still open (Vercel side):** `vercel.json` still has `"30 8 * * *"` in UTC. Fix by changing it to `"30 13 * * *"` (UTC, equivalent to 8:30 AM EST). Note this will shift by one hour in summer (EDT). Vercel does not support timezone-aware cron natively.

---

## 3. ~~Live Production Secrets Committed to the Repository~~

**Resolved — not an issue.**

`ec-meals-backend/.gitignore` contains `.env*.local`, which covers `.env.local`. The file is not tracked. Firebase credential files in `ec-meals-app/` are also not tracked. No secrets are committed to either repo.

---

## 4. Weak Default Credentials in Production Config

**Severity: Medium**

The committed `.env.local` has:
```
ADMIN_USERNAME=admin
ADMIN_PASSWORD=admin
JWT_SECRET=EC_MEAL_APP_2024
```

A guessable admin password and a short, predictable JWT secret are both exploitable. Any valid JWT signed with the known secret can be used to impersonate any user.

**Resolution needed:** Replace `JWT_SECRET` with a long random string (e.g. `openssl rand -hex 32`). Change the admin password before next deployment.

---

## 5. ~~Hardcoded Machine-Specific Paths in DAGs~~

**Resolved.** `NODE_BIN` is now a key in the `config.dev` / `config.prod` Airflow Variables. The Google Sheets service account is also loaded from `GCP_AUTH` in the same config Variable, replacing the hardcoded file path in `get_service.py`.

---

## 6. ~~Fragile Inter-Task File Handoff in Notifications DAG~~

**Resolved.** `NOTIFICATIONS_FILE` is now defined as `Path(__file__).parent / "notifications.json"` — a stable absolute path derived from the DAG file's own location. Both the write task and the Node.js subprocess tasks use this same constant. The BashOperators were also replaced with `@task` + `subprocess.run` so env vars are injected at runtime, not at parse time.

---

## 7. ~~Timezone Naming Inconsistency~~

**Resolved.** All three DAGs now use `America/Toronto` consistently.
