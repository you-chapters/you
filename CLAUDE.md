# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# YOU — meta-repo

Documentation and planning for the YOU personal memory engine. Code lives in sibling repos:

- `../you-api` — FastAPI + AWS Lambda + DynamoDB (see its CLAUDE.md for commands and architecture)
- `../you-ui` — React SPA (see its CLAUDE.md for commands and architecture)

This repo contains only `README.md`, `CLAUDE.md`, and per-project implementation docs under `../you-api/docs/` and `../you-ui/docs/`.

---

## Current status

**Phase 1 complete.** Entries are created, stored in DynamoDB, embedded via OpenAI, and upserted into Pinecone. Tags (people, locations, topics, mood, time_markers) are extracted asynchronously and written back to DynamoDB via a DynamoDB Streams Lambda. Semantic search via `/entries/search` is live.

**Phase 2 substantially complete.**
- `GET /entries/summary` — 7/30-day aggregation of topics, people, mood. Lives on the dashboard as `PeriodStrip` + `PeopleCard`.
- `GET /entries/narrative` — lazy on-demand LLM prose (gpt-4o-mini), cached in the `narratives` DynamoDB table. Dashboard shows `NarrativeStack` (two-deck week + month navigation) for current and previous periods.
- `PlacesCard` is live in the UI but the API does not yet return `top_locations` in `PeriodSummary` — places will always render empty until the backend is extended (see Known gaps).
- Stage 2 (EventBridge cron to pre-generate narratives for all users via `handler_narrative.py`) is **not started**.
- "On this day" is **not started**.

Implementation specs:
- `../you-api/docs/phase2-dashboard.md`
- `../you-api/docs/narrative-recap.md`
- `../you-ui/docs/phase2-dashboard.md`
- `../you-ui/docs/narrative-recap.md`

---

## Cross-project architecture

The two services are decoupled: the UI is a static SPA hosted on S3/CloudFront; the API is AWS Lambda behind API Gateway. They share nothing at runtime except the API contract and Cognito for auth.

**Entry lifecycle** (spans both projects):

```
UI: user submits form
  → POST /entries (API Gateway → YouApiFunction Lambda)
    → Entry saved to DynamoDB with tags: null
    → 201 returned to UI immediately

DynamoDB Stream (INSERT event)
  → YouEmbeddingFunction Lambda
    → OpenAI: extract tags  (gpt-4o-mini)
    → OpenAI: embed augmented text  (text-embedding-3-small)
    → Pinecone: upsert vector + metadata
    → DynamoDB: SET tags = :tags
```

The write path is non-blocking by design. The UI receives an entry with `tags: null` on creation; tags appear only after the async pipeline completes.

**Narrative lifecycle:**

```
UI: DashboardPage mounts
  → GET /entries/narrative?type=week&key=2026-W21
    → NarrativeService checks DynamoDB narratives table
      → cache hit (not stale): return cached text
      → cache miss or stale: call gpt-4o-mini → cache → return
```

Staleness rule: current period refreshes if cached record is >24 h old; past periods are frozen on first generation.

**Authentication:** Cognito issues JWTs. API Gateway validates them. The `sub` claim is the `user_id` — it is extracted server-side and never accepted from request bodies. The UI attaches the Cognito ID token as `Authorization: Bearer` on every request.

**Secrets:** stored in AWS SSM Parameter Store (`/you-api/*`). At runtime, `app/config.py` detects SSM paths by a leading `/` and fetches them. Locally, set env vars to raw values.

**Pinecone metadata per vector:** `user_id`, `timestamp` (unix int), `topics`, `mood`, `people`, `locations`. All searches filter by `user_id` at the Pinecone level.

---

## API surface

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/entries` | Create entry |
| `GET` | `/entries` | List all entries (newest-first) |
| `GET` | `/entries/{entry_id}` | Fetch single entry |
| `POST` | `/entries/search` | Semantic search via Pinecone |
| `GET` | `/entries/summary` | 7/30-day signal aggregation |
| `GET` | `/entries/narrative` | LLM prose recap (`?type=week\|month&key=…&refresh=true`) |

Route ordering matters: `/summary` and `/narrative` must be declared before `/{entry_id}` in the router.

---

## Key design decisions

- **Tags are async-only.** The HTTP API never writes tags. This keeps the create endpoint fast and the tag-extraction pipeline independently deployable.
- **DynamoDB streams only on the prod table.** `test_entries` has no stream; the embedding Lambda never fires in tests.
- **`entry_id` is a UUID sort key, not time-ordered.** Time-range queries filter `timestamp` (ISO-8601 string) in Python after fetching all user entries. A GSI on `timestamp` is not yet needed at current scale.
- **topics are a controlled vocab:** `work, family, travel, health, reading, finance, relationships, hobbies, food, exercise`. Enforced in the OpenAI system prompt.
- **Narrative Stage 1 is lazy.** The API generates on first request per period and caches in DynamoDB. Stage 2 (EventBridge pre-generation for all users) is not yet implemented.
- **NarrativeStack goes beyond the spec.** The UI implements two-deck navigation (current + previous period) with fade transitions rather than the single-card design in the doc.

---

## Known gaps

- **`top_locations` missing from API.** `PlacesCard` in the UI expects `top_locations: LocationCount[]` in the `PeriodSummary` response, but the API's `PeriodSummary` model does not include it. `EntryService.get_summary()` needs a location aggregation pass and a `LocationCount` model. Until then, `PlacesCard` always receives an empty list and renders nothing.
- **Narrative Stage 2 not started.** `handler_narrative.py` Lambda and EventBridge rules (weekly Sunday 23:55 UTC, monthly 1st 00:05 UTC) are documented in `../you-api/docs/narrative-recap.md` but not implemented.
- **`embedding_port.py` dead code.** `app/embedding/embedding_port.py` in you-api is a leftover from a refactor and is unused.
