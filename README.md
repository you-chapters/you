# YOU

A personal memory engine that transforms diary entries into structured life phases and evolving self-understanding.

## Goal

Enable natural interaction with personal history:
- Ask questions about past events
- Extract meaningful patterns
- Detect and analyze life phases
- Understand how behavior and “personality” change over time

## Core Concepts

- **Entries** — raw diary text
- **Signals** — extracted data (topics, people, mood, activities)
- **Phases** — time segments with stable patterns
- **Personality** — aggregated tendencies across phases

## Project Phases

### Phase 1 — Foundation
- Store diary entries
- Implement semantic search (RAG)
- Basic chat interface with contextual answers

### Phase 2 — Signal Extraction
- Extract structured data from entries:
    - topics
    - people
    - mood
- Store alongside raw data

### Phase 3 — Phase Detection
- Segment timeline into phases
- Generate summaries for each phase
- Enable phase-based queries

### Phase 4 — Personality Layer
- Aggregate patterns across phases
- Provide contextual insights:
    - behavioral tendencies
    - recurring patterns
    - phase comparisons

### Phase 5 — Insight Engine
- Detect transitions and trends
- Surface non-obvious patterns
- Provide reflective and predictive insights

## Architecture

### Backend
- AWS Lambda — core logic
- API Gateway — API layer

### Storage
- S3 — raw diary entries
- DynamoDB — structured data (signals, phases)

### Vector Search
- Pinecone / Weaviate / FAISS

### AI
- OpenAI API:
    - embeddings
    - chat/completion models

### Frontend
- Minimal web UI (React) or CLI

## Notes

- Start simple: entries + search
- Derive everything else incrementally
- Personality is inferred, not predefined
