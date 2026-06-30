# Provenance Guard — Planning Document

## Architecture Narrative

A creator submits a piece of text via `POST /submit` along with a `creator_id`. The
request hits the Flask app, which is rate-limited per-creator to prevent flooding.
The raw text is passed to two independent detection signals:

1. **LLM signal** (Groq, `llama-3.3-70b-versatile`) — the model is prompted to
   assess whether the text reads as human- or AI-written and to return a score
   from 0 (confidently human) to 1 (confidently AI), plus a short rationale.
2. **Stylometric signal** (pure Python) — sentence-length variance, type-token
   ratio (vocabulary diversity), and punctuation density are computed and
   combined into a single 0–1 "AI-likeness" score. AI text tends to be more
   uniform; human text is noisier.

Both raw signal scores are passed to the **confidence scorer**, which combines
them (weighted average, LLM weighted higher since it's more holistic) into a
single combined confidence score in [0,1], where the score represents
*confidence that the content is AI-generated*. That combined score is mapped to
one of three **label tiers** (likely AI / uncertain / likely human) using
asymmetric thresholds (see below) and rendered as the exact label text a
reader would see. A unique `content_id` is generated, and the full record
(text hash, both raw signals, combined score, label, timestamp, status) is
written to the **audit log** (SQLite). The endpoint returns `content_id`,
`attribution`, `confidence`, and `label` to the caller.

If a creator disputes the result, they call `POST /appeal` with the
`content_id` and their reasoning. This updates that content's `status` to
`under_review` in the same SQLite table and appends an audit log entry
recording the appeal reasoning alongside the original decision — no
re-classification happens automatically. `GET /log` exposes the most recent
audit entries as JSON for review/grading.

## Architecture

```
                         ┌─────────────────────────┐
                         │      POST /submit        │
                         │  {text, creator_id}      │
                         └────────────┬─────────────┘
                                      │ rate-limited (Flask-Limiter)
                                      ▼
                         ┌─────────────────────────┐
                         │   raw text fan-out        │
                         └──────┬───────────┬───────┘
                                │           │
                    ┌───────────▼──┐   ┌────▼────────────┐
                    │ Signal 1:     │   │ Signal 2:        │
                    │ Groq LLM      │   │ Stylometric      │
                    │ -> score 0-1  │   │ heuristics       │
                    │ + rationale   │   │ -> score 0-1     │
                    └───────┬───────┘   └────────┬─────────┘
                            │                    │
                            └─────────┬──────────┘
                                      ▼
                       ┌───────────────────────────┐
                       │  Confidence Scorer          │
                       │  weighted combine -> 0-1     │
                       └────────────┬────────────────┘
                                      ▼
                       ┌───────────────────────────┐
                       │  Label Generator             │
                       │  score -> label tier + text   │
                       └────────────┬────────────────┘
                                      ▼
                       ┌───────────────────────────┐
                       │  Audit Log (SQLite)          │
                       │  content_id, signals, score, │
                       │  label, status, timestamp     │
                       └────────────┬────────────────┘
                                      ▼
                       ┌───────────────────────────┐
                       │  Response: content_id,       │
                       │  attribution, confidence,    │
                       │  label                       │
                       └───────────────────────────┘


                         ┌─────────────────────────┐
                         │      POST /appeal         │
                         │ {content_id, reasoning}   │
                         └────────────┬─────────────┘
                                      ▼
                       ┌───────────────────────────┐
                       │ Look up content_id in log   │
                       │ set status=under_review     │
                       │ append appeal_reasoning      │
                       └────────────┬────────────────┘
                                      ▼
                       ┌───────────────────────────┐
                       │  Response: confirmation      │
                       └───────────────────────────┘

                         GET /log -> returns last N
                         structured audit entries as JSON
```

## API Surface

- `POST /submit` — body: `{text, creator_id}` → returns `{content_id, attribution, confidence, label, signals: {llm_score, stylometric_score}}`
- `POST /appeal` — body: `{content_id, creator_reasoning}` → returns `{content_id, status, message}`
- `GET /log` — returns `{entries: [...]}` (most recent N audit entries)
- `GET /log/<content_id>` — returns the single record for a content_id (convenience, not required)

## 1. Detection Signals

| Signal | What it measures | Output format | Blind spot |
|---|---|---|---|
| **LLM (Groq)** | Holistic semantic/stylistic coherence — does the prose "feel" like typical LLM output (hedging, balanced-both-sides phrasing, generic transitions like "furthermore," "it is important to note") vs. idiosyncratic human voice | float 0–1 (0=human, 1=AI) + short rationale string, parsed from a JSON-constrained prompt | Confidently fooled by AI text that has been deliberately humanized/edited, or by very formal, hedge-heavy *human* academic/business writing that mimics LLM register |
| **Stylometric heuristics** | Structural statistics: sentence-length variance, type-token ratio (vocabulary diversity), punctuation density | Each metric normalized to 0–1, averaged into a single 0–1 AI-likeness score | Blind to *content/meaning* — a human writer with a very uniform, controlled style (e.g., legal writing, a non-native speaker writing carefully) scores artificially high; very short text gives unstable statistics |

These are genuinely independent: one is semantic (the model reads for meaning
and "feel"), the other is purely structural (counts and ratios on the raw
text), so they fail in different cases — that's why combining them is more
informative than either alone.

**Combination:** `combined = 0.65 * llm_score + 0.35 * stylometric_score`.
LLM is weighted higher because it captures meaning, not just surface
statistics, but stylometrics acts as a sanity check / independent corroboration.

## 2. Uncertainty Representation

The combined score is *confidence that the content is AI-generated*, not a
generic "confidence" magnitude. `0.6` means "60% lean toward AI-generated, but
real ambiguity remains" — it must produce a visibly hedged label, not a flat
binary verdict.

**Asymmetric thresholds** (false positives — calling a human's work AI — are
worse than false negatives on a creative platform, so the "likely AI" bar is
set higher than the "likely human" bar is set low):

| Combined score | Tier | Label shown |
|---|---|---|
| `>= 0.75` | likely AI | "high-confidence AI" label |
| `0.35 – 0.74` | uncertain | "uncertain" label |
| `< 0.35` | likely human | "high-confidence human" label |

This means content needs to score quite high before it's confidently called
AI, but doesn't need to score as low to be confidently called human — biasing
the system away from false accusations.

## 3. Transparency Label Design (exact text)

- **High-confidence AI:**
  `"This content shows strong signals of AI generation (confidence: {score}). Our system is fairly confident this was produced by an AI tool rather than written by a human."`
- **High-confidence human:**
  `"This content shows strong signals of human authorship (confidence: {score}). Our system found little evidence of AI generation."`
- **Uncertain:**
  `"We can't confidently determine whether this content was AI-generated or human-written (confidence: {score}). Treat this attribution as inconclusive."`

`{score}` is the combined confidence score formatted as a percentage AI-likeness, e.g. "82%".

## 4. Appeals Workflow

- Any creator can appeal a classification on their own content by submitting
  `content_id` and free-text `creator_reasoning`.
- On receipt: the system looks up the original record by `content_id`, sets
  `status = "under_review"`, and writes a new audit log entry that includes
  the original decision (signals, score, label) plus the appeal reasoning and
  a timestamp.
- No automated re-classification occurs — a human reviewer is expected to
  look at the appeal queue (in this implementation, the `GET /log` endpoint
  filtered to `status=under_review` serves as that queue) and would see: the
  original text's content_id, both signal scores, the combined score, the
  original label, and the creator's reasoning side by side.

## 5. Anticipated Edge Cases

1. **Very short submissions** (a haiku, a one-line caption): stylometric
   metrics like sentence-length variance and type-token ratio are statistically
   unstable on tiny samples, so the stylometric signal may swing wildly and
   drag the combined score in either direction. Mitigation: flag content under
   ~40 words as low-confidence in the rationale, but still return a label.
2. **Formal, hedge-heavy human writing** (academic abstracts, business memos,
   non-native English speakers writing carefully): both signals can be fooled
   here — the LLM signal because the register resembles typical AI output, and
   stylometrics because careful, edited writing is unusually uniform. This is
   exactly the false-positive case the asymmetric thresholds are designed to
   reduce, but it will still occasionally misfire — which is why appeals exist.

## AI Tool Plan

- **M3 (submission endpoint + first signal):** Provide the AI tool the
  "Detection Signals" table above plus the architecture diagram. Ask it to
  generate the Flask app skeleton with a stubbed `POST /submit` route, and
  the Groq-based `get_llm_signal(text)` function returning `(score, rationale)`.
  Verify by calling `get_llm_signal` directly on 2–3 sample texts before
  wiring it into the route, checking the score is in [0,1].
- **M4 (second signal + confidence scoring):** Provide the Detection Signals
  table, the Uncertainty Representation section, and the diagram. Ask for the
  `get_stylometric_signal(text)` function and the `combine_scores(llm, style)`
  function implementing the weighted formula and thresholds above. Verify by
  printing both signals separately on the 4 test inputs from the project spec
  and confirming the combined score lands in the expected tier for each.
- **M5 (production layer):** Provide the Transparency Label and Appeals
  Workflow sections plus the diagram. Ask for `generate_label(score)` mapping
  to the three exact strings above, and the `POST /appeal` endpoint. Verify by
  calling `generate_label` at 0.2, 0.5, 0.9 and confirming all three variants
  appear verbatim, then testing the appeal endpoint updates status and is
  visible via `GET /log`.

## Stretch Feature: Analytics Dashboard

**Goal:** a simple view surfacing detection patterns, appeal rates, and one
additional metric, built on top of the existing `audit_log` table — no new
storage needed.

**Metrics to surface:**
1. **Detection patterns** — count and percentage of submissions in each
   attribution tier (`likely_ai` / `uncertain` / `likely_human`), plus the
   average combined confidence score overall.
2. **Appeal rate** — `(# entries with status = under_review) / (total
   entries)`, expressed as a percentage, since that's the cleanest proxy for
   "how often creators dispute a verdict."
3. **Additional metric — signal disagreement rate:** the percentage of
   submissions where the LLM signal and stylometric signal land on opposite
   sides of the midpoint (one says "more AI-like," 0.5+, the other says "more
   human-like," <0.5). This is a genuinely useful operational metric beyond
   what the spec asks for: a high disagreement rate would tell a platform
   operator the two signals are capturing different things often enough that
   single-signal detection would have been unreliable — i.e., it's a
   real-time justification for the multi-signal design itself, not just a
   vanity stat.

**API surface:**
- `GET /analytics` — returns the three metrics above as JSON, computed live
  from the `audit_log` table (no caching needed at this scale).
- `GET /dashboard` — a minimal server-rendered HTML page (no JS framework,
  no build step) that fetches `/analytics` and renders the numbers as plain
  text/simple bars, so a non-technical reviewer can see it directly in a
  browser without needing `curl` or a JSON viewer.

**Implementation plan:** add a `compute_analytics()` helper to `app.py` that
runs a few `SELECT` aggregate queries against `audit_log` (counts grouped by
`attribution`, count where `status = 'under_review'`, average confidence,
and a Python-side pass over rows to compute the disagreement rate since it
requires comparing two columns per row rather than a single aggregate).
Both routes reuse this helper so the JSON and HTML views never disagree with
each other.

**Verification plan:** after building, submit a small mixed batch of AI,
human, and borderline test inputs (reusing the ones from Milestone 4) plus
at least one appeal, then check that `/analytics` numbers match a manual
count from `/log`, and that `/dashboard` renders without error in a browser.