# Companion AI Memory Context and Token Optimization

**中文：[陪伴 AI 记忆上下文 token优化](README.md)**

This proposal draws on familiar aspects of human memory: recent and repeated information is easier to access, while older details can be recovered from a useful cue. It targets the growing input size, latency, and token consumption of long-running AI conversations. [Coopanion](https://github.com/Pal-AI-Lab/Coopanion) is the intended integration target: keep a small relevant working set, move infrequent memories into recoverable cold storage, and retrieve old details only from a user's cue.

This repository contains a design, prompts, example policy and acceptance scenarios. **Version 0.1 is a proposal and prompt package. It has not been integrated into Coopanion or measured in production.** The tiered-memory and forgetting concepts have prior art; see [References](REFERENCES.en.md).

## The memory model

Persistent storage may grow, but each model request has a firm context budget. “Forgetting” means an old detail is absent from the current request, not that its source record was destroyed or that the assistant should pretend to be human.

| Area | Contents | When used |
| --- | --- | --- |
| Recent window | Recent exchanges and the current task's required state | Bounded continuity for the current topic |
| Working memory | A few explicitly pinned facts and relevant, recent or repeated user topics | Selected by relevance and strict item/token limits |
| Cold archive | Infrequent or older records, with their source and metadata | Searched only after a current user cue |
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

## Estimated savings versus measured savings

**As of 2026-10-25, there is no Coopanion measurement for this proposal. No “measured savings up to X%” claim is supported.** A smaller input also does not imply an equal reduction in latency or total cost.

Here is a calculation example using *assumed* inputs: suppose a baseline request contains 8,000 tokens of fixed instructions and tool schemas, 20,000 of repeated history, and 2,000 for the current message: **30,000 input tokens**. Suppose the proposed request contains the same 8,000 fixed tokens, a 2,000-token recent window, a 1,600-token memory snapshot, and the same 2,000-token current message: **13,600 input tokens**. The illustrative input reduction is `(30,000 − 13,600) / 30,000 ≈ 54.7%`, or **about 55% in this example only**. It is not a measured Coopanion saving, bill reduction, or speedup.

An actual test must include all model turns, cache-hit pricing, outputs, tool results, retrieval, and maintenance. Compare the current version, a recent-window-only variant, and this proposal under the same replay. Report input tokens, total cost, latency, and memory accuracy per user interaction. See the [evaluation plan](docs/evaluation.en.md). Until then the publication claim is **“Designed to reduce repeated context; savings remain to be measured.”**

## How to use this package

1. Read the [memory lifecycle](docs/design.en.md) and [cold index](docs/cold-storage-index.en.md).
2. Follow the [Coopanion integration design](docs/coopanion-integration.en.md) to implement host-controlled storage, user-cue permission, budget checks, and the actual outbound context view.
3. Append the [English conversation policy](prompts/companion-policy.en.md) to the existing persona only after the host can supply a bounded memory snapshot. The [Chinese policy](prompts/companion-policy.zh.md) is an alternative, not an additional prompt to send every turn.
4. Use the [incremental extractor](prompts/memory-extractor.en.md) for proposed updates. Use the [cue router](prompts/cue-router.en.md) only when simpler rules cannot resolve a cue.
5. Check the [example policy](config/policy.example.json), [synthetic records](examples/memories.en.json), [30 acceptance scenarios](examples/scenarios.en.json), and [evaluation plan](docs/evaluation.en.md).

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

The JSON examples are available in [English](examples/scenarios.en.json) and [Chinese](examples/scenarios.json). They are specifications, not results from a running model.

## Implementation status

- [x] Design, prompts, synthetic records, and acceptance scenarios.
- [ ] Persistent cold archive with provenance and user-mention accounting.
- [ ] Coopanion/Cortico outbound context filtering and protection against generic-tool bypass.
- [ ] Verified token budgeting that preserves complete active tool transactions.
- [ ] Controlled A/B replay and publication of measured cost, cache, latency, and recall results.

This is an independent proposal for Coopanion, not an official Pal-AI-Lab feature or endorsement. Source analysis is pinned to the 2026-10-25 snapshots described in [References](REFERENCES.en.md); upstream changes require a new integration review. This package contains original documentation, prompts, and synthetic examples, not copied upstream source. Each upstream project retains its own license. The publisher should choose a distribution license before adding a LICENSE file.
