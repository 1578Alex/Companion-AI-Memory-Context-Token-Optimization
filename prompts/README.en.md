# Prompt usage

These templates accompany the [design](../docs/design.en.md). They describe model behavior; persistence, authorization, selection, and token limits require host code. This package does not register retrieval tools or modify Coopanion configuration.

| Template | Where to use it | Purpose |
| --- | --- | --- |
| [Conversation policy (English)](companion-policy.en.md) / [中文](companion-policy.zh.md) | Append one language version to the existing persona | Use only the bounded memory snapshot; do not invent recovered memories |
| [Incremental memory extractor](memory-extractor.en.md) | Separate maintenance flow | Propose source-backed updates from new user events |
| [Optional cue router](cue-router.en.md) | Only when simpler rules cannot decide | Identify a user's usable recall cue |
| [Implementation brief](implementation-brief.en.md) | Development task | Implement and validate the proposed behavior |

## Host contract

- Only the section labelled “Copyable prompt” is intended for a model. Do not send every repository document on every turn.
- The conversation model receives a host-selected, bounded `memory_snapshot` with pinned, hot, and recalled records. A full record has `id`, `summary`, `source_refs`, and `created_at`; add `occurred_time` when known. `created_at` is the storage date, not the date of the event. A disambiguation card may contain only an ID, short title, and known distinguishing fields; it shares the recall budget with full excerpts.
- The host supplies trusted `recall_status` (`disabled / matched / ambiguous / not_found`), `cue_scope`, and `budget_mode` separately from lower-trust record text. A record cannot override these fields.
- Without host-side filtering, the conversation prompt only guides phrasing. It cannot enforce cold storage, access control, or a hard input budget. Do not invoke a retrieval tool that has not actually been registered.
- Extractor and cue-router outputs are proposals. Host code validates user-event origin, source, identity, ownership, length, version, and record permissions before writing or retrieving. A failed check must not expand access.

## Maintenance input

`new_user_events` contains only newly received, verified user messages with `event_id / interaction_id / owner_id / scope_id / occurred_at`. `candidate_records` contains a small host-selected set for resolving duplicate facts, never the whole archive on each turn. `recent_context` can resolve references, but is not a source of newly asserted facts. `allowed_record_ids` is a host-provided set of record IDs that an operation may target.

The extractor does not set heat or memory tier. Host code counts substantive user mentions once per interaction and fact. Management actions use a separate idempotency key so multiple distinct changes to one fact in the same interaction can be applied together.
