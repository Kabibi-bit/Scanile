# Scanline Backend (scaffold)

This is a starting scaffold matching the architecture in `SPEC.md` — it
runs and defines the right shape, but several functions are stubbed with
TODOs where they need a real database session wired in. Treat it as the
skeleton to build on, not a finished product.

## What's real vs. stubbed

**Real, working logic:**
- `app/services/matching.py` — full scoring/ranking engine, ported from the demo
- `app/services/ingestion.py` — real Adzuna API integration + tag extraction via Claude
- `app/routes/chat.py` — working Claude chat endpoint
- `db/schema.sql` — complete Postgres schema, ready to run

**Stubbed (need a DB session wired in — marked with TODO):**
- `app/routes/profile.py` — profile CRUD
- `app/routes/listings.py` — pulling stored listings + matches
- `app/services/chat_memory.py` — persistent memory storage/retrieval
- `app/services/scheduler.py` — the actual DB queries inside the scan job

## Setup

1. `pip install -r requirements.txt`
2. Create a Postgres database, then run `db/schema.sql` against it
   (needs the `pgvector` extension available on your Postgres instance)
3. Copy `.env.example` to `.env` and fill in real values:
   - `DATABASE_URL`
   - `ANTHROPIC_API_KEY`
   - `ADZUNA_APP_ID` / `ADZUNA_APP_KEY` (free tier: https://developer.adzuna.com)
4. Wire a SQLAlchemy session into the routes/services marked TODO —
   the standard pattern is a `get_db()` dependency injected into each route
5. Run: `uvicorn app.main:app --reload`

## Next steps, in order (matches SPEC.md build order)

1. Wire the DB session into `profile.py` and `listings.py` — this alone
   makes phases 1-2 functional
2. Point the frontend demo's `fetch` calls at this API instead of its
   mock `LISTINGS` array
3. Add outcome tracking + roadmap generation (new route + LLM call,
   same pattern as `chat.py`)
4. Only after 1-3 are working with real users: persistent chat memory,
   then auto-send with guardrails
