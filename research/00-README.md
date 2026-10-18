# infinity — memory research pack (2026-10-18)

Research for a single endless personal-agent chat that remembers a whole life (behaviour, moods,
decisions, people). Goal of this pack: know the real bottlenecks, how people counter them, what
users want and fear, and where a novel design can add value.

## Reading order
| # | File | What it is | Words |
|---|---|---|---|
| 06 | `06-synthesis-and-design-directions.md` | **Start here.** Bottlenecks ranked with counters, the zeitgeist, a layered design for infinity, novel experiments, build order | ~3,000 |
| 01 | `01-uniichat-optchat-deep-dive.md` | Taelin's UniiChat/OptChat from primary sources: mechanism, numbers, community pitfalls, gaps | ~2,500 |
| 05 | `05-bottlenecks-and-compression.md` | Context rot curves, cost/caching economics, retrieval failures, compaction loss, every compression technique with a verdict table | ~5,000 |
| 02 | `02-academic-landscape.md` | 15 architectures (MemGPT to Hindsight), 12 benchmarks, surveys, affect/user-modelling papers, comparison table | ~4,400 |
| 03 | `03-production-systems.md` | How ChatGPT, Claude, Gemini, Letta, Mem0, Zep, Supermemory, Honcho, OpenClaw actually work, with complaints | ~4,300 |
| 04 | `04-community-pain-points.md` | Top-10 pain points with quotes, what people dream of, DIY patterns that work, mood-tracking attempts, notable quotes | ~4,700 |
| — | `sources/taelin-uniichat-optchat-gist-rev-2026-10-18.md` | Verbatim copy of the canonical spec | — |

## Ten headline findings
1. **Compression is not memory.** Keeping verbatim text beats extracted facts or summaries by 15-22 points in three independent 2026 studies. Every winning 2026 design keeps a lossless log and treats summaries as an index.
2. **Compaction compounds loss.** Claude Code's compaction keeps 53% of constraints after one round and 10% after five. Token pruners like LLMLingua-2 delete negations.
3. **Usable context is far smaller than advertised.** Latent-association recall has an effective length of 2-16K tokens on frontier models (NoLiMa). Item count and distractors hurt more than raw length.
4. **The economy is the prompt cache.** Agents run ~100:1 input:output; cached reads cost 0.025-0.1x. UniiChat's whole design exists to keep the prefix byte-stable (97-99% cache reads measured).
5. **State, not recall, is the unsolved problem.** Contradiction and knowledge-update handling is the weakest capability on every 2026 benchmark; rendering superseded facts leaks stale values back into answers.
6. **Automatic extraction produces junk** (97.8% of one Mem0 deployment's memories) and feeds on itself unless injected memory is tagged and excluded from re-ingestion.
7. **Summariser hallucination is permanent.** Haiku-class compactors invented facts (blind grades 40-62 vs 73-76 for Sonnet); audits for unsupported claims are the fix.
8. **Users' top complaints are bleed-through, staleness, opacity, and creepiness**, not lack of recall. They want scoping, visible retrieval, editable memory with history, hard delete, and local storage.
9. **Mood/affect memory has no validated technique.** Injected profiles lowered emotional-reasoning accuracy in 11/15 models. The only thing that worked in practice is evidence-gated claims with abstention.
10. **UniiChat is the closest existing artifact** to the goal: one endless chat, verbatim log, binary summary tree, 64-128 KB view, zoom. Its gaps are typed state, cross-cutting retrieval, consolidation, affect, and auditing.

## Caveats
Vendor benchmark numbers (Mem0, Zep, Hindsight, OpenAI) are not comparable with each other; LoCoMo itself has a flawed answer key. Reddit was unreachable, so Reddit sentiment is second-hand. X posts were read via syndication or search snippets. Each doc ends with its own unverified-items list.
