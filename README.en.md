# Companion AI Memory Context and Token Optimization

**中文：[陪伴 AI 记忆上下文 token优化](README.md)**

I recently came across Da Ha Yu ([Coopanion](https://github.com/Pal-AI-Lab/Coopanion)), an AI companion project struggling with the limits of a long conversation. One comment stayed with me: “爸爸想给你完整的生命” — “Dad wants to give you a complete life.” It made me want an AI companion to keep the moments you've shared, without carrying every old chat into every new reply.

So I put together this plan: save the full conversation in a cold archive, keep the things you talk about often close at hand, and find older moments when you give a clue — “Remember that time in Xi'an?” That should use fewer tokens while holding on to the past. The prompts and made-up examples are here for anyone who wants to try them and help test the idea. It's still a proposal, not a feature running in Coopanion; the memory ideas build on earlier work, including MemGPT, MemoryBank, MemoryOS, and Zep, compared in [References](REFERENCES.en.md).

## The memory model

The design keeps every turn of the conversation, from both sides, in a host-managed cold archive unless the user asks to delete it. Each model request gets only a small recent window and a few relevant notes. “Forgetting” means an old chat is out of sight for now; the user can bring it back with a clue.

| Area | Contents | When used |
| --- | --- | --- |
| Recent window | Recent exchanges and the current task's required state | Bounded continuity for the current topic |
| Working memory | A few explicitly pinned facts and relevant, recent or repeated user topics | Selected by relevance and strict item/token limits |
| Cold archive | The full conversation and searchable notes about older moments | Searched only after a current user cue |
| Temporary recall | A few cold records matched to the cue | Expires when the topic or authorization ends |

Only verified, substantive user mentions increase a memory's frequency. An assistant's restatement, a tool read, a timer, and a retry do not. A single recall does not promote a cold record to working memory. Corrections, explicit pinning, “only mention this when I ask,” and deletion have distinct behavior.

## Old memories have multiple index entries

The host indexes cold records by **type, event time, place, person, object, and topic**. One record is stored once, with several ways to find it. The full archive directory is never sent to the conversation model.

For example, “Do you remember that time in Xi'an years ago?” supplies a place cue but no exact year. The host first searches the Xi'an place index. If several events match, it can return up to three very short cards and ask, “The camera purchase or the photography exhibition?” “The camera purchase” then narrows the search by place, object, and event type. Candidate cards and final excerpts share the same recall allowance. See [Cold archive indexing](docs/cold-storage-index.en.md) and the [synthetic search plan](examples/recall-plan.en.json).

## Initial limits

These are illustrative parameters to test, not existing Coopanion settings or proven optimal values.

| Component | Per-request limit |
| --- | ---: |
| Entire serialized memory snapshot | 1,600 tokens |
| Within it: pinned | 160 tokens / 4 records |
| Within it: relevant working memories | 960 tokens / 8 records; at most 2 per topic |
| Within it: cue-based recall | 480 tokens / 3 cards or records |
| Recent conversation window, counted separately | 2,000 tokens / up to 6 complete interactions |

Unused allowances stay unused. The system prompt, tool definitions, current user message, tool transaction, output reserve, and protocol overhead count toward the full request separately. A model-specific token counter is required to verify a hard token limit.

## How many tokens could it save?

We don't have a real number yet. As of 2026-10-25, this hasn't been built into Coopanion and measured, so I wouldn't call any percentage “tested savings.”

For a rough example, imagine one request carries 8,000 tokens of fixed instructions and tools, 20,000 of old conversation, and 2,000 for the current message: **30,000 input tokens**. Keep the same fixed part and current message, but use 2,000 tokens of recent chat and a 1,600-token memory snapshot: **13,600 input tokens**. That is `(30,000 − 13,600) / 30,000 ≈ 54.7%`, or **about 55% fewer input tokens in this made-up case**. It says nothing yet about the real app's bill or speed.

To find the real number, we'd also count cached input, extra tool calls, model outputs, and the work of finding and updating memories. Then we'd compare cost, response time, and whether it remembers the right things. The [evaluation plan](docs/evaluation.en.md) spells out that test.

## How to use this package

1. Read the [memory lifecycle](docs/design.en.md) and [cold index](docs/cold-storage-index.en.md).
2. Follow the [Coopanion integration design](docs/coopanion-integration.en.md) to implement host-controlled storage, user-cue permission, budget checks, and the actual outbound context view.
3. Append the [English conversation policy](prompts/companion-policy.en.md) to the existing persona only after the host can supply a bounded memory snapshot. The [Chinese policy](prompts/companion-policy.zh.md) is an alternative, not an additional prompt to send every turn.
4. Use the [incremental extractor](prompts/memory-extractor.en.md) for proposed updates. Use the [cue router](prompts/cue-router.en.md) only when simpler rules cannot resolve a cue.
5. Check the [example policy](config/policy.example.json), [three-conversation archive sample](examples/dialogue-archive.en.json), [synthetic records](examples/memories.en.json), [31 acceptance scenarios](examples/scenarios.en.json), and [evaluation plan](docs/evaluation.en.md).

A prompt alone cannot reduce input already sent to the model. The host must enforce the storage and retrieval boundary before each request, including old session history, handoff notes, and tool outputs.

## Files

| Area | Contents |
| --- | --- |
| [Design](docs/design.en.md) | Record states, temperature, cue gate, budgets, updates, deletion |
| [Cold index](docs/cold-storage-index.en.md) | Multi-facet partitioning and progressive retrieval |
| [Integration](docs/coopanion-integration.en.md) | Pinned upstream source analysis and implementation boundaries |
| [Evaluation](docs/evaluation.en.md) | Acceptance conditions, cost and quality measurements |
| [Prompts](prompts/README.en.md) | Prompt roles and contracts |
| [References](REFERENCES.en.md) | Prior research, official projects, and differences |

The [conversation archive sample](examples/dialogue-archive.en.json) shows three complete synthetic exchanges, including ordinary chat that never became a memory card. It is not the source archive for all seven example cards. The JSON examples are available in [English](examples/scenarios.en.json) and [Chinese](examples/scenarios.json). They are specifications, not results from a running model.

## Implementation status

- [x] Design, prompts, synthetic records, and acceptance scenarios.
- [ ] Persistent cold archive with provenance and user-mention accounting.
- [ ] Coopanion/Cortico outbound context filtering and protection against generic-tool bypass.
- [ ] Verified token budgeting that preserves complete active tool transactions.
- [ ] Controlled A/B replay and publication of measured cost, cache, latency, and recall results.

This is an independent proposal for Coopanion, not an official Pal-AI-Lab feature or endorsement. Source analysis is pinned to the 2026-10-25 snapshots described in [References](REFERENCES.en.md); upstream changes require a new integration review. This package contains original documentation, prompts, and synthetic examples, not copied upstream source. Each upstream project retains its own license. The publisher should choose a distribution license before adding a LICENSE file.
