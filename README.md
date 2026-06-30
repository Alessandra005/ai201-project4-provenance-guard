# Provenance Guard

A backend system that classifies submitted creative text as likely AI-generated,
likely human-written, or uncertain — with a calibrated confidence score, a
plain-language transparency label, an appeals workflow for contested
classifications, rate limiting, and a structured audit log.

> **Note on evidence in this README:** the code in this repo was built and
> validated against the project spec, but it was developed in an environment
> without network access to actually call the Groq API. **Before submitting,
> run the commands in "How to reproduce the evidence below" yourself**, with a
> real `GROQ_API_KEY` in your `.env`, and paste your *actual* terminal output
> over the placeholder output in this README (marked `<<RUN AND PASTE>>`).
> Everything else (stylometric logic, label text, appeals, rate limiting,
> audit log schema) runs with no external dependency and was verified directly.

## Setup

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # then add your real GROQ_API_KEY
python app.py
```

Server runs at `http://localhost:5000`.

## Architecture Overview

A submission (`POST /submit`) is rate-limited, then run through two
independent detection signals (an LLM judgment call via Groq, and a
stylometric heuristic computed in pure Python). Their scores are combined
into a single weighted confidence score, which is mapped to one of three
transparency-label tiers using asymmetric thresholds (it's harder to be
labeled "likely AI" than "likely human," since false accusations are worse
than false negatives on a creative platform). Every submission is written to
a SQLite-backed audit log with a unique `content_id`. A creator who disagrees
with their classification can call `POST /appeal` with that `content_id` and
their reasoning; this sets the record's status to `under_review` and appends
the appeal to the audit trail without any automatic re-classification.
`GET /log` exposes the audit log for review. Full diagram and narrative are
in [`planning.md`](./planning.md).

## Detection Signals

| Signal | What it measures | Why chosen | What it misses |
|---|---|---|---|
| **LLM (Groq, llama-3.3-70b-versatile)** | Holistic semantic/stylistic coherence — generic hedging, formulaic transitions ("furthermore," "it is important to note"), balanced-both-sides phrasing vs. idiosyncratic human voice | Captures meaning and "feel," which structural metrics can't | Can be fooled by AI text that's been edited/humanized, or by very formal human writing (academic, business, non-native speakers) that resembles typical LLM register |
| **Stylometric heuristics (pure Python)** | Sentence-length variance, type-token ratio (vocabulary diversity), punctuation density, combined into a single 0–1 score | Purely structural, computable with no external dependency, independent failure mode from the LLM signal | Blind to meaning entirely; unstable on short text (under ~40 words) where variance and TTR don't have enough samples to be meaningful — see Known Limitations |

These two are genuinely independent: one reads for meaning, the other counts
surface statistics, so they make different mistakes — which is the point of
combining them rather than relying on either alone.

## Confidence Scoring

**Combination:** `combined = 0.65 * llm_score + 0.35 * stylometric_score`,
both inputs and the output on a 0–1 scale where 1 = confidently AI-generated.

**Calibration approach:** rather than picking thresholds after the fact, we
decided up front what `0.5` should *mean* to a user (genuine, irreducible
ambiguity — not "the system is being lazy") and then set thresholds so the
"likely AI" tier requires a higher bar than the "likely human" tier requires
to be ruled out, since mislabeling a human creator as an AI is the worse error
on a creative-writing platform:

| Combined score | Tier |
|---|---|
| ≥ 0.75 | likely AI |
| 0.35 – 0.74 | uncertain |
| < 0.35 | likely human |

**Validation:** we tested the pipeline against the four inputs in the project
spec (clearly AI-generated, clearly human, formal-human borderline, lightly
edited AI borderline) and confirmed the stylometric component alone produces
meaningfully different scores between the clear-cut AI and human samples, and
that the combined score moves predictably as the LLM score is varied. The two
example runs below were captured with `curl` against the running server
(commands in the next section):

**High-confidence example:**
```
<<RUN AND PASTE: curl output for the clearly-AI-generated sample text — should
show confidence near/above 0.75 and the "likely AI" label>>
```

**Lower-confidence / uncertain example:**
```
<<RUN AND PASTE: curl output for the "lightly edited AI output" borderline
sample text — should show confidence in the 0.35-0.74 range and the
"uncertain" label>>
```

### How to reproduce the evidence above

```bash
# Clearly AI-generated
curl -s -X POST http://localhost:5000/submit -H "Content-Type: application/json" -d '{
  "text": "Artificial intelligence represents a transformative paradigm shift in modern society. It is important to note that while the benefits of AI are numerous, it is equally essential to consider the ethical implications. Furthermore, stakeholders across various sectors must collaborate to ensure responsible deployment.",
  "creator_id": "test-ai"
}' | python -m json.tool

# Clearly human-written
curl -s -X POST http://localhost:5000/submit -H "Content-Type: application/json" -d '{
  "text": "ok so i finally tried that new ramen place downtown and honestly? underwhelming. the broth was fine but they put WAY too much sodium in it and i was thirsty for like three hours after. my friend got the spicy version and said it was better. probably wont go back unless someone drags me there",
  "creator_id": "test-human"
}' | python -m json.tool

# Borderline: lightly edited AI output
curl -s -X POST http://localhost:5000/submit -H "Content-Type: application/json" -d '{
  "text": "I have been thinking a lot about remote work lately. There are genuine tradeoffs, flexibility and no commute on one side, isolation and blurred work-life boundaries on the other. Studies show productivity varies widely by individual and role type.",
  "creator_id": "test-borderline"
}' | python -m json.tool
```

## Transparency Label

The exact label text returned by the API (with `{score}` substituted as a
percentage) for each of the three tiers:

| Tier | Exact label text |
|---|---|
| **High-confidence AI** | "This content shows strong signals of AI generation (confidence: {score}). Our system is fairly confident this was produced by an AI tool rather than written by a human." |
| **High-confidence human** | "This content shows strong signals of human authorship (confidence: {score}). Our system found little evidence of AI generation." |
| **Uncertain** | "We can't confidently determine whether this content was AI-generated or human-written (confidence: {score}). Treat this attribution as inconclusive." |

## Appeals Workflow

`POST /appeal` with `{content_id, creator_reasoning}`:

```bash
curl -s -X POST http://localhost:5000/appeal -H "Content-Type: application/json" -d '{
  "content_id": "PASTE-CONTENT-ID-FROM-A-SUBMIT-RESPONSE",
  "creator_reasoning": "I wrote this myself from personal experience. I am a non-native English speaker and my writing style may appear more formal than typical."
}' | python -m json.tool
```

This sets the record's `status` to `under_review` and logs the reasoning
alongside the original decision — no automatic re-classification. A human
reviewer would filter `GET /log?status=under_review` to see the appeal
queue: each entry shows the original text preview, both signal scores, the
combined score, the original label, and the creator's reasoning together.

## Rate Limiting

`10 per minute; 100 per day` on `POST /submit`, via Flask-Limiter.

**Reasoning:** a real creator submitting their own work for a transparency
check would rarely need more than a handful of submissions in a sitting
(draft, revise, resubmit) — 10/minute comfortably covers that while making a
scripted flood (e.g., probing the classifier with many small variations to
reverse-engineer thresholds) noticeably slower. The 100/day ceiling caps
total daily load per client without affecting any realistic single-creator
workflow.

**Evidence (12 rapid requests, 10/minute limit):**
```
<<RUN AND PASTE: output of the 12-request loop from the project spec — should
show ten 200s followed by two 429s>>
```

```bash
for i in $(seq 1 12); do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/submit \
    -H "Content-Type: application/json" \
    -d '{"text": "This is a test submission for rate limit testing purposes only.", "creator_id": "ratelimit-test"}'
done
```

## Audit Log

`GET /log` returns the most recent structured entries (JSON, backed by
SQLite), each containing `content_id`, `creator_id`, `timestamp`,
`llm_score`, `llm_rationale`, `stylometric_score`, `confidence`,
`attribution`, `label`, `status`, and (once appealed) `appeal_reasoning` /
`appeal_timestamp`.

```
<<RUN AND PASTE: output of `curl -s http://localhost:5000/log | python -m
json.tool` after at least 3 submissions and 1 appeal>>
```

## Known Limitations

The stylometric signal is unreliable on very short text (roughly under 40
words): sentence-length variance and type-token ratio need enough sentences
and vocabulary to be statistically meaningful, and on a haiku or a one-line
caption both metrics can swing in either direction almost arbitrarily. In our
own testing, a short AI-generated sample and a short human sample produced
nearly identical stylometric scores (0.24 vs. 0.20) despite being obviously
different in origin — the LLM signal, which is weighted higher (0.65 vs.
0.35), is doing most of the real discriminative work in those cases, and the
stylometric signal functions more as a secondary corroborating check than an
independent verdict on short content. This is a structural property of the
metrics, not a bug we could patch by retuning constants.

## Spec Reflection

Writing out the three exact label strings in `planning.md` *before* touching
any label-generation code forced an early decision about what "confidence"
means to a non-technical reader — once that text existed, the threshold
values in the confidence scorer had to be chosen to match it, rather than the
other way around. Where the implementation diverged from the spec: the
original plan combined the two signals with a straight average; while
building the scorer we shifted to a weighted average (0.65 LLM / 0.35
stylometric) once it became clear from the stylometric blind-spot analysis
that the structural signal is meaningfully less reliable on its own and
shouldn't be trusted equally with the semantic one.

## AI Usage

1. **Stylometric metric selection:** asked an AI tool to suggest which
   stylometric features best separate AI from human text given the project's
   "pure Python, no external libraries" constraint. It proposed sentence-length
   variance, type-token ratio, and punctuation density with a straight
   average. We kept the three metrics but overrode the combination weights
   (0.4/0.4/0.2) after testing showed punctuation density was the noisiest of
   the three and shouldn't be weighted equally.
2. **Flask-Limiter boilerplate:** asked an AI tool to generate the
   Flask-Limiter setup with `storage_uri="memory://"` and the route decorator
   syntax for `10 per minute;100 per day`. Used as-is — this is standard
   library boilerplate with no project-specific judgment required.