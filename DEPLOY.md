# Deploying Scanline — step by step

You already have: Supabase, Adzuna, GitHub, Vercel, Render. Here's exactly
what to do with each, in order.

## 1. Get your database running (Supabase)
1. Go to your Supabase project → **Project Settings → Database**
2. Copy the **Connection string** (URI format). It looks like:
   `postgresql://postgres:[YOUR-PASSWORD]@[HOST]:5432/postgres`
3. Go to the **SQL Editor** in Supabase, paste in the full contents of
   `db/schema.sql` from this project, and run it. This creates every table.
4. Supabase already has `pgvector` available — no extra setup needed for
   the `CREATE EXTENSION IF NOT EXISTS vector;` line at the top of the schema.

## 2. Get your Adzuna keys
1. Log into your Adzuna developer account → your app should already have
   an **App ID** and **App Key** listed. Copy both.

## 3. Get your Anthropic API key
1. console.anthropic.com → **API Keys** → create one if you haven't. Copy it.

## 4. Push this code to GitHub
1. Create a new repository on GitHub (e.g. `scanline-backend`)
2. Easiest no-command-line way: on the new repo's page, click
   **"uploading an existing file"** and drag in every file/folder from
   this project, keeping the folder structure intact
   (`app/`, `db/`, `requirements.txt`, `render.yaml`, etc.)
3. Commit the upload

## 5. Deploy on Render
1. Render dashboard → **New → Web Service**
2. Connect the GitHub repo you just created
3. Render should auto-detect `render.yaml` and prefill the build/start
   commands. If it doesn't, set them manually:
   - Build command: `pip install -r requirements.txt`
   - Start command: `uvicorn app.main:app --host 0.0.0.0 --port $PORT`
4. Under **Environment**, add these variables (values from steps 1-3):
   - `DATABASE_URL`
   - `ANTHROPIC_API_KEY`
   - `ADZUNA_APP_ID`
   - `ADZUNA_APP_KEY`
   - `SCAN_INTERVAL_MINUTES` = `1440` (once a day)
5. Click **Deploy**. Watch the logs — it should end with something like
   `Uvicorn running on http://0.0.0.0:10000`

## 6. Test it's actually alive
Once deployed, Render gives you a URL like `https://scanline-backend.onrender.com`.
Visit `https://scanline-backend.onrender.com/health` in your browser —
if you see `{"status":"ok"}`, the server is genuinely running.

## 7. Create your first test user
Using the interactive docs Render/FastAPI gives you for free at
`https://scanline-backend.onrender.com/docs`, try:
1. `POST /users` with `{"email": "you@example.com"}` → copy the returned `user_id`
2. `POST /profile` with that `user_id` and your survey answers
3. `POST /listings/scan/{user_id}` — this triggers a real scan right now
   (pulls from Adzuna, tags via Claude, scores against your profile)
4. `GET /listings/matches/{user_id}` — see your real, live-scored matches

## 8. Point the frontend at it (Vercel)
1. Take the `scanline.html` demo file
2. Replace the mock `LISTINGS` array and matching logic with real `fetch()`
   calls to your new Render URL (e.g. `fetch('https://scanline-backend.onrender.com/listings/matches/' + userId)`)
3. Push that updated file to a new GitHub repo, connect it to Vercel,
   deploy — Vercel gives you a public URL for the actual frontend

## What "daily scanning" actually means once this is live
The `SCAN_INTERVAL_MINUTES=1440` setting means the background scheduler
inside your Render service fires once every 24 hours automatically —
no manual trigger needed. It'll keep running as long as the Render
service is up. **Render's free tier spins services down when idle**,
which would silently break the daily scan — for the scheduler to
actually run reliably every day unattended, you need Render's paid
tier (starts around $7/month) or an external cron service (e.g.
cron-job.org) hitting a `/trigger-scan` endpoint on a schedule instead.

## Known gaps to expect on first run
- **Adzuna's `type` field** always comes back as `"job"` — the ingestion
  code doesn't yet distinguish internships from full-time roles. You'll
  want to refine `normalize_adzuna()` in `ingestion.py` to detect
  internship keywords in the title before this is fully accurate.
- **No college/fellowship data source is wired in yet** — Adzuna only
  covers jobs. You'd need to add a second ingestion function for
  admissions/fellowship data, following the same pattern as `fetch_adzuna`.
- **Chat memory uses recency, not true semantic search** — see the
  docstring in `chat_memory.py` for why, and how to upgrade it later.
- **This has not been run end-to-end against a live database** — the
  code is syntax-valid and follows the schema correctly, but the first
  real run may surface a bug or two. Treat this as "ready to test," not
  "guaranteed to work perfectly on the first deploy."
