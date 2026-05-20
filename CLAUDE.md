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

**Phase 2 in design.** Implementation specs for the dashboard homepage are in:
- `../you-api/docs/phase2-dashboard.md`
- `../you-ui/docs/phase2-dashboard.md`

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

The write path is non-blocking by design. The UI receives an entry with `tags: null` on creation; tags appear only after the async pipeline completes. The frontend does not currently display tags, but the type should include them (`tags?: EntryTags | null`).

**Authentication:** Cognito issues JWTs. API Gateway validates them. The `sub` claim is the `user_id` — it is extracted server-side and never accepted from request bodies. The UI attaches the Cognito ID token as `Authorization: Bearer` on every request.

**Secrets:** stored in AWS SSM Parameter Store (`/you-api/*`). At runtime, `app/config.py` detects SSM paths by a leading `/` and fetches them. Locally, set env vars to raw values.

**Pinecone metadata per vector:** `user_id`, `timestamp` (unix int), `topics`, `mood`, `people`, `locations`. All searches filter by `user_id` at the Pinecone level.

---

## Key design decisions

- **Tags are async-only.** The HTTP API never writes tags. This keeps the create endpoint fast and the tag-extraction pipeline independently deployable.
- **DynamoDB streams only on the prod table.** `test_entries` has no stream; the embedding Lambda never fires in tests.
- **`entry_id` is a UUID sort key, not time-ordered.** Time-range queries filter `timestamp` (ISO-8601 string) in Python after fetching all user entries. A GSI on `timestamp` is not yet needed at current scale.
- **topics are a controlled vocab:** `work, family, travel, health, reading, finance, relationships, hobbies, food, exercise`. Enforced in the OpenAI system prompt.