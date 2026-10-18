# Synthesis: memory bottlenecks, how to counter them, and a design direction for "infinity"

Written 2026-10-18. This pulls together docs 01-05. Where a claim comes from a specific source the
doc number and section are given, e.g. (05 §A1). Numbers are the sources' own; see those docs for caveats.

## 0. The short answer

The field has converged on one hard lesson and one open problem.

The hard lesson: **compression is not memory.** Every system that replaced the raw conversation
with summaries or extracted facts lost 15-22 points against keeping verbatim text (02 §13:
Fidelity Before Structure, MemDelta, SUMER), and summaries compound their losses with every round
(05 §A6: 53% of constraints survive one Claude Code compaction, 10% survive five). The winners in
2026, from Taelin's UniiChat to OpenHands to Karpathy's wiki, all keep an immutable raw log and
treat everything else as a rebuildable index on top of it.

The open problem: **state, not recall.** Benchmarks now agree the weakest capability across every
memory product is handling facts that change (02 §13: MemConflict, BEAM, Memora, "When Does Memory
Help"). Users say the same thing in plain words: "it's not because it forgets the facts, it's
because the invalidation doesn't propagate" (04 pain point #2). Mem0 gave up on inline updates;
OpenAI, Letta, Mem0 and OpenClaw all moved conflict resolution to a nightly "dream" pass (03 §0).
Nobody has a clean in-band answer.

For a personal agent that must know "my whole life, behaviour, moods", the design that follows
from the evidence is: a lossless temporal log with a bounded multi-resolution view (UniiChat's
backbone), plus a small typed **current-state ledger** with validity intervals and evidence
pointers (the thing UniiChat lacks), plus agent-navigated retrieval that is never auto-injected,
plus an evidence-gated affect track kept out of the persona. Section 3 spells it out.

## 1. The bottlenecks, ranked, each with its evidence-backed counter

### 1.1 Long context is not usable context
- **Evidence.** Latent-association recall (the kind "how did I feel last Tuesday" needs) has an
  effective length of 2-16K tokens even on frontier models: GPT-4.1 drops from 97 to 65 at 128K on
  NoLiMa (05 §A1). Failure rises with the *number* of relevant items more than with length (2 needles
  35% failure, 8 needles 61%). Distractors actively hurt; coherent irrelevant prose hurts more than
  noise (Chroma). A 500K-token endless chat costs about $2 per uncached turn on Opus 5.5 and every
  idle gap over 5 minutes re-pays the cache write (05 §A2). Consumer Gemini users measured ~16K of
  effective conversational memory against the advertised 1M (04 #8).
- **Counter.** Keep the part of the context the model must *reason over* small and focused. The
  bounded view (64-128 KB, recent fine, old coarse) with zoom is the right shape: the whole life is
  *addressable* without being *loaded*. Treat the context window as a cache, not the store.

### 1.2 Compaction is lossy, compounding, and decided before the question is known
- **Evidence.** Compaction Cliff, TRACE, KVMem, the rate-distortion paper (05 §A6); Claude Code
  issue #13112 closed "not planned"; "once it gets confused, start over" (04 #3). LLMLingua-2 kept
  21% of rules and deleted a negation.
- **Counter.** Never compact the ground truth. Summaries are an index you can zoom through, not a
  replacement. Route typed facts (preferences, constraints, life-state) through an exact-preservation
  path, not through a summariser (05 B10: typed compaction scored 1.00 where LLM compactors scored
  0.45-0.58). UniiChat's compaction prompt priorities (user's words near-verbatim > lasting effects
  and failures > findings > tool steps) are the best-tested ordering found (01 §2.6, 01 §3).

### 1.3 Cost is prefill, and prefill is only affordable when cached
- **Evidence.** Agents run at ~100:1 input:output (Manus). Cached reads are 0.1x, 0.05x on Opus/
  Sonnet 5.5, 0.025x on Fable 5.1; writes 1.25x; TTL 5 minutes (05 §A2). The original OptChat
  "trashed the prompt cache"; batching merges took rewritten lines per turn from ~147 to ~2 and
  cache reads from 58-76% to 97% (01 §4.3).
- **Counter.** Byte-stability of the prefix is the design constraint. Append-only growth; batched
  rewrites (sawtooth); nothing volatile (dates, device state, recalled snippets) before the stable
  block; the compactor shares the main system prompt so it hits the same cache. Order the prompt by
  volatility, ascending (see 3.6).

### 1.4 Retrieval fails on exactly the questions a life-agent gets
- **Evidence.** Embeddings ignore negation and time; multi-hop and multi-session are the hardest
  category everywhere (Mem0 multi-hop 51; Mastra ~87% ceiling); memory products' retrieval precision
  clusters at 0.22; a plain BM25 baseline beat several products (05 §A4, 04 debate section).
  Timestamps alone moved temporal accuracy from 22 to 55 (Mem0 vs OpenAI memory).
- **Counter.** Time-first filtering, then hybrid lexical+dense, then rerank (05 B7 ranking).
  Exact-name lexical search matters for people and places. Most importantly: let the *main model*
  navigate (zoom, recall with a time range) instead of a retriever guessing top-k and stuffing it
  in. Every UniiChat reimplementation added a search tool to the pure zoom design (01 §4.4); do the
  same, but keep it a tool, not an injection.

### 1.5 Facts change and nothing propagates the change
- **Evidence.** "Rendering superseded edges puts stale values back into the answerer's view"
  (Graphiti scored 7.0 on knowledge-update for exactly this; 05 §A5). Mem0's additive design
  regressed on updates. ChatGPT congratulates people on jobs they lost (04 #9). Agents act on a
  correctly retrieved updated value only 55% of the time (02 §13).
- **Counter.** Append-only event log plus a *derived* current-state view. Facts carry valid-from /
  valid-to and are never deleted, but **only current facts are rendered**; superseded ones are
  reachable by zoom. "The view's latest word on a thing is the truth" (UniiChat) is the right
  default rule; a typed ledger makes it mechanical rather than hoping the model notices.

### 1.6 Automatic extraction produces junk and feeds on itself
- **Evidence.** 97.8% of 10,134 Mem0 memories were junk in a 32-day audit; 808 copies of "user
  prefers Vim" from re-ingesting recalled memories (03 §5). Bee stored a TV plot as a fact about
  its wearer (04 mood section).
- **Counter.** Tag every injected memory with provenance (`recalled`, `imported`, `tool`) and make
  the extractor/compactor skip those tags (OpenClaw does this; Mem0 did not). Weight the user's own
  words above everything inferred. Treat casual remarks as evidence, not as preferences, until
  repeated (life-OS users converged on "three repeats before it counts", 04 DIY #4).

### 1.7 Hallucinated summaries become permanent
- **Evidence.** Haiku-written summaries "invented facts" (blind grades 40-62 vs 73-76 for Sonnet);
  older stored summaries had the most unsupported claims (01 §3, §4.1). A lifelogger measured ~10%
  hallucinated sentences including a false fact about a family member's health (04 mood section).
- **Counter.** Use a strong compactor for the leaves (this is where integrity is decided); run a
  blind audit that counts unsupported claims per line against the input; make "unsupported claims
  per 100 lines" a tracked metric. Require evidence pointers (message ids) on every ledger fact.

### 1.8 Bleed-through, dossiers, and creepiness
- **Evidence.** The #1 user complaint (04 #1, #4, #7): Half Moon Bay signs, Pokemon-card grandma,
  merged sibling profiles, opaque summaries, opt-outs that don't stick. The Personalization Trap
  study found injected profiles *lowered* emotional-reasoning accuracy in 11/15 models (02 §14).
- **Counter.** Scope facts and episodes by life-area; make retrieval visible as tool calls (Willison
  prefers Claude's design for exactly this); provide an editable ledger with version history and a
  real delete; offer a per-message "fresh mode" that drops the ledger and recall. Do not inject a
  persona paragraph into every turn.

### 1.9 Memory can make the model worse: creativity, intent dilution, sycophancy
- **Evidence.** "Dilution of intent"; OpenAI's own postmortem that memory "in some cases"
  exacerbated sycophancy; delusional-spiral concerns (04 #5, #6).
- **Counter.** Small always-on block (Claude Code caps its index at 25 KB; OpenClaw has a bootstrap
  budget); everything else on demand. Keep user corrections and the agent's own failures visible in
  the view (Manus: keep failures) so the model does not drift into flattery.

### 1.10 Mood and affect memory is unvalidated and risky
- **Evidence.** No affect-memory paper with empirical results was found (02 §14); the one rigorous
  study shows profiles hurt emotional reasoning; practitioners who tried it disabled cross-session
  memory after hallucinations caused "whiplash and distrust" and limited reassurance-seeking (04 mood
  section). What worked: evidence-gated claims ("two dated quotes per claim or drop it", gaps shown
  as gaps, not estimates).
- **Counter.** Keep affect as a separate, evidence-gated time series with explicit abstention,
  derived only from the user's own words, surfaced by a tool or on request, never baked into the
  system prompt as "the user is anxious".

## 2. What people are searching for and thinking (the zeitgeist, condensed)

- **"The context window is a cache, not memory."** This phrase, in several forms, is the 2026
  consensus (Hindsight, swyx, Karpathy's KV-cache-as-working-memory). Nobody believes 1M-10M windows
  make memory systems obsolete; swyx: "Gemini has had a million token context for two years and
  no one's using it."
- **Trust, not recall, is the unsolved problem** (Ajay Kumar's Oct 2026 review). People want to see
  what was stored, why, and be able to delete it.
- **Markdown files plus grep beat vector databases** for personal memory in practitioners' hands
  (Karpathy's LLM wiki ~5k stars in days; Jason Liu's vault; Ask HN threads). The reason given:
  diffs become a review surface, and "memory is text, text needs management".
- **Local-first is a demand, not a preference.** Lock-in and the Limitless/Meta shutdown made
  "the cloud is just LLM request fulfilment" a recurring line.
- **The whole-life-in-one-log dream is explicit** at the top (Altman's trillion-token life context)
  and the bottom (Taelin: "a single unified log of my entire life"; Wooders: "perpetual without
  being interrupted by compaction"). UniiChat is the closest existing artifact to what you want.
- **Memory is the moat / the personalization turn is coming** (swyx, Chase, Wooders), paired with
  the fear that a lifelong memory "turns knowledge into leverage" (Tech Policy Press) and collapses
  context boundaries into "the entire mosaic of people's lives" (CDT).
- **Rising queries** (inferred from SEO clusters, not measured): "ChatGPT memory not working / full
  / turn off", "AI with memory", "personal AI agent memory", "second brain AI".

## 3. A design direction for infinity

The backbone is UniiChat (01). The additions address the gaps listed in 01 §6 using the mechanisms
with the best evidence from 02, 03 and 05. Everything lives on disk; the model is a service.

### 3.1 Layer 0: the log (lossless, append-only)
Exactly UniiChat §1: `{i, kind, text, size, date}` per message, kinds `user | agent | tool | echo |
work | note`, plus two additions:
- `provenance` on every message: `live | imported | recalled | consolidated`. The compactor and
  consolidator never learn from `recalled` (feedback-loop fix) and discount `imported`.
- `scope` tags (life-areas, free-form, e.g. `work`, `health`, `family`) set by the compactor at
  leaf time. Cheap, and they make the fresh-mode and bleed-through controls possible.
Redaction is the one exception to append-only: a `redact(i)` operation blanks a message and marks
every tree node covering it as dirty for rebuild. This is the "hard delete" users ask for; accept
the one-time cache miss.

### 3.2 Layer 1: the temporal tree and the view (UniiChat §2-4, unchanged)
512-byte nodes, binary merges, `due = (T - last) / 2^l`, sawtooth 128 KB to 64 KB, persisted view,
zoom and date tools. Use the revised compaction prompt verbatim (it won the blind study). Two
choices the reimplementations settled: keep the previous exchange verbatim (<= 16 KB) rather than
summary-only, and use a Sonnet-class compactor for leaves, not Haiku.

### 3.3 Layer 2: the ledger (typed current state, the main novelty over UniiChat)
A small document, budgeted (8-16 KB), always in context, holding only *current* typed facts:
```
people:   Priya | partner | since 2024-03 | ev: 8812, 14020
work:     Infinity (personal agent) | active project | since 2026-10-04 | ev: 15011
health:   sleep: poor on Sundays (pattern, 4 obs) | ev: 12004, 12981, 13877, 14960
prefs:    replies: terse, no em-dashes | ev: 9120
goals:    ship infinity MVP by 2026-11 | open | ev: 15011
```
Rules:
- Every fact has `valid_from`, optional `valid_to`, a confidence tier (`stated | repeated | inferred`),
  and evidence message ids. Stated-by-user outranks inferred, always.
- Superseded facts get `valid_to` and drop out of the rendered ledger; they stay in history and
  are reachable by `ledger_history(key)`. Never render a superseded value (the Graphiti lesson).
- **Ledger changes are themselves log messages** (kind `ledger`, provenance `consolidated`). So the
  tree summarises *why* a fact changed, the agent can zoom into it, and the ledger is fully
  reconstructible from the log. Diffs are the review surface for the user.
- Inferred patterns need repetition before promotion (three independent observations), and the
  agent must be able to answer "why do you think that?" with the evidence ids.
This is User-as-Code (typed state) + Zep's bi-temporal validity + Letta's core block + Karpathy's
lint, reduced to one small table.

### 3.4 Layer 3: recall (agent-navigated, never auto-injected)
An index over leaf lines and raw messages: SQLite FTS (exact names, dates) + embeddings + time
filter + scope filter, with a cheap reranker. Exposed as `recall(question, from, to, scope)`
returning ranked passages with message ids. Not injected per turn. The system prompt tells the
model: find the latest mention in the view first, zoom, then recall if the view is silent. This
keeps UniiChat's "latest word is truth" discipline while fixing its cross-cutting-question weakness.

### 3.5 Layer 4: the affect track (separate, evidence-gated, abstaining)
Per day (or per session) a tiny record built by the compactor from the user's own words only:
`{date, valence: -1..1 | null, energy: low|mid|high | null, themes: [...], evidence: [ids]}`.
`null` is allowed and preferred over guessing. Exposed as `mood(from, to)`; never placed in the
system prompt; the agent may consult it when the user asks or when a decision plausibly depends on
it. Trend questions ("how has my sleep been since June?") are answered by this track plus the
ledger, not by zooming through months of tree.

### 3.6 Prompt layout, ordered by volatility
```
[tools] [system prompt] [view] [ledger] [previous exchange verbatim] [turn state: date, scope] [message]
```
The view is large and changes only at its end between batches, so it goes first after the static
prefix. The ledger is small and changes a few times a day, so it goes *after* the view: a ledger
edit then invalidates only ~10 KB of cache, not ~100 KB. Compactions reuse `[tools][system]` so
they hit the same cache.

### 3.7 The sleep pass (offline consolidation, audited)
Runs when idle, with a strong model:
1. Lint the ledger against new leaves: contradictions, stale `valid_to` candidates, promotions.
2. Blind-audit recent summary lines for unsupported claims; flag, don't silently rewrite.
3. Write structured weekly chapters (events, decisions, people, open loops, affect summary),
   stored as `note` messages so they enter the tree too. Structured beats prose for recall (05 B10).
4. Emit a diary of what changed, for the user to review as a diff.
Nothing in the sleep pass deletes; it appends ledger events.

### 3.8 Controls users actually ask for
Visible memory tool calls; `/fresh` for a turn without ledger or recall; scope on or off per
life-area; an editable ledger with history; `redact`; export everything (it is already JSONL);
local storage with the model as a request service.

### 3.9 Evaluate against your own log
You hold the ground truth, so generate your own LongMemEval: sample messages, hide the future,
auto-write questions (single-hop, temporal, knowledge-update, multi-session, abstention, affect),
and score the whole stack against a full-context baseline on the same questions. Track: accuracy
by question type, unsupported-claims rate in summaries, cache-read percentage, cost per 1,000
messages. Treat vendor LoCoMo numbers as incomparable (02 §13).

## 4. Novel directions worth an experiment (ranked by promise over risk)

1. **Ledger events in the log.** Memory updates as first-class messages so the temporal tree
   explains the ledger. Not seen anywhere; cheap; directly attacks "invalidation doesn't propagate".
2. **Volatility-ordered prefix.** A general rule for cache-first harnesses; a measurable win.
3. **Three-axis memory mapped to question types.** When (tree), what-is-true-now (ledger),
   who/what (recall index). Route each benchmark question class to an axis; this gives a principled
   reason for each component and a way to see which axis fails.
4. **Evidence-gated affect with abstention.** The only version of mood memory with any practitioner
   evidence behind it; the research literature has nothing validated.
5. **Compactor audit as a first-class metric.** pi-optchat's blind "unsupported claims" count,
   run continuously on the tree.
6. **Importance-aware merge ordering.** UniiChat merges purely by age. Among pairs that are equally
   due at batch time, prefer merging low-importance pairs first (Generative Agents' importance
   score), so important weeks stay fine-grained longer. Keep it within batches so the cache
   discipline survives. Untested; measure against the pure-age schedule.
7. **Self-generated evals from the user's own log** (3.9). The one evaluation that is not gameable
   by vendors.
8. **Later, on open weights only:** Cartridges / Still style compressed KV for the frozen archive
   (months-old history), leaving the live tree for the recent past (05 B3). Not reachable via APIs.

## 5. What to build first
1. Log + tree + view + zoom + date, cache-aware, with the revised compaction prompt. Replay your
   exported chats and measure cache reads and cost (targets: >95% cache reads, ~$8 per 1,000
   messages on Haiku-class compaction, more on Sonnet).
2. The ledger with validity intervals, evidence ids, and ledger events in the log.
3. `recall` with FTS + time + scope filters; keep it a tool.
4. The affect track with abstention; a `mood` tool.
5. The sleep pass with the blind audit and the change diary.
6. Your own eval set from your own log, run after each step.

## 6. Open questions the research could not settle
- Whether a pure zoom-navigated tree is enough for cross-cutting questions at the scale of years,
  or whether `recall` ends up doing most of the work. Measure on your own log.
- How much a strong compactor costs over a year of heavy use; the Sonnet-vs-Haiku quality gap is
  large but so is the price gap.
- Whether narrative chapters add anything over structured ones. The only evidence points to
  structured; no controlled study exists.
- The right promotion threshold for inferred patterns (three repeats is folk wisdom, not measured).
- Anything about affect memory beyond "gate it with evidence and allow abstention".
