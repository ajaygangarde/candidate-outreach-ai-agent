# Multi-Channel Autonomous Candidate Outreach (WhatsApp + Voice AI)

Autonomous agents that reach job seekers on WhatsApp or by phone, run a structured screening
conversation, and write typed results back into the recruitment pipeline — with a human involved
only on explicit escalation.

> **This is the overview.** For the engineering deep dive — queue dispatch, retry policy,
> dead-letter handling and idempotency — see


---

## Problem

Before a recruiter talks to a candidate, someone has to collect five basics: current salary,
expected salary, notice period, location, and work status. Doing this by hand doesn't scale — and
"just plug in an AI" breaks in three ways:

**One channel isn't enough.** Some candidates never answer calls; others never reply to texts. You
need both voice and WhatsApp — but building them as separate systems doubles the work.

**Conversations don't become data on their own.** A great call still just leaves a sentence in a
notes field, not a searchable, filterable record.

**Partial replies get lost.** A candidate who answers 3 of 6 WhatsApp questions and goes quiet is
still valuable — but without tracking progress, that looks identical to never replying at all.

---

## Solution

**Same playbook, two channels.** Voice and WhatsApp run on the same underlying design — just
different scripts (voice: greet → ask → wrap up; WhatsApp: get consent → ask → wrap up). Adding a
third channel later means reusing this playbook, not building from scratch.

**Simple, event-driven logic** — no heavy framework. Each message or call turn is handled as it
comes in, triggered directly by the incoming webhook. No complex engine running in the background,
just a straightforward decision at each step.

**Memory that fits how each channel behaves.** A voice call is a live, ~30-second conversation, so
its state lives in memory for that call. WhatsApp replies can come hours later from anywhere, so its
state is saved in the database and picked up wherever it left off.

**Most replies are understood instantly, without AI.** Simple word-matching — including
Hindi-English mixed phrases like *haan*, *nahi*, *kitna* — handles the overwhelming majority of
replies immediately, for free. AI only steps in for the tricky ones.

**Voice never makes the candidate wait.** Every question is pre-recorded in advance, and the next
question is prepared in the background while the candidate is still answering the current one — so
there's no dead air.

**One AI call per conversation, not per message.** Instead of asking AI to think on every reply,
it's used once at the end — to read the whole conversation, pull out salary, notice period,
location, and save them directly to the system.

**Nothing gets lost.** Duplicate messages are ignored, failed sends are retried, and candidates who
go quiet get a nudge before their session is closed out — so partial answers are never thrown away.

**Built to grow.** Adding a new channel (like SMS) reuses the same core pieces.

---

## How it fits together

```mermaid
flowchart TB
  recruiter["Recruiter UI"] -->|enqueue| queue["SQS: outreach queue"]
  queue --> voiceWorker["Voice worker"]
  queue --> waWorker["WhatsApp worker"]

  voiceWorker --> plivo["Plivo"]
  plivo --> voiceAgent["Voice agent\nintro → question → wrapup"]
  voiceAgent --> cartesia["Cartesia TTS + Whisper STT"]

  waWorker --> meta["Meta Graph API"]
  meta --> waAgent["WhatsApp agent\nconsent → question → wrapup"]
  waAgent --> pg[("PostgreSQL\nsession state")]

  voiceAgent --> extract["LLM extraction\n(1 call / conversation)"]
  waAgent --> extract
  extract --> ats[("ATS tables\nstructured write-back")]

  abandonWorker["Abandonment worker\n18h nudge / 24h finalize"] --> pg
  ecs["ECS · PM2 processes"] --> voiceWorker
  ecs --> waWorker
  ecs --> abandonWorker
```

The recruiter's screen never waits on a phone call or a WhatsApp send. Work is handed to a queue and
returns immediately; the conversation happens in the background and results appear in the pipeline
when it completes.

**Stack:** Node/Express · Plivo (voice) · Meta Graph API (WhatsApp) · Cartesia TTS + Whisper STT ·
`gpt-4o-mini` with `gemini-2.5-flash` fallback · PostgreSQL · SQS · ECS/PM2.

---

## Result

- **End-to-end autonomy** — dial/message → converse → classify → extract → update the pipeline, with
  a human in the loop only on explicit escalation.
- **One contract, two channels** — a third channel is a config change, not a new system.
- **Structured data, not transcripts** — five fields populated automatically on every completed
  conversation.
- **Durable by construction** — conversations survive restarts, duplicate messages, and candidate
  abandonment without data loss.
- **Cost and latency bounded by design** — one AI call per conversation, not per turn; no waiting on
  speech generation during a live call.

---

## Extending it

- **Add a new question:** update the config file — no database changes needed.
- **Add a new channel (e.g. SMS):** reuse the same building blocks already used for voice and
  WhatsApp — no need to rebuild the system.
- **Teach it a new phrase:** add it to the keyword list. Anything unrecognized still falls back to
  AI, so nothing goes unanswered.

---

## What's next

- Move voice session state out of in-process memory to support horizontal scaling.
- Add real-time streaming voice to support barge-in (candidate interrupting mid-prompt).
- Promote screening questions from code config to a database table, so questions can be customized
  per job without a deploy.
- Add automated tests around the agent decision functions.
- Show WhatsApp delivery status per candidate, so a message that was never delivered is visible in
  the campaign screen instead of looking identical to one that was.
