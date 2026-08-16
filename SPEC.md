# Scanline — Project Specification

## 1. Vision
A continuously-running opportunity watch that scans real job/internship/fellowship
listings, scores them against a user's stated long-term goal (not just keywords),
explains its reasoning, tracks outcomes, builds a career roadmap, and gives the
user a chatbot with persistent memory of their goals and history. Optionally,
high-confidence matches can be auto-applied to with guardrails.

## 2. System architecture

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Frontend    │────▶│   API layer   │────▶│   Postgres DB    │
│  (React)     │◀────│  (FastAPI)    │◀────│  + pgvector      │
└─────────────┘     └──────┬───────┘     └─────────────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
      ┌──────────────┐ ┌──────────┐ ┌──────────────┐
      │ Job queue     │ │ LLM API  │ │ Job data APIs │
      │ (scan worker) │ │ (Claude) │ │ (Adzuna, etc) │
      └──────────────┘ └──────────┘ └──────────────┘
```

## 3. Core modules & functions

### 3.1 Data ingestion
- `fetch_listings(source, query)` — pull from external job APIs
- `normalize_listing(raw)` — map to canonical schema
- `dedupe_listings(listings)`
- `extract_tags(description)` — LLM or NLP-based tag extraction
- `refresh_scheduler()` — background job, runs on interval

### 3.2 Profile & goals
- `create_profile(user_id, survey)`
- `update_profile(user_id, deltas)`
- `get_profile_history(user_id)`

### 3.3 Matching engine
- `score_listing(listing, profile)`
- `explain_score(listing, profile)`
- `rank_compounding_value(listing, profile, roadmap)`
- `filter_dealbreakers(listings, profile)`

### 3.4 Roadmap engine
- `generate_roadmap(profile)` — LLM call
- `update_roadmap_progress(user_id, event)`
- `get_milestones(user_id)`

### 3.5 Outcome tracking
- `log_outcome(user_id, listing_id, status)`
- `get_outcome_stats(user_id)`
- `retrain_weights(user_id)` — heuristic reweighting, not full ML retrain for v1

### 3.6 Chatbot & memory
- `build_context(user_id)` — assembles profile + roadmap + outcomes + matches
- `store_conversation_summary(user_id, conversation)`
- `retrieve_relevant_memory(user_id, query)` — lightweight RAG
- `chat_respond(user_id, message)`

### 3.7 Application drafting & auto-send (build last)
- `draft_application(listing, profile)`
- `get_confidence_threshold(listing, profile)`
- `queue_for_review(user_id, draft)`
- `send_application(user_id, listing_id)`
- `undo_window(user_id, application_id)`

### 3.8 Notifications
- `notify_new_match(user_id, listing)`
- `notify_deadline_urgent(user_id, listing)`

## 4. Data model
See `db/schema.sql` for full DDL. Core tables: `users`, `profiles`,
`listings`, `match_scores`, `outcomes`, `roadmap_milestones`,
`chat_memory`, `applications`.

## 5. Build order

| Phase | Scope | Notes |
|---|---|---|
| 1 | Data ingestion + normalized schema + basic scoring | No AI needed yet |
| 2 | Profile storage in real DB | Move off browser state |
| 3 | Chatbot, session-only memory | Uses existing profile + matches |
| 4 | Outcome tracking + roadmap generation | First LLM-driven features |
| 5 | Persistent chatbot memory | Needs real outcome data to ground it |
| 6 | Auto-send with guardrails | Build last; legal review first |

## 6. Guardrails for auto-send (non-negotiable)
- Nothing sends without a defined confidence threshold
- Everything below threshold goes to a human-review queue
- Every sent application is logged with an undo window before final submission
- No auto-send without a legal review of target site ToS

## 7. Open questions to resolve before Phase 6
- Which job sources can actually be scraped/used under their ToS?
- What confidence score justifies auto-send vs. human review?
- How long is memory retained, and how is it summarized/pruned?
