# Rehearse: Master Build Prompt

> A personal app for practicing spoken English and professional communication through a structured curriculum, a live AI voice partner, and evidence-based feedback from audio, video, and transcripts.
>
> "Rehearse" is a working name. Rename it freely.

---

## How to use this file

1. Create an empty repository and save this file at the root as `CLAUDE.md` (for Claude Code) or as your project rules file in Cursor or a similar tool.
2. Tell the agent: **"Read CLAUDE.md fully, then start Phase 0."**
3. The agent builds one phase, stops, and reports back. Review it, run it, then say **"Start Phase N"** for the next one.
4. If something is off, fix it within the current phase before moving on. Later phases depend on earlier ones being solid.

---

## 0. Instructions for the coding agent

You are building this app for a single user: a developer who wants to improve their spoken English and workplace communication. Follow these rules throughout:

- **Work one phase at a time.** At the end of each phase, stop and report: what you built, how to run it, and the result of every acceptance criterion (pass or fail, with evidence). Do not start the next phase until told to.
- **Never guess third-party APIs.** Before writing any integration, read the provider's current official docs (Sarvam: docs.sarvam.ai; Anthropic: docs.claude.com; Azure Speech; MediaPipe). Pin exact package versions. If the docs contradict this file, follow the docs and tell me what differed.
- **Keep providers swappable.** All AI services sit behind the interfaces in section 3. No route, component, or core module may import a provider SDK directly.
- **Secrets stay on the backend.** API keys live in `api/.env` and are never sent to the browser.
- **Test the logic that produces numbers.** Metrics, evidence gates, alignment, ranking, and mastery evaluation need unit tests with fixtures.
- **Log the cost of every external API call** (provider, model, units, estimated USD) to the database.
- **Ask before adding dependencies** outside the stack in section 4.
- **Follow the truthfulness contract in section 2 exactly.** It is the core of the product, not a nice-to-have.

---

## 1. Product summary

The user moves through a curriculum of tracks, units, and lessons (similar in shape to Duolingo or Stimuler). Each lesson has a measurable objective. The user learns a few target phrases, has a spoken conversation with an AI partner who plays a role and steers toward the lesson goal, then receives a small number of verified insights with replayable evidence, retries their weakest moments, and passes or repeats the lesson based on explicit mastery criteria.

**Lesson flow:** Learn (2–3 min) → Talk (5–10 min) → Analyze → Insights (max 3) → Retry → Mastery check → next lesson or auto-generated review lesson.

**Tracks for v1:** Workplace basics, Meetings, Interviews, Presentations, Client communication.

---

## 2. Truthfulness contract (non-negotiable)

The user explicitly wants no fake insights. Every rule below must be enforced in code, not only in prompts.

1. **Numbers come from code, never from an LLM.** Speaking rate, filler counts, pauses, talk ratio, vocabulary range, eye contact, and target-phrase usage are computed deterministically. LLMs may receive these numbers to interpret, but are never asked to produce a number shown to the user.
2. **Every language insight quotes the user, and code verifies the quote.** The findings model must return the exact quoted words and the `turn_id`. A finding is dropped unless its normalized quote appears in that turn's transcript.
3. **Two transcribers must agree before grammar is flagged.** Each user turn is transcribed by Sarvam (verbatim mode) and a local faster-whisper model. If the two transcripts disagree over the flagged span (word-level similarity below 0.9 after normalization), the finding is dropped, because the "mistake" may be a transcription error.
4. **A verifier pass removes nitpicks.** A second model call reviews each surviving finding and rejects anything that is acceptable in natural spoken professional English. Regional Indian English usages (for example "prepone", "do the needful") are labeled "may confuse international listeners", never "wrong".
5. **No trends without enough data.** Trend statements require at least 3 sessions. Comparisons are against the user's own baseline and the lesson criteria, never generic population norms.
6. **Video metrics require good video.** Visual metrics are reported only if a face was detected in at least 90% of speaking frames. Otherwise the report says there wasn't enough usable video.
7. **Never claim:** emotion or confidence scores from the face, personality judgments, pronunciation scores (until a dedicated pronunciation scorer is added), or generic praise. Positive feedback must also cite evidence.
8. **At most 3 insight cards per session,** ranked by the scoring rule in Phase 4. Everything else goes into a collapsed details section.
9. **Every dropped finding is logged** with the gate that dropped it, and viewable in a developer debug panel.
10. **Every insight card has "Accurate" and "Not accurate" buttons.** Ratings feed the eval harness in Phase 6.

---

## 3. Core interfaces (backend, Python)

```python
class Transcriber(Protocol):
    name: str
    async def transcribe(self, audio: bytes, sample_rate: int) -> Transcript: ...
    # Transcript: text, segments[{start_ms, end_ms, text}], words[{start_ms, end_ms, text}] | None

class Partner(Protocol):
    def reply(self, lesson: LessonSpec | None, scenario: Scenario,
              history: list[Turn], targets: list[str]) -> AsyncIterator[str]: ...

class Voice(Protocol):
    def speak(self, text: str, voice_id: str) -> AsyncIterator[bytes]: ...

class FindingsEngine(Protocol):
    async def propose(self, timeline: Timeline, metrics: SessionMetrics,
                      taxonomy: PatternTaxonomy) -> list[CandidateFinding]: ...

class Verifier(Protocol):
    async def verify(self, finding: CandidateFinding, context: TurnContext) -> Verdict: ...
```

**Default implementations:**

| Role | Implementation | Notes |
|---|---|---|
| Primary transcriber | Sarvam Saaras, `mode="verbatim"` | Keeps fillers and repetitions. REST handles clips up to 30 s; use streaming or batch for longer turns. Check docs for current model name (`saaras:v3` or later). |
| Second transcriber | faster-whisper, local, `word_timestamps=True` | Used for agreement checks and word timings. |
| Partner | Claude Haiku 4.5 (`claude-haiku-4-5-20251001`) | Fast, streamed. Use prompt caching for the system prompt. |
| Findings + Verifier | Claude Sonnet 5 (`claude-sonnet-5`) | Runs once per session, after it ends. |
| Voice | Sarvam Bulbul v3 (Indian English) and Azure Neural TTS (US/UK/AU) | Voice chosen per scenario or in settings. Stream audio. |
| Voice activity | Silero VAD, local | Pauses, voiced time, response latency. |
| Visual analysis | MediaPipe Tasks Vision in the browser | Face Landmarker; raw video never leaves the machine. |

---

## 4. Tech stack

- **Monorepo** with `pnpm` workspaces for the web app and `uv` for Python.
- **Web:** Next.js (App Router, latest stable), TypeScript strict, Tailwind CSS v4, shadcn/ui (restyled to the tokens below), lucide-react icons, `@mediapipe/tasks-vision`.
- **Fonts:** Geist Sans and Geist Mono via `next/font/google` (`Geist`, `Geist_Mono`) or the `geist` npm package. Expose as CSS variables `--font-geist-sans` and `--font-geist-mono`.
- **API:** Python 3.12, FastAPI, SQLModel on SQLite, Alembic migrations, `anthropic`, `sarvamai`, `azure-cognitiveservices-speech`, `faster-whisper`, Silero VAD, `pytest`.
- **Streaming:** Server-Sent Events for partner text; chunked audio for TTS.
- **Storage:** SQLite at `api/data/app.db`; recordings in `api/data/sessions/{session_id}/`.
- **One command to run everything:** `pnpm dev` starts both web and API.

---

## 5. Repository structure

```
/
├── CLAUDE.md
├── web/
│   ├── app/
│   │   ├── (shell)/learn/          # curriculum path
│   │   ├── (shell)/lesson/[id]/    # learn → talk → review → retry
│   │   ├── (shell)/progress/       # dashboard
│   │   ├── (shell)/settings/
│   │   └── design/                 # living design system page
│   ├── components/ui/              # restyled shadcn primitives
│   ├── components/session/         # voice ring, push-to-talk, transcript
│   ├── components/review/          # timeline, insight cards, player
│   ├── lib/capture/                # recorder, MediaPipe, calibration
│   └── styles/tokens.css
├── api/
│   ├── app/api/                    # FastAPI routes
│   ├── app/core/                   # orchestrator, timeline, metrics, gates, ranking, mastery
│   ├── app/providers/              # sarvam_stt.py, whisper_stt.py, claude_partner.py, ...
│   ├── app/prompts/                # partner.md, findings.md, verifier.md
│   ├── app/db/
│   ├── curriculum/                 # tracks/*.yaml, patterns.yaml
│   ├── evals/
│   └── tests/
└── package.json
```

---

## 6. Design system: Vercel-like dark

The feel is calm, precise, and technical: a near-monochrome interface where the only strong color is information. Geist does the heavy lifting. No gradients, no glassmorphism, no glow.

### 6.1 Color tokens (`web/styles/tokens.css`)

```css
:root {
  --bg: #000000;              /* page */
  --bg-raised: #0a0a0a;       /* panels, sidebar */
  --bg-hover: #1a1a1a;
  --border: #242424;          /* default 1px hairline */
  --border-strong: #333333;
  --text: #ededed;
  --text-secondary: #a1a1a1;
  --text-muted: #6b6b6b;
  --primary-bg: #ededed;      /* primary button: light on dark */
  --primary-text: #0a0a0a;
  --focus: #0070f3;           /* focus rings, links, active state */

  /* Information colors: used ONLY for insight categories and status */
  --lang: #3b82f6;            /* language insights (grammar, phrasing, vocabulary) */
  --delivery: #f5a524;        /* pace, fillers, pauses */
  --presence: #a78bfa;        /* video: eye contact, framing */
  --strength: #22c55e;        /* evidence-backed positives, passed criteria */
  --danger: #ef4444;          /* errors, recording indicator */
}
```

Rules: color never decorates. If something is colored, it means a category or a state, and the same color means the same thing everywhere (timeline marker, card edge, dashboard line).

### 6.2 Typography

- **Geist Sans** for all interface text. **Geist Mono** only for timestamps, metric values, keyboard hints, and code. Use `font-variant-numeric: tabular-nums` on every changing number.
- Scale: 12 / 14 (body) / 16 / 20 / 24 / 32 / 48 px. Weights 400, 500, 600 only.
- Headings use tight tracking (`letter-spacing: -0.02em` at 24 px and up). Body line-height 1.6. Line length under 72 characters in reading areas.
- Sentence case everywhere. No all-caps labels, no eyebrow labels above headings.

### 6.3 Shape, space, and motion

- 4 px spacing grid. Radius: 6 px for inputs and buttons, 10 px for panels, full for the voice ring and avatars. Not everything is a card; use hairline dividers and whitespace for grouping.
- 1 px borders in `--border`; no drop shadows except a single subtle shadow on popovers and dialogs.
- Motion: 150 ms ease-out for state changes that respond to the user. The only ambient animation in the app is the voice ring. Respect `prefers-reduced-motion`.
- Visible focus ring on every interactive element: 2 px `--focus` with 2 px offset.

### 6.4 Signature elements

Spend visual boldness in exactly two places and keep everything else quiet:

1. **The voice ring (live session).** A thin circular ring in the center of the screen that responds to real audio amplitude: gray and still while idle, white and reacting to your voice while you hold Space, softly pulsing with the partner's audio while it speaks. Your transcript and the partner's appear in a narrow column beside it.
2. **The evidence timeline (review).** A full-width strip under the session video showing your speech as a waveform, with small colored markers (using the information colors) at every event: fillers, long pauses, eye-contact drops, and each insight. Clicking a marker seeks the video and opens its card. This is the product's identity: every claim is a point you can click and replay.

### 6.5 Key screens (wireframes)

**Learn path**
```
┌ sidebar ──┐┌──────────────────────────────────────────────┐
│ Learn     ││ Meetings                          Level B1   │
│ Progress  ││ ─────────────────────────────────────────── │
│ Settings  ││ Unit 1  Status updates                       │
│           ││   ✓ Sharing what you did yesterday           │
│           ││   ● Giving a status update      [Start]      │
│           ││   ○ Flagging a blocker                       │
│           ││ Unit 2  Disagreeing politely        locked   │
│           ││ Review  2 patterns to practice     [Start]   │
└───────────┘└──────────────────────────────────────────────┘
```

**Live session**
```
┌─────────────────────────────────────────────────────────────┐
│ Giving a status update   Priya, your manager      04:12  ■  │
│                                                             │
│                    ◯  voice ring           │ Priya: So where│
│                                            │ are we on the  │
│                                            │ migration?     │
│           Hold Space to talk               │ You: We've...  │
│                                                             │
│ Goal: headline first, use 2 of 3 target phrases  [End]      │
└─────────────────────────────────────────────────────────────┘
```

**Review**
```
┌─────────────────────────────────────────────────────────────┐
│ ┌ session video ──────────────┐  Insight 1 of 3             │
│ │                             │  At 2:14 you said:          │
│ │                             │  "We are working on this    │
│ └─────────────────────────────┘   since two weeks."         │
│ ▁▃▅▂▇▃▁▂▅▃▁▇▅▂▃▁  timeline      Better: "We've been working │
│   ●   ●  ●     ●   markers        on this for two weeks."   │
│                                   [Replay] [Try it now]     │
│ Pace 142 wpm   Fillers 3.1/100   Eye contact 71%            │
│ Mastery: 3 of 4 criteria met                 [Retry]        │
└─────────────────────────────────────────────────────────────┘
```

### 6.6 Copy rules

- Plain, specific, active voice. Buttons say what happens: "Start lesson", "End session", "Try it now".
- Insight language is factual and kind: what happened, the better version, why it matters. No "Great job!", no scores without an explanation.
- Empty and error states tell the user what to do next ("Microphone access is blocked. Allow it in your browser's site settings, then reload.").

---

## 7. Data model

| Table | Key fields |
|---|---|
| `track` | id, title, order |
| `unit` | id, track_id, title, order |
| `lesson` | id, unit_id, order, spec_json (the YAML spec, see appendix A) |
| `scenario` | id, title, persona, goal, voice_id (free-practice and lesson scenarios) |
| `session` | id, lesson_id?, scenario_id, started_at, duration_ms, video_path, calibration_json, wpm, filler_rate, eye_contact_pct, video_quality_ok, cost_usd |
| `turn` | id, session_id, speaker, start_ms, end_ms, audio_path, transcript_primary, transcript_secondary, words_json |
| `event` | id, session_id, turn_id?, type, start_ms, end_ms, payload_json |
| `candidate_finding` | id, session_id, turn_id, pattern_key, quote, suggestion, explanation, gate_results_json, dropped_reason? |
| `insight` | id, session_id, finding_id?, kind, category, rank, quote, clip_start_ms, clip_end_ms, better_version, why_it_matters, user_rating? |
| `retry_attempt` | id, insight_id, audio_path, transcript, improved_form_used, created_at |
| `lesson_attempt` | id, lesson_id, session_id, criteria_results_json, passed |
| `mistake_pattern` | id, pattern_key, category, times_seen, sessions_seen, last_seen, status (active, improving, mastered) |
| `api_cost` | id, session_id?, provider, model, units, unit_type, usd, created_at |

`event.type` values: `filler`, `long_pause`, `response_latency`, `eye_contact_drop`, `face_lost`, `target_phrase_used`, `insight`. New metrics add new types, not new columns.

---

## 8. Deterministic metric definitions

All computed in `api/app/core/metrics.py` with unit tests.

| Metric | Definition |
|---|---|
| Speaking rate | Words in the turn divided by voiced minutes (from VAD), reported per turn and per session. |
| Fillers | Count of unambiguous fillers from the primary transcript: `um, uh, er, ah, hmm` and the phrases `you know, I mean`. Reported per 100 words. |
| Hedge words | `basically, actually, like, kind of, sort of, just` counted separately and shown as information, never as errors, because they are often used correctly. |
| Long pauses | Silences longer than 1.5 s inside a user turn (not between turns). |
| Response latency | Time from the partner's audio ending to the user's voice starting. |
| Talk ratio | User voiced time divided by total voiced time. |
| Vocabulary range | Moving-average type-token ratio over 50-word windows (MATTR), so longer sessions aren't penalized. |
| Target phrase usage | Fuzzy match (token-level, threshold 0.85) of each lesson target phrase against user turns; each match becomes an event with its timestamp. |
| Eye contact | Share of the user's speaking time where head pose and iris position are within the calibrated "facing camera" range. Requires the 3-second calibration at session start, and is reported only when `video_quality_ok` is true. |

---

## 9. Build phases

### Phase 0: Foundation and design system

**Build:** the monorepo, Next.js app with Geist fonts and `tokens.css`, restyled shadcn primitives (button, input, dialog, tabs, tooltip, toast, progress), the app shell (sidebar with Learn, Progress, Settings), a `/design` page showing every token and component, and the FastAPI skeleton with SQLite, Alembic, a health route, and `.env.example`.

**Acceptance criteria:**
- `pnpm dev` starts web and API; the web app shows API health status.
- `/design` renders all colors, type sizes, and components in their states (default, hover, focus, disabled).
- Every interactive element has a visible keyboard focus ring; tab order is logical.
- No colors outside `tokens.css` are used anywhere.

### Phase 1: Voice core (backend only)

**Build:** the interfaces from section 3; Sarvam and faster-whisper transcribers; Claude Haiku partner with streaming and prompt caching; Bulbul and Azure voices; the cost logger; a CLI: `uv run python -m app.cli turn path/to/clip.wav --scenario status-update`, which prints both transcripts, streams the partner reply, and saves reply audio.

**Acceptance criteria:**
- The CLI works end to end on three recorded fixture clips, including one with obvious fillers.
- Sarvam verbatim output keeps fillers that faster-whisper may drop; a test documents the difference.
- Swapping the voice provider is a one-line config change.
- Every API call writes an `api_cost` row.

### Phase 2: Live session (voice only)

**Build:** the live session screen with the voice ring; hold-Space push-to-talk (and a pointer-hold button for mouse users); `POST /sessions`, `POST /sessions/{id}/turns` (audio in; SSE text and audio chunks out), `POST /sessions/{id}/end`. Pipeline the reply: split the streamed partner text at sentence boundaries and synthesize each sentence as soon as it's complete. Add three free-practice scenarios. Add a developer overlay (toggle with `?debug=1`) showing per-turn latency broken down by stage.

**Acceptance criteria:**
- Median time from releasing Space to hearing the partner is under 2.5 s over 10 turns, as shown in the debug overlay.
- The partner never corrects the user during the conversation.
- Turns longer than 30 s are handled without errors.
- Mic permission denied, network failure, and provider errors all show clear, actionable messages.

### Phase 3: Capture and deterministic metrics

**Build:** full-session webcam and mic recording (uploaded in chunks, saved locally); a 3-second "look at the camera" calibration before each session; MediaPipe Face Landmarker running in a Web Worker, sending per-second visual summaries (never raw frames) to the API; Silero VAD on the backend; every metric in section 8; the event log; session summary fields.

**Acceptance criteria:**
- Unit tests cover every metric with hand-labeled fixtures (for example, a clip with exactly 4 fillers and 2 pauses over 1.5 s).
- With the face out of frame for over 10% of speaking time, `video_quality_ok` is false and no visual metric is shown.
- MediaPipe runs at a steady frame rate without making the voice ring stutter.
- Session summary numbers match a manual recount on one real session.

### Phase 4: Insight engine and review

**Build:**
1. **Alignment:** align primary and secondary transcripts per turn (word-level, normalized: lowercase, stripped punctuation, numbers spelled consistently).
2. **Findings:** one Sonnet 5 call per session with the timeline, metrics, lesson spec, and the pattern taxonomy from `curriculum/patterns.yaml`. Output must match the JSON schema in appendix B.
3. **Gates, in order:** schema valid → quote exists in the turn → transcripts agree over the quoted span → verifier accepts → for trend claims, at least 3 sessions of data. Record every gate result on `candidate_finding`.
4. **Ranking:** `score = occurrences_this_session × severity (1–3, from verifier) × (1.5 if the pattern appeared in 2 or more of the last 5 sessions else 1)`. Metric-based insights (for example, filler rate above the user's baseline by 25% or more) compete in the same ranking with severity 2.
5. **Review screen:** video player, evidence timeline, top 3 insight cards, collapsed details, session metrics, "Accurate" and "Not accurate" buttons, and a debug panel listing dropped findings with reasons.

**Acceptance criteria:**
- A test injects a fabricated finding with a quote that isn't in the transcript; it is dropped with reason `quote_not_found`.
- A test with deliberately disagreeing transcripts drops the finding with reason `transcripts_disagree`.
- Every card's Replay button plays the exact moment it describes (clip boundaries padded 1 s on each side).
- No card contains a number that didn't come from `metrics.py`.

### Phase 5: Curriculum

**Build:** the lesson spec loader and validator; seed content for the Workplace basics and Meetings tracks (draft at least 3 units with 3 lessons each and show them to me for review before finalizing); the placement session (a 5-minute general conversation that sets the starting level and baselines); the learning path UI; the Learn step (target phrases, a short explanation, a model answer played by the partner voice); the partner's hidden rubric (create openings for target language without naming it); the Retry flow (replay the partner's question from the insight's moment, record a new answer, analyze it with the same gates, show whether the improved form was used); mastery evaluation against the lesson's criteria; unlocking.

**Acceptance criteria:**
- Invalid lesson YAML fails validation with a clear message naming the field.
- In a test lesson, the partner creates an opening for each target phrase within the conversation.
- Mastery results show each criterion with its evidence (a timestamp or metric value).
- A failed lesson queues a review lesson built from the session's top patterns.

### Phase 6: Skill profile and progress

**Build:** mistake bank updates after each session (a pattern becomes `improving` after 2 consecutive eligible sessions without it, and `mastered` after 4; "eligible" means the partner created an opportunity for it); the review lesson generator; the Progress dashboard (per-pattern history, delivery metrics over time, lessons completed) that shows trends only with at least 3 sessions; a cost view by day and provider; an eval harness, `uv run python -m app.evals run`, that replays saved sessions whose insights were rated and reports precision per insight category.

**Acceptance criteria:**
- With fewer than 3 sessions, the dashboard shows "Not enough sessions yet" instead of a trend line.
- The eval harness produces a precision table and fails loudly if precision drops versus the last saved run.
- Pattern status transitions are unit tested.

### Phase 7: Polish

**Build:** first-run onboarding (mic and camera permissions, voice accent preference, camera placement tips, calibration), settings (voices, models, recording retention), keyboard shortcuts help, empty and error states for every screen, data export and delete, and a performance pass.

**Acceptance criteria:**
- A fresh install reaches the first lesson in under 3 minutes.
- Deleting a session removes its database rows and its recordings folder.
- Lighthouse accessibility score of 95 or higher on the main screens.

---

## 10. Prompt skeletons

Store these in `api/app/prompts/` and iterate on them with the eval harness.

### Partner (`partner.md`)

```
You are {persona}, in this situation: {scenario}. Stay in character for the whole conversation.
Speak like a real person in a workplace: short turns (1–3 sentences), natural follow-up questions,
occasional mild pressure appropriate to the role.

Never correct the user's English, never comment on how they speak, and never mention that this
is practice. Corrections happen later, outside the conversation.

Hidden goals (never reveal them):
- Steer toward the lesson objective: {objective}
- Create natural openings where the user would need these phrases: {target_phrases}
- Also create openings for these patterns the user is practicing: {review_patterns}

If the user goes off topic, respond briefly and steer back the way {persona} naturally would.
```

### Findings (`findings.md`)

```
You are reviewing a spoken practice session for a professional English learner.
You receive: the lesson spec, the turn-by-turn transcript with turn_ids (verbatim, including
fillers), deterministic metrics, and a pattern taxonomy.

Propose findings about language (grammar, word choice, phrasing, clarity, structure) only.
Rules:
- Each finding must include the exact words the user said in "quote", copied character for
  character from one turn, and that turn's "turn_id".
- Use a pattern_key from the taxonomy. If none fits, use "other:<short-slug>".
- Judge spoken English, not written English. Fragments, contractions, and informal but clear
  phrasing are fine.
- Do not invent or restate numbers. Metrics are for context only.
- Also report evidence-based strengths: moments where the user used a target phrase or a
  practiced pattern correctly, with the quote.
- Return JSON only, matching the schema. If there is nothing worth reporting, return an
  empty list.
```

### Verifier (`verifier.md`)

```
You are a strict second reviewer. For the finding below, decide whether it is a real, useful
issue in natural spoken professional English, given the surrounding turns.

Reject if: the original is acceptable in speech, the suggestion changes the meaning, the issue
is purely stylistic preference, or the quote alone doesn't show the problem.
If the usage is standard in Indian English but may confuse international listeners, accept it
with label "regional".

Return JSON: {"accept": bool, "severity": 1|2|3, "label": "error"|"regional"|"style",
"reason": "<one sentence>"}
```

---

## Appendix A: Lesson spec format

```yaml
id: meetings-u1-l2
track: meetings
unit: 1
title: Giving a status update
level: B1
objective: Report progress clearly, starting with the headline
learn:
  explanation: >
    Lead with the overall status in one sentence, then give details.
    Use the present perfect for work finished recently.
  model_answer: >
    Overall we're on track. We've completed the database migration,
    and the main blocker is the API rate limit. By Friday, we'll have
    the fix deployed.
target_language:
  - "We've completed..."
  - "The main blocker is..."
  - "By Friday, we'll have..."
scenario:
  persona: Priya, the user's engineering manager. Busy, direct, asks follow-up questions.
  situation: Monday stand-up; Priya wants a quick update on the migration project.
  voice_id: bulbul-indian-english-female
  duration_minutes: 6
mastery_criteria:
  - type: target_phrases_used
    min: 2
  - type: llm_judgment            # must return a supporting quote, gated like findings
    question: Did the user state the overall status before giving details?
  - type: metric_vs_baseline
    metric: filler_rate
    max_ratio: 0.8
  - type: pattern_absent
    pattern_key: tense.present_perfect_recent_past
```

## Appendix B: Findings JSON schema

```json
{
  "findings": [
    {
      "turn_id": 12,
      "quote": "We are working on this since two weeks",
      "pattern_key": "tense.present_perfect_continuous_duration",
      "category": "grammar",
      "better_version": "We've been working on this for two weeks",
      "why_it_matters": "Present perfect continuous with 'for' is the standard way to describe something that started in the past and is still ongoing.",
      "is_strength": false
    }
  ]
}
```

## Appendix C: Starter pattern taxonomy (`curriculum/patterns.yaml`)

Start with these keys and let the agent propose additions from `other:*` findings for review:
`tense.present_perfect_recent_past`, `tense.present_perfect_continuous_duration`, `tense.past_simple_vs_present_perfect`, `article.missing`, `article.unnecessary`, `preposition.redundant` (for example "discuss about"), `preposition.wrong`, `agreement.subject_verb`, `word_choice.false_friend`, `word_choice.overly_formal`, `phrasing.indirect_request`, `structure.buried_headline`, `structure.rambling_answer`, `clarity.vague_reference`, `regional.indian_english`.

---

## Agent skills

### Issue tracker

Issues live in GitHub Issues (via `gh`); external PRs are not a triage surface. See `docs/agents/issue-tracker.md`.

### Triage labels

Default vocabulary: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `CONTEXT.md` and `docs/adr/` at the repo root. See `docs/agents/domain.md`.
