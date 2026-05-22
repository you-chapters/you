# Phase 3.1 — Segment Timeline into Phases

## Philosophy

### What is a phase?

A phase is the unit of lived experience that memory actually organizes around — not the unit that calendars organize around. When a person looks back on a year, they do not think "January was interesting." They think: "that was the year I burned out at work," or "the period after we moved," or "when I was training for the marathon." These are phases: coherent, bounded chapters in which the character of daily life held a recognizable shape.

The defining property of a phase is **signal stability**. During a phase, the distribution of topics, mood, people, and locations is more internally consistent than it is across the phase boundary. The phase is not "what happened in March" — it is "the cluster of experience that cohered, whatever calendar span it covered." A phase can be three weeks or eight months. Its duration is determined by how long the signal pattern held.

### Why signal-based segmentation is more meaningful than calendar segmentation

Calendar segmentation is a proxy for something real, not the thing itself. The reason week/month narratives feel meaningful is not that a calendar week is a natural unit of human experience — it is that over short horizons, life tends to be relatively stable, so the calendar acts as a rough proxy. The proxy breaks down the moment life changes at a rate that does not align with calendar boundaries. A burnout beginning mid-November and ending in early January is a single experience. Calendar segmentation cuts it in half and discards the coherence.

More importantly, calendar segmentation is invisible to meaning-making. Telling a user "your week 47 narrative" gives them a container. Telling them "you went through a Social Flowering phase for six weeks this spring" gives them a concept — a named chapter of their own story. The name is only possible because the signal coherence was detected first.

### What cognitive model this mirrors

Human autobiographical memory is organized episodically, not chronologically. People chunk their life into what psychologists call "extended events" or "lifetime periods" — spans of time bound by a dominant theme or activity, retrieved as a unit. "My time in Berlin," "when I was writing the dissertation," "the period after the diagnosis." These chunks are identified by persistent defining attributes across the period — not by start and end dates.

When YOU detects a phase, it is not inventing a structure. It is discovering the structure that already exists in the signal data — the same structure the user's own memory would eventually surface. The system is doing computationally what the brain does implicitly over months of reflective consolidation.

### Relationship to the rest of the system

Phase 1: raw record. Phase 2: period digest. Phase 3: life chapter. The natural progression is complete. Phases are also the foundation for Phase 4's personality layer — personality should be characterized within phases, not across arbitrary date windows, because who a person is during a travel phase and who they are during a burnout phase may be substantially different signals. Without phase boundaries, Phase 4 aggregation would be meaningless.

---

## Implementation Plan

### 1. Signal Aggregation into Time Windows

All entries for a user are fetched and sorted by timestamp, then sliced into fixed-size windows (7 days, matching the existing week cadence; 14 days is an option for sparse journals).

Per window, four signal dimensions are computed:

- **Topic distribution vector** — 10-dimensional over the controlled vocabulary (work, family, travel, health, reading, finance, relationships, hobbies, food, exercise). Each component is the fraction of entries in the window that include that topic. Primary signal — controlled vocab makes it the most stable extractor.
- **Mood score** — categorical mood mapped to a numeric axis, mean taken over the window. Secondary signal.
- **People diversity** — count of distinct people mentioned. High diversity = social activity; low = isolation or consistency.
- **Location novelty** — fraction of locations in the window that are new relative to the prior 4 weeks. Captures travel and relocation.

Sparse windows (< 2 entries) are excluded from distance calculations but still assigned to whichever phase they temporally fall within.

### 2. Change-Point Detection

The algorithm operates on the sequence of window signal vectors, ordered chronologically.

**Primary: sliding cosine distance on topic distribution.** For each pair of adjacent windows, compute the cosine distance between their topic distribution vectors. The threshold is not fixed globally — it is calibrated per user as the **75th percentile** of all pairwise window distances in their history. This makes the system adaptive: a user who rarely changes topics has a lower absolute threshold than one who jumps topics constantly.

**Secondary: mood trajectory.** A sustained mood shift (mean mood changes sign and holds for 3+ consecutive windows) reinforces a topic-based candidate. Mood alone is not a boundary unless it persists for 6+ windows (6 weeks).

**Tertiary: location novelty burst.** A spike in location novelty (> 0.5 new locations in a window where the prior 4 windows averaged < 0.1) signals a travel or relocation boundary — detects "The Berlin Chapter" even when topics did not shift.

**Merge pass.** Adjacent micro-phases shorter than 3 windows are merged with their more similar neighbor by cosine distance, preventing over-segmentation of noisy weeks.

### 3. Phase Boundary Rules

| Rule | Value |
|------|-------|
| Minimum phase duration | 3 windows (21 days) — shorter spans absorbed into predecessor |
| Minimum entry density | 5 entries per phase — sparser phases absorbed into predecessor |
| Current phase | `end_date = null`, re-evaluated on every detection run |
| Frozen past phases | Boundaries and LLM content frozen once `end_date` is > 4 weeks in the past |
| Re-detection cadence | Weekly (EventBridge, Sunday 23:50 UTC) + on-demand via `?refresh=true` |

### 4. LLM Naming and Description

Once boundaries are determined, an LLM pass produces the human-facing representation for each phase.

**Two outputs per phase:**
1. **Title** — 2–5 word evocative chapter heading. No corporate language, no vague labels like "Period 1." Examples: "The Quiet Rebuilding," "Summer in Motion," "The Work Surge."
2. **Description** — 3–6 sentences of warm reflective prose: what characterized this phase, its emotional tone, who/what was prominent, how it felt to live through it. Same voice as existing narrative recap prompts.

**LLM inputs:**
- Full entry text within the phase (~40k character budget; representative sample of longest and most tag-rich entries)
- Aggregated signals: dominant topics (top 3), mean mood, top people (top 3), top locations (top 3), duration in days
- `phase_hint` string: e.g. "Heavy work + declining mood + limited people contact" — biases the title toward the detectable character

Past phases are frozen once generated (same staleness rule as past period narratives). The current open phase regenerates weekly.

### 5. Storage

The `you_narratives` DynamoDB table already reserves `phase#<uuid>` as a SK prefix (by design, Phase 2). No new infrastructure.

**Per-phase record (SK: `phase#<uuid>`):**

| Field | Type | Notes |
|-------|------|-------|
| `title` | str | LLM-generated chapter title |
| `description` | str | LLM-generated prose |
| `start_date` | str | "YYYY-MM-DD" |
| `end_date` | str \| null | null if current open phase |
| `entry_count` | int | |
| `dominant_topics` | list[str] | top 3 by frequency |
| `mean_mood` | float | numeric mood mean |
| `top_people` | list[str] | top 3 by mention count |
| `top_locations` | list[str] | top 3 by mention count |
| `generated_at` | str | ISO-8601 UTC |
| `is_open` | bool | true = current unclosed phase |

**Phase index record (SK: `phase_index#latest`)** — single record per user, overwritten on each detection run:

| Field | Notes |
|-------|-------|
| `phase_ids` | Ordered list of UUIDs, oldest to newest |
| `last_detected_at` | ISO-8601 UTC |
| `window_size_days` | 7 or 14, stored for reproducibility |

### 6. API Endpoints

Route ordering: `/phases/current` must be declared before `/phases/{phase_id}`.

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/phases` | All phases, oldest to newest. Lazy detection on first call. `?refresh=true` re-runs. Empty list if < 4 weeks of history. |
| `GET` | `/phases/current` | The current open phase only. |
| `GET` | `/phases/{phase_id}` | Single phase by UUID. 404 if not found or wrong user. |

`PhaseService` mirrors `NarrativeService`. Wiring in `dependencies.py` reuses the existing cached `_narrative_repository()` and `_llm_client()` factories.

### 7. Scheduled Detection

`handler_phase.py` — EventBridge rule fires weekly (Sunday 23:50 UTC, 5 min before narrative handler). Iterates all active user IDs via the same `_distinct_user_ids` scan pattern, calls `PhaseService.detect_and_store(user_id)` per user, logs failures without raising.

### 8. UI — Timeline Page

Dedicated route `/phases`. NavBar gains "Timeline" link.

**PhasesPage:**
- Vertical timeline scroll, oldest phase top, current bottom
- `PhaseCard` per phase: chapter title, date range + entry count, prose description, signal chips (topic pills, mood indicator, top people/locations)
- Current phase: no end date, "ongoing" indicator, refresh button
- "Explore entries" link → `/entries?from=YYYY-MM-DD&to=YYYY-MM-DD`
- Empty state: threshold hint if < 4 weeks of history

---

## Data Flow

```
you_entries (DynamoDB)
  ↓  EntryRepository.list_by_user(user_id)

PhaseService.detect_and_store(user_id)
  ├─ 1. AGGREGATE
  │     Sort entries chronologically → 7-day windows
  │     Per window: topic_vector (10-dim), mood_score, people_diversity, location_novelty
  │
  ├─ 2. DETECT BOUNDARIES
  │     Cosine distance between adjacent window topic_vectors
  │     Threshold = 75th percentile of user's distance distribution
  │     Reinforce with mood trajectory + location novelty bursts
  │     Merge micro-phases < 3 windows
  │     → list of (start_date, end_date) pairs
  │
  ├─ 3. ASSIGN + AGGREGATE PER PHASE
  │     Assign entries to phases by timestamp range
  │     Compute dominant_topics, mean_mood, top_people, top_locations per phase
  │
  ├─ 4. LLM NAMING  (skip frozen past phases that already have a record)
  │     phase_hint from signal summary
  │     LLMClient.generate_phase(entries, signals_summary, hint) → {title, description}
  │
  └─ 5. STORE
        Write phase#<uuid> records to you_narratives (new or open phases only)
        Overwrite phase_index#latest

GET /phases
  ↓  get(user_id, "phase_index#latest") → phase_ids
  ↓  batch_get(user_id, phase_ids)      → PhaseRecord list

PhasesPage (React)
  → getPhases() on mount → PhaseCard[] in timeline order
  → "Explore entries" → /entries?from=...&to=...
```

---

## Critical Files

| File | Change |
|------|--------|
| `app/llm/llm_client.py` | Add `generate_phase` to `LLMClient` ABC + implementations |
| `app/repositories/narrative_repository.py` | Add `batch_get` for fetching multiple phase records |
| `app/dependencies.py` | Add `get_phase_service()` |
| `app/routers/phases.py` | New router, 3 endpoints |
| `app/services/phase_service.py` | New service: detection + LLM + storage |
| `handler_phase.py` | New Lambda handler, mirrors `handler_narrative.py` |
| `../you-ui/src/api/phases.ts` | `getPhases`, `getPhase` |
| `../you-ui/src/types/phase.ts` | `PhaseRecord` TypeScript type |
| `../you-ui/src/pages/PhasesPage.tsx` | Timeline page |
| `../you-ui/src/App.tsx` | Add `/phases` route |