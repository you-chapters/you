# Roadmap

## Small gaps in existing features

**Narrative Stage 2** — `handler_narrative.py` Lambda and EventBridge rules (Sunday 23:55 UTC weekly, 1st of month 00:05 UTC) are documented in `you-api/docs/narrative-recap.md` but not implemented. Right now narratives are lazy (generated on first request). Stage 2 would pre-generate them for all active users so the dashboard loads instantly.

**`embedding_port.py` dead code** — `app/embedding/embedding_port.py` in you-api is an unused leftover from a refactor.

---

## Phase 4 — Personality Layer

- **Cross-phase signal trends** — how dominant topics, mood, and people change from phase to phase
- **Recurring pattern detection** — things that appear in every phase regardless of context
- **Phase comparison** — side-by-side view of two phases: mood delta, topic overlap, people in common
- **Personality summary** — LLM-generated prose portrait derived from all phase signals, periodically refreshed

Likely: `PersonalityService`, `/personality` endpoint, dedicated UI page.

---

## Phase 5 — Insight Engine

- **Transition detection** — identify when and why phase shifts happened
- **Non-obvious pattern surfacing** — things you wouldn't notice yourself (e.g. mood consistently drops before travel)
- **Predictive/reflective insights** — "you're in a phase that looks like phase 2 from 2023; here's what happened next"

Most LLM-intensive phase. Needs a large entry corpus to be useful.

---

## Quality-of-life features

| Feature | Effort | Notes |
|---|---|---|
| **Mood chart** | Low | Data already extracted — needs a time-series visualization on the dashboard |
| **Entity pages** | Low–medium | Click a person/location chip → dedicated page with all mentions and a timeline |
| **Entry editing** | Medium | Needs to re-trigger the embedding pipeline on save |
| **Export** | Low | `GET /entries/export` returning JSON or markdown; no new infra |
| **Streak counter** | Low | Consecutive days with entries |

---

## Priority order

1. Narrative Stage 2 — finishes existing work
2. Mood chart + entity pages — high value, low effort, data already exists
3. Entry editing — basic journal hygiene
4. Phase 4 — when there's enough phase history for cross-phase comparisons to be meaningful
5. Phase 5 — needs a large corpus; long-term
