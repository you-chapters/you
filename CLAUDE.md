# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# YOU — meta-repo

Documentation and planning for the YOU personal memory engine. Code lives in sibling repos:

- `../you-api` — FastAPI + AWS Lambda + DynamoDB (see its CLAUDE.md for commands and architecture)
- `../you-ui` — React SPA (see its CLAUDE.md for commands and architecture)

This repo contains only `README.md`, `CLAUDE.md`, and per-project implementation docs under `../you-api/docs/` and `../you-ui/docs/`.

---

## Current status

**Phase 1 complete.** Entries are created, stored in DynamoDB, embedded via OpenAI, and upserted into Pinecone. Tags (people, locations, topics, mood, time_markers) are extracted asynchronously and written back to DynamoDB via a DynamoDB Streams Lambda. Semantic search via `/entries/search` is live. Grounded Q&A via `POST /entries/ask` completes the RAG layer: question is embedded, top-k entries retrieved from Pinecone, sent to gpt-4o-mini with tag metadata as structured context; returns `answer: str` + `sources: list[Entry]`. UI page at `/ask`.

**Phase 2 substantially complete.**
- `GET /entries/summary` — 7/30-day aggregation of topics, people, mood, locations. Lives on the dashboard as `PeriodStrip` + `PeopleCard` + `PlacesCard`.
- `GET /entries/narrative` — lazy on-demand LLM prose (gpt-4o-mini), cached in the `narratives` DynamoDB table. Dashboard shows `NarrativeStack` (two-deck week + month navigation) for current and previous periods.
- Stage 2 (EventBridge cron to pre-generate narratives for all users via `handler_narrative.py`) is **not started**.
- `GET /entries/on-this-day` — entries from the same month+day in previous years (up to 10 years back), queried via `user_timestamp_index` GSI. Dashboard renders `OnThisDayCard` at the top: hidden when empty, single card or chevron carousel otherwise.

Implementation specs:
- `../you-api/docs/phase2-dashboard.md`
- `../you-api/docs/narrative-recap.md`
- `../you-ui/docs/phase2-dashboard.md`
- `../you-ui/docs/narrative-recap.md`

**Phase 3 complete.** Timeline segmentation via 7-day sliding windows — topic cosine distance + sustained mood shifts + location bursts → boundaries at 75th-percentile divergence → sparse phase merging → gpt-4o names each phase (2–5 word title + prose description). Phases stored in the `narratives` table under `phase#{uuid}` / `phase_index#latest` sort keys. `YouPhaseFunction` Lambda runs weekly (Monday 01:15 UTC via EventBridge). UI at `/phases`: `PhaseTimeline` (horizontal colored bands) + `PhaseCard` (date range, signals, "Explore entries →" filtered by date range). NavBar link: "Timeline".

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

**Q&A lifecycle:**

```
UI: AskPage submits question
  → POST /entries/ask  { question }
    → QaService.ask_question(user_id, question)
      → EntryService.search_entries(user_id, question)  [embed → Pinecone top-k → DynamoDB batch-get]
      → LLMClient.answer_question(entries, question)    [gpt-4o-mini, strict grounding prompt]
      → QaResult { answer, sources: [Entry, …] }
```

The LLM is explicitly instructed to answer only from the provided entries and to admit when information is absent. Each entry is formatted with its extracted tags (topics, people, mood, location) above the prose text — the same structure used in the embedding pipeline — so the LLM has richer signal for questions about mood or people that may not be named explicitly in the text.

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
| `POST` | `/entries/ask` | Grounded Q&A — answer from entries only, returns `answer` + `sources` (entry IDs) |
| `GET` | `/entries/on-this-day` | Entries from same month+day in prior years, newest year first |
| `GET` | `/phases` | All detected phases (`?refresh=true` to re-run detection) |
| `GET` | `/phases/current` | The currently open phase (`is_open=true`), or null |
| `GET` | `/phases/{phase_id}` | Single phase record |

Route ordering matters: `/search`, `/ask`, `/summary`, `/narrative`, and `/on-this-day` must be declared before `/{entry_id}` in the router.

---

## Key design decisions

- **Tags are async-only.** The HTTP API never writes tags. This keeps the create endpoint fast and the tag-extraction pipeline independently deployable.
- **DynamoDB streams only on the prod table.** `test_entries` has no stream; the embedding Lambda never fires in tests.
- **`entry_id` is a UUID sort key, not time-ordered.** Time-range queries filter `timestamp` (ISO-8601 string) in Python after fetching all user entries. A `user_timestamp_index` GSI (PK: `user_id`, SK: `timestamp`) exists on both `entries` and `test_entries` tables; currently used only by `list_by_day` (on-this-day queries). ISO-8601 strings are lexicographically sortable, so `between("2023-03-15", "2023-03-16")` correctly captures all entries on that day regardless of time component.
- **topics are a controlled vocab:** `work, family, travel, health, reading, finance, relationships, hobbies, food, exercise`. Enforced in the OpenAI system prompt.
- **Narrative Stage 1 is lazy.** The API generates on first request per period and caches in DynamoDB. Stage 2 (EventBridge pre-generation for all users) is not yet implemented.
- **NarrativeStack goes beyond the spec.** The UI implements two-deck navigation (current + previous period) with fade transitions rather than the single-card design in the doc.
- **Q&A is a dedicated service.** `QaService` composes `EntryService` (for retrieval) and `LLMClient` (for generation) rather than embedding Q&A logic into `EntryService`. This keeps `EntryService` focused on CRUD/search and `QaService` independently testable.
- **Phases share the `narratives` DynamoDB table.** Sort key prefix `phase#` distinguishes phase records from narrative caches (`cache#week#…`, `cache#month#…`). `phase_index#latest` stores the ordered list of phase IDs per user. No separate phases table.
- **Phase detection is idempotent.** `get_phases(refresh=False)` returns the cached index; `refresh=True` (or the weekly cron) re-runs the full pipeline and overwrites. The `_FREEZE_WEEKS = 4` constant prevents re-naming phases older than 4 weeks.
- **Q&A sources are full Entry objects.** `QaResult.sources: list[Entry]` returns the complete entries retrieved from Pinecone. This lets the UI render date, location, and a text preview without a second fetch. Source cards open in a new tab.

---

## Known gaps

- **Narrative Stage 2 not started.** `handler_narrative.py` Lambda and EventBridge rules (weekly Sunday 23:55 UTC, monthly 1st 00:05 UTC) are documented in `../you-api/docs/narrative-recap.md` but not implemented.
- **`embedding_port.py` dead code.** `app/embedding/embedding_port.py` in you-api is a leftover from a refactor and is unused.
