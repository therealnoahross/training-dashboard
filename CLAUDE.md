# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

A personal training dashboard that combines Oura, Garmin, and nutrition data. It has a single user and is for personal use only. Multi-user support, auth, and hosting are out of scope.

## Constraints

- **Data stays local** (from `PRIVACY.md`): all data lives on the owner's machine and is never shared with third parties. Don't add telemetry, analytics, or cloud sync. The only network calls allowed are to the Oura and Garmin APIs.
- Secrets go in `.env`. Oura OAuth tokens go in `tokens.json`. Data goes in `dashboard.db`. All of these are gitignored and must never be committed.

## File structure

```
training-dashboard/
├── CLAUDE.md            # project context for Claude Code
├── requirements.txt     # pinned dependencies
├── .env / .env.example  # secrets / placeholder names
├── tokens.json          # gitignored
├── goals.json
├── oura_auth.py
├── oura_client.py
├── garmin_client.py
├── db.py
├── fetch_daily.py
├── metrics.py
├── status.py
├── app.py
├── pages/
│   └── food_log.py
└── dashboard.db         # gitignored
```

## Architecture

Data flows one way: **APIs → `fetch_daily.py` → `dashboard.db` → `metrics.py` / `status.py` → Streamlit UI.**

| File | Responsibility |
|---|---|
| `oura_auth.py` | One-time OAuth2 flow for Oura. It opens the browser, catches the localhost redirect, and writes access and refresh tokens to `tokens.json`. Run it manually once, and again only if the refresh token stops working. |
| `oura_client.py` | Thin Oura API v2 wrapper (`https://api.ouraring.com/v2/usercollection/...`). It refreshes tokens automatically and saves them back to `tokens.json`. Oura issues a new refresh token on every refresh, so the old one stops working. It returns plain dicts keyed by date and never touches the DB. |
| `garmin_client.py` | Official **Garmin Connect Personal API**, using plain `requests` with no wrapper library. On each run it POSTs `GARMIN_API_KEY` (header `Authorization: ApiKey <key>`) to `https://personal.connectapi.garmin.com/personalapitoken-service/personal-api/personal-api-key/token` and gets a fresh token. Data requests then send `Authorization: Bearer <token>` to the `personal.connectapi.garmin.com` host. Tokens from this endpoint don't work on `connectapi.garmin.com`. Tokens last 24 hours, so nothing is stored or refreshed. On a 401 it prints a clear "Garmin API key rejected or expired — create a new one in Garmin Connect" message. Garmin supplies workouts, and recovery and readiness come from Oura. The API is read-only and covers only some endpoints. It returns plain dicts keyed by date and never touches the DB. |
| `db.py` | The only module that runs SQL. It owns the SQLite schema (created on first connect) and exposes small upsert and query functions. Rows are keyed by local date (`YYYY-MM-DD`), and upserts are idempotent so it's safe to re-fetch the same day. |
| `fetch_daily.py` | CLI entry point and the only code that calls the APIs. It pulls a date range from both clients and upserts into the DB. With no arguments it fetches the last few days to pick up late syncs. `--days N` backfills. Meant to run from cron or launchd. |
| `goals.json` | User-editable targets: sleep, HRV/readiness thresholds, weekly training load, and calories and macros. It's committed and has no secrets. |
| `metrics.py` | Pure functions that turn DB rows into derived metrics: rolling averages, baselines, training load (including the 7-day vs 28-day load ratio), and nutrition totals. No I/O except through `db.py`. |
| `status.py` | Compares `metrics.py` output with `goals.json` and returns a per-area status (on track, watch, off track) with a short reason. Pure logic that the UI renders. |
| `app.py` | Streamlit home page. Read-only view of status and metrics charts. **It never calls the Oura or Garmin APIs.** It reads only from the DB, so it loads fast and works offline. |
| `pages/food_log.py` | Streamlit page for entering and editing nutrition by hand. Nutrition data comes only from here and is written through `db.py`. |

Layering rule: clients don't know about the DB, `db.py` doesn't know about APIs, and the UI doesn't know about either. It talks to `db.py`, `metrics.py`, and `status.py`.

## Commands

```bash
python3 -m venv .venv && source .venv/bin/activate.fish   # owner's shell is fish
pip install -r requirements.txt
cp .env.example .env                 # then fill in values
python oura_auth.py                  # one-time Oura OAuth
python fetch_daily.py                # fetch recent days
python fetch_daily.py --days 90      # backfill
streamlit run app.py                 # run the dashboard
```

## Build plan: one file at a time

Build the files in this order. Each step creates **one** file (step 1 is the exception and sets up the project). After each step:
1. Run that step's check.
2. Tick its box here.
3. **Stop and wait for the owner to review before starting the next file.**

Don't scaffold files ahead of their step or stub out later modules.

- [ ] **1. Setup:** `requirements.txt` (`streamlit`, `pandas`, `requests`, `python-dotenv`), `.env.example` (`OURA_CLIENT_ID`, `OURA_CLIENT_SECRET`, `OURA_REDIRECT_URI`, `GARMIN_API_KEY`), and add `tokens.json` to `.gitignore`. The owner creates the Garmin key in Garmin Connect web under Personal API Key. It's shown only once, so paste it straight into `.env`. Choose the 1-year expiry. *Check:* `pip install -r requirements.txt` succeeds, and `git status` doesn't show `.env` or `tokens.json`.
- [ ] **2. `db.py`:** schema plus upsert and query helpers for daily Oura, daily Garmin, activities, and food entries. *Check:* importing it and calling the init function creates `dashboard.db` with the expected tables.
- [ ] **3. `oura_auth.py`:** *Check:* running it writes `tokens.json` with access and refresh tokens.
- [ ] **4. `oura_client.py`:** sleep, readiness, activity, and HRV by date range, with automatic refresh. *Check:* one call for yesterday returns data.
- [ ] **5. `garmin_client.py`:** first, read the Personal API endpoint reference (Swagger docs on the Garmin developer portal). Find the activities endpoint and record here which fields hold duration, distance, and heart rate or training load. These fields drive the load ratio in `metrics.py`. Then write the client for activities and daily summaries by date range. *Check:* one call for yesterday returns data, and a bad key prints the "key rejected" message instead of a stack trace.
- [ ] **6. `fetch_daily.py`:** *Check:* `--days 7` fills the DB, and running it again creates no duplicates.
- [ ] **7. `goals.json`:** owner fills in real targets. *Check:* valid JSON with every key `status.py` will need.
- [ ] **8. `metrics.py`:** *Check:* the functions return sensible values against the real DB.
- [ ] **9. `status.py`:** *Check:* prints a status for each area for today.
- [ ] **10. `app.py`:** *Check:* `streamlit run app.py` shows today's status and trend charts.
- [ ] **11. `pages/food_log.py`:** *Check:* an entry saves, appears again after reload, and shows up in nutrition metrics and status.
