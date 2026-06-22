# Growth Features

Features prioritized for cold-start appeal and user retention. Ordered by impact on getting someone from "first open" to "I can't stop using this".

## Priority table

| # | Feature | Why it matters | Effort | Impact |
|---|---|---|---|---|
| 1 | **Voice entry** | Removes the biggest friction point — typing a diary entry feels like work; speaking for 30 seconds while commuting doesn't. Transcribe via Whisper API, pipe through existing entry pipeline unchanged. | Medium | Very high |
| 2 | **Import (Day One / markdown / plain text)** | Users who already have years of diary data get an instant "aha moment" — phases, Q&A, and narratives work on day 1 instead of month 6. Eliminates the cold-start problem for the target demographic. | Medium | Very high |
| 3 | **Weekly email digest** | One Sunday email: narrative recap + one On This Day memory + a prompt to reflect. Brings the app to the user; no app open required. Best retention loop for a journaling product. | Low–medium | High |
| 4 | **Mood chart** | Signal data already exists. A simple time-series chart of mood over weeks/months is the first thing that makes users feel "this thing knows me". No new infra. | Low | High |
| 5 | **Entity pages** | Click a person or location chip → see every entry mentioning them, ordered by date. Makes the social/spatial layer feel real rather than decorative. Data already extracted. | Low–medium | Medium–high |
| 6 | **Entry deletion** | "Tearing out a page" — entries cannot be edited (cast in stone by design) but can be deleted entirely. Simpler than editing: no pipeline re-trigger needed, just remove from DynamoDB and Pinecone. | Low | Medium |
| 7 | **Phase 4 — Personality layer** | Cross-phase signal trends, recurring pattern detection, LLM-generated prose portrait. This is the feature that turns YOU from "a better diary search" into "a mirror for your life". Requires a few months of entries to be meaningful. | High | Very high (long-term) |
| 8 | **Narrative Stage 2 (pre-generation)** | Dashboard loads instantly instead of spinning on first visit per period. Quality-of-life, not a growth feature — but makes the product feel polished. | Low–medium | Low–medium |
| 9 | **Export (JSON / markdown)** | Trust signal. Users are more willing to write private thoughts when they know they can get their data out. Single endpoint, no new infra. | Low | Medium (trust) |
| 10 | **Streak counter** | Simple but effective engagement hook — consecutive days with entries. Pairs well with the weekly digest. | Low | Low–medium |

---

## Recommended sequence

**Now (cold-start removal):**
Voice entry + import. These two together mean a new user can either start writing instantly (voice) or arrive with years of data (import). Neither requires any change to the existing pipeline — just new ingestion paths feeding `POST /entries`.

**Next (retention loop):**
Weekly digest + mood chart. Once someone has a week of data, these two features give them a reason to open the app and a reason to check their email. Both are low-effort relative to impact.

**Then (depth):**
Entity pages + entry deletion. These make the app feel complete for a serious journaler who has been using it for 1–2 months.

**Long game:**
Phase 4. Ship it when users have at least 2–3 detected phases. The personality layer is the moat — no other journaling app builds a longitudinal self-model.