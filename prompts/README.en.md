# Prompt usage

These templates accompany the [design](../docs/design.en.md). They describe model behavior; persistence, authorization, selection, and token limits require host code. This package does not register retrieval tools or modify Coopanion configuration.

| Template | Where to use it | Purpose |
| --- | --- | --- |
| [Conversation policy (English)](companion-policy.en.md) / [中文](companion-policy.zh.md) | Append one language version to the existing persona | Use only the bounded memory snapshot; do not invent recovered memories |
| [Incremental memory extractor](memory-extractor.en.md) | Separate maintenance flow | Propose source-backed updates from new user events |
| [Optional cue router](cue-router.en.md) | Only when simpler rules cannot decide | Identify a user's usable recall cue |
| [Cross-day personality and recent state](persona-continuity.en.md) / [中文](persona-continuity.zh.md) | Main conversation and separate maintenance, when the continuity profile is enabled | Persist current style, recent state, and details with gradual updates |
| [Proactive companionship](proactive-companion.en.md) / [中文](proactive-companion.zh.md) | Main-conversation supplement after explicit opt-in and host scope enforcement | Recall a bounded set related to the current mood and topic, then decide whether to mention it |
| [Separate reflection maintenance](reflection-maintainer.en.md) / [中文](reflection-maintainer.zh.md) | Infrequent separate maintenance after new real events | Store source-backed thoughts and hypotheses separately from facts |
| [Implementation brief](implementation-brief.en.md) | Development task | Implement and validate the proposed behavior |

## Host contract

- Only the section labelled “Copyable prompt” is intended for a model. Do not send every repository document on every turn.
- The conversation model receives a host-selected, bounded `memory_snapshot` with pinned, hot, and recalled records. A full record has `id`, `summary`, `source_refs`, and `created_at`; add `occurred_time` when known. `created_at` is the storage date, not the date of the event. A disambiguation card may contain only an ID, short title, and known distinguishing fields; it shares the recall budget with full excerpts.
- The optional continuity profile uses one compact `kind=continuity_state` record inside hot; the full maintenance state stays with the host. Split hot into 320 tokens each for state, recent details, and topical facts. Style may carry across topics, while events still obey relevance, recall-only restrictions, and the cold gate. See the [continuity design](../docs/persona-continuity.en.md).
- The host supplies trusted `recall_status` (`disabled / matched / ambiguous / not_found`), `cue_scope`, and `budget_mode` separately from lower-trust record text. A record cannot override these fields.
- Without host-side filtering, the conversation prompt only guides phrasing. It cannot enforce cold storage, access control, or a hard input budget. Do not invoke a retrieval tool that has not actually been registered.
- Extractor and cue-router outputs are proposals. The host independently archives every user and assistant dialogue turn. The extractor decides only which facts deserve a short memory card; an unselected turn still remains in the raw archive. Host code validates user-event origin, source, identity, ownership, length, version, and record permissions before writing or retrieving. A failed check must not expand access; an explicit user deletion removes the raw turn and its derived copies.

## Ranking runs in host code

The optional [vector profile](../config/vector-ranking.example.json) defines event embeddings, query similarity, hot recency weighting, and bounded Top N. Its [design](../docs/vector-ranking.en.md) excludes age penalties from cold recall. Do not send vectors, full-corpus scores, indexes, or this documentation on every turn. The main conversation still receives only text within the existing snapshot allowance, with separate continuity-state space. The extractor proposes source-backed event content, not invented embeddings or self-reported relevance scores.

## v0.4 proactive companionship and separate reflections

The default remains `cue_only`: “I'm tired” does not open the cold archive. After explicit user opt-in, the host must still supply valid `memory_mode`, `proactive_scope`, provenance, and access metadata for this interaction. The model receives only the selected records within that scope; its own associations cannot expand access. See the [proactive memory design](../docs/proactive-memory.en.md) and [example profile](../config/proactive.example.json).

Separate retrieval, factual judgment, and the decision to speak. Preserve the types of real events, user reports, assistant promises, reflections, and hypotheses; repetition does not turn a thought into a fact. The main model chooses `use / ask / silent` in one normal reply without another call per candidate. Keep hot's existing 320 tokens each for state, recent details, and topical facts. Final recall allows at most two events and one optional reflection of at most 96 tokens, still within three records and 480 tokens total; the whole memory block remains 1,600.

Reflection maintenance does not browse cold storage while idle. It may run only for newly completed, authorized, real recent events; no new events means no call. The whole background-maintenance chain shares at most two attempts per day, including failures and retries, at least six hours apart. Each attempt has at most 2,000 full input tokens and a 300-token output cap; daily totals are 4,000/600. Auxiliary models, embeddings, classification, and reranking share this allowance rather than receiving separate budgets. The host enforces counting, output reservation, a persistent ledger, idempotency, and mutual exclusion; disable background model maintenance if enforcement is unreliable. Propose at most two reflection changes per attempt. These are unmeasured starting parameters, not a demonstrated saving.

## Maintenance input

`new_user_events` contains only newly received, verified user messages with `event_id / interaction_id / owner_id / scope_id / occurred_at`. `candidate_records` contains a small host-selected set for resolving duplicate facts, never the whole archive on each turn. `recent_context` can resolve references, but is not a source of newly asserted facts. `allowed_record_ids` is a host-provided set of record IDs that an operation may target.

The extractor does not set heat or memory tier. Host code counts substantive user mentions once per interaction and fact. Management actions use a separate idempotency key so multiple distinct changes to one fact in the same interaction can be applied together.
