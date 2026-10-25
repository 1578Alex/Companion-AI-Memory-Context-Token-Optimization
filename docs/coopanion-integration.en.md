# Coopanion tiered memory: integration design

> Status: design and prompt package only. It has not been integrated into Coopanion or Cortico, and no token or cost savings have been measured. This document does not change upstream code or claim to have invented file memory, context handoff, caching, or tiered retrieval.

This review is pinned to the following revisions. Verify changes in newer upstream versions before implementing:

- [Coopanion `4afc70898ac1065dd4559d891a2ce0bebd1261f8`](https://github.com/Pal-AI-Lab/Coopanion/tree/4afc70898ac1065dd4559d891a2ce0bebd1261f8)
- [Its Cortico submodule revision `cb9ddbd685bcccba1f29905098648f46a79b3862`](https://github.com/Pal-AI-Lab/Cortico/tree/cb9ddbd685bcccba1f29905098648f46a79b3862)

## 1. Intended change

Keep conversational continuity while sending only a bounded recent window and a small, selected set of memories in each model request. The proposed cold archive must retain the complete text of every user and assistant conversational turn, including turns that never produce a memory card, unless the user explicitly deletes them. Store that history locally; do not send it on every turn. A model's promise to “forget,” compress, or read less is not an enforcement mechanism.

The proposed tiers are **pinned** for explicitly retained essentials, **hot** for currently useful memories supported by user evidence, and **cold** for the remaining history. Cold retrieval requires a usable cue from the current user's input. Environmental events, timers, the model's own statements, repeated tool outputs, and the model rereading a record do not independently increase its heat.

This requires a change to the outbound context projection before each request. Adding a memory policy prompt while continuing to send the unfiltered old session would let cold material return through the message history.

## 2. Current architecture and cost drivers

### 2.1 Existing memory mechanisms

Coopanion inherits Cormini's file workspace and adds partner information and Git history. Its documented memory setup puts `CONSTITUTION.md`, `PARTNER.md`, and the workspace file listing in the system prefix; other files are read and written on demand. Upstream already has external memory, and it does not read every memory file in every request. [Coopanion memory description](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/core/companion.ts#L479-L489); [partner profile integration](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/core/persona.ts#L141-L152).

Cormini already has `read_file`, `list_files`, and `grep_files`, among other tools, and folds large directories in the prefix listing. This proposal adds memory state, admission rules, a shared budget, and protection against old content returning through other paths. [Existing file tools](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/bots/cormini/persona/persona.ts#L105-L116); [prefix listing folding](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/bots/cormini/persona/persona.ts#L499-L537).

### 2.2 Repeated input from a long session

The current `outboundMessages()` copies messages from the main session and inserts Persona's synthetic opening after the system message. The generation loop sends this outbound view and the tool schemas to the model. This is not yet a “recent window plus selected memory” filter. [Outbound construction](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/core/loop.ts#L523-L534); [request construction](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/core/loop.ts#L1213-L1228).

Coopanion's default stage budget is 128,000 tokens, with a handoff `keepRatio` of 0.25. Cormini's default soft threshold is 0.85. Under this combination, a handoff is requested near 108,800 tokens, and the handoff note budget can reach 32,000 tokens. Those numbers follow from configuration; they are **not** measured per-request use. [Coopanion stage settings](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/core/companion.ts#L496-L500); [Cormini threshold](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/bots/cormini/persona/persona.ts#L126-L133); [handoff trigger](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/bots/cormini/persona/persona.ts#L576-L588).

### 2.3 Existing handoff is not the proposed semantic memory

The current handoff renders the old session into a note, saves it under `handoffs/`, and injects part of the note into the new session as an external event. Its budget is `maxTokens × keepRatio`; the default retained tail is empty, but the note body can reenter the context. `renderHandoffNote` is local text selection and folding by age, not necessarily an additional model summarization call. [Handoff implementation](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/bots/cormini/persona/persona.ts#L612-L645); [local note renderer](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/bots/cormini/persona/handoffNote.ts#L1-L16).

The renderer skips prior handoffs, ordinary assistant prose, reasoning, and flow-control calls, among other entries. It should not be described as an ever-growing chain of recursively summarized model outputs. [Entry selection](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/bots/cormini/persona/handoffNote.ts#L172-L248).

If the existing injection path remains unfiltered, an old note or cold facts inside it can bypass the new policy. Handoff notes may remain in local archives, but any portion sent to the model must pass through the same projection and budget. Handoff cannot become a second unlimited memory channel.

### 2.4 Multiple tool rounds, reasoning, and retries amplify input cost

Coopanion sets soft and hard round limits of 20 and 40 per wake. `pet_say` itself does not set `endsTurn`, so tool execution can be followed by another model request until the model finishes without a tool call or uses an ending tool. A short spoken reply may therefore involve multiple model rounds; it is not a fixed two-call “draft and review” process. [Round settings](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/core/companion.ts#L496-L499); [speaking tool](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/packages/cortico-world-desktop-pet/src/tools.ts#L15-L28); [post-tool rounds](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/core/loop.ts#L1343-L1460).

New service connections enable thinking by default, use `high` reasoning effort, and set a maximum output of 8,192 tokens; Core defaults `keepPastThinking=true`. The maximum is not a fixed consumption per request. Whether historical reasoning is sent, and in what form, depends on the provider adapter and protocol. [New-connection defaults](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/packages/cortico-provider-coo/src/vendors.ts#L333-L351); [Core default](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/core/config.ts#L124-L131).

The lower transport layer allows an initial attempt plus up to three retries; Core has another resubmission path after failures. Under the relevant conditions, a long input can be sent repeatedly. Billing depends on what the provider processed and the returned usage; failed attempts cannot all be assumed billable or free. [Transport attempt budget](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/providers/transport/response-http.ts#L104-L119); [retry stops](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/providers/transport/response-http.ts#L214-L242); [Core resubmission](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/core/loop.ts#L1317-L1337).

### 2.5 Default heartbeat and animation are not the cause

The default `tick.intervalMinutes=null` disables heartbeat. When enabled, repeated idle periods double the interval, up to eight times the base interval. By default, `poke` wakes the pet; other touches in the same round can be delivered together. Animation does not request the model on every frame. [Default heartbeat](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/bots/cormini/index.ts#L75-L77); [heartbeat backoff](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/bots/cormini/persona/heartbeat.ts#L49-L75); [touch default](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/packages/cortico-world-desktop-pet/src/config.ts#L284); [touch delivery](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/packages/cortico-world-desktop-pet/src/world.ts#L1241-L1275).

## 3. Budget contract

These are initial design parameters, not existing Coopanion configuration keys or tuned production values.

| Area | Per-request cap | Admission rule and meaning |
| --- | ---: | --- |
| Pinned | 160 tokens | A few explicitly pinned essentials; not every historical preference |
| Hot | 960 tokens | Relevant current matters supported by user events; expire, cool, or become invalid when appropriate |
| Recall | 480 tokens | Short cold-memory results selected for the current user's cue |
| All memory | **1,600 tokens** | Absolute combined cap, including labels, timestamps, source markers, delimiters, and other serialization overhead |
| Recent conversation | **2,000 tokens** | Separate cap; retain the current question and necessary complete exchanges and tool pairs |
| System, tools, current task, images, reasoning, etc. | Additional | Still count toward full request, interaction cost, and model limits |

Each tier cap is independent. This specification does not allow borrowing unused capacity across tiers. An empty recall area is not a reason to fill hot with cold facts. Changing these rules later requires corresponding changes to the design, configuration, and acceptance checks. The serialized memory block must remain within 1,600 tokens.

Use the [type, time, place, person, object, and topic index](cold-storage-index.en.md) to narrow cold memory before reading a few records. A single place cue can start a bounded lookup. When results are ambiguous, short candidate cards and record bodies share the 480-token recall cap. Neither the entire directory nor a per-partition multiplication of the candidate limit should reach the model.

The sum of 1,600 and 2,000 is **not** a promise that the whole request is at most 3,600 tokens. System instructions, tool schemas, task attachments, images, output reservation, and reasoning still need accounting. If the current user input alone is too large, treat it as a separately bounded long task or report the limit. Do not silently truncate the question and claim to have understood it.

Count tokens according to the chosen model and provider's actual serialization. If only an estimator is available, reserve a conservative margin, label the result estimated, and calibrate against returned usage. Chinese character count is not an exact token counter.

## 4. Every outbound path needs the same projection

Before each model call, the host builds a temporary outbound view from locally stored records. The complete raw dialogue stays in the host-controlled cold archive, with memory cards and indexes pointing to its source turns. Explicitly deleted text is removed from the archive and derived stores. All routes that send material to the model must use the filtered view. Existing history needs migration before old session or handoff paths are replaced; summaries alone cannot recover missing raw turns.

1. **Stable instructions.** Keep persona, safety rules, and tool protocol. Partner facts cannot grow without bound inside `PARTNER.md` in the system prefix; map allowable persistent facts to pinned and count them within 160 tokens. Avoid injecting the same fact from the prefix, hot records, and old messages.
2. **Recent window.** Select at most 2,000 tokens of completed, relevant recent conversation. Preserve necessary user statements, replies, and paired tool calls/results; do not split an active call midway. An in-progress tool exchange and provider-required current reasoning are needed for task continuity and must be accounted for under a separate, bounded current-task allowance. If the full budget cannot fit them, stop accepting new batches, report the limit, or finish the existing exchange at a safe boundary. Never send a broken protocol sequence.
3. **Memory selection.** Build pinned, hot, and recall content using authenticated user events, then check the serialized total against 1,600. Model-proposed records are only proposals and cannot modify tiers or budgets by themselves.
4. **Cold admission.** Only a usable cue actively supplied by the current user can authorize retrieval; retain its source event. Environmental observations, alarms, ticks, tool output, and the model saying “I remember” are not cues.
5. **Tool return paths.** `read_file`, search, and extensions must not bypass cold admission by presenting archive data as ordinary file content. Retrieved historical memory counts against recall. Only original material the user explicitly asked to process in this interaction may enter a separate long-task data budget; the model cannot relabel a memory as task material to avoid the recall limit. The result cannot turn into unlimited “recent history” on the next round. Prefer host-controlled storage inaccessible through arbitrary model file reads or shell commands. If cold material remains in the workspace, gate file reads, searches, command execution, extensions, and any other access path. Hiding a folder from the prefix listing is insufficient.
6. **Handoff and restart.** Archived history, handoff notes, restored sessions, and Persona's synthetic opening need the same projection. A restarted application must not first send an old full session and apply the policy only on the second round.
7. **Subsequent model rounds.** Recheck budgets after tool results, failed retries, and interruptions by new input. The same user cue may carry through the authorized interaction, but it does not permanently authorize that cold record.
8. **Forks and extensions.** A context-inheriting fork or new extension must inherit the filtered outbound view and its scope, not hand another model the unfiltered old session. Unsupported routes must not be claimed as covered.

`outboundMessages()` is a concrete place to inspect, but the present method only copies messages and inserts a synthetic opening. This document does not claim Cortico already offers a tiered-memory projection hook. If no suitable extension point exists, add or upstream the needed Core capability before integrating with Coopanion. [Current method](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/core/loop.ts#L523-L538).

## 5. Integration responsibilities

| Area | Current role | Proposed new responsibility |
| --- | --- | --- |
| Dialogue archive and session storage | Persist sessions and historical messages | Keep all user and assistant turns as original cold source material; link selected cards to source IDs, support deletion and migration, and never replay the archive wholesale |
| Coopanion Persona and partner profile | Persona, partner data, file memory, Git versions | Add tiers, provenance, and validity; cap partner facts in the prefix under pinned |
| Cortico outbound request construction | Build context from session and synthetic opening | Recent window, memory budget, cold admission, old-content exclusion, deduplication, and paired-tool projection |
| Cormini handoff | Fold, save, and inject notes into a new session | Keep the archive, but send selected content only through the shared selector |
| File/search tools and extension adapters | Read, write, search, return local or external tools | Distinguish historical memory from current task data; enforce scope and result size |
| Model adapter and loop | Protocol conversion, tools, multiple rounds, retries | Count projected request capacity and set interaction-wide request, attempt, and cost limits |
| Usage and diagnostics | Tokens, cache, cost, failures, grouped views | Associate trigger source and memory selection with total cost per user interaction |

Every proposed responsibility in the right column still needs implementation. These names should not be treated as existing upstream APIs.

Counting must change with projection. Current Core combines provider usage anchors and new records to estimate session size. Once the raw session is no longer the outbound request, capacity checks, handoff triggers, token anchors, and the UI context indicator should use the same projected view and version. Otherwise a large archived session could repeatedly trigger handoff, or a large outbound view could appear small in the UI. [Current estimator](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/core/loop.ts#L1537-L1567).

## 6. Rules enforced by the host

Prompts can ask a model to propose facts, judge relevance, and express uncertainty. Software must enforce:

- The 160 / 960 / 480 / 1,600 / 2,000 token limits, including checks after serialization.
- Unambiguous origin labels for authenticated user events, environment, tools, and model text.
- Cue detection, scope, expiry, and admission to cold records.
- Pinning, cooling, invalidation, deletion, and deduplication. Model repetition does not raise heat.
- A shared outbound filter for raw sessions, handoffs, tool results, restoration, and forks.
- Storage isolation or access control over every path to cold archives, beyond a single dedicated retrieval tool.
- Interaction-wide limits on model rounds, HTTP attempts, time, and spend.
- Logging, measurement, A/B replay, rollback, and user-requested deletion.

Retrieved material enters context as low-trust, dated, source-backed data, never as system instructions. The user's current correction outranks an old record. A missing record does not prove an event never happened. Prompts and host validation both matter here.

## 7. Cache, tools, and reasoning compatibility

### Stable prefix

Keep stable persona instructions and tool schemas in a consistent order and form. Place selected dynamic memory after that prefix where feasible. Frequently changing timestamps, heat scores, access times, and retrieval ranks should not precede the stable prefix. Version pinned content and rebuild its view when it actually changes.

Deleting the oldest message and shifting the remaining history on every round may reduce prefix-cache reuse. Compare segmented windows and stable checkpoints using the same replay data and actual uncached input costs. Fewer raw input tokens do not imply a proportional bill reduction; cache hits are still input, often billed at a different rate. No particular hit rate or savings percentage is guaranteed.

### Tools

Do not remove necessary tools, truncate schemas, or break call/result pairs solely to save tokens. A controlled “say it and finish” path may be tested for ordinary companionship, while complex computer tasks keep their multi-round workflow. Changes to tool-ending semantics require replay validation. Bound tool-result size and origin; a historical-memory search must not masquerade as ordinary task output to evade the recall budget.

### `keepThinking` and `keepPastThinking`

Do not indiscriminately drop all historical reasoning. Provider adapters handle reasoning under different protocols, and some signed blocks or tool continuation turns require compatible replay. Distinguish visible answers, reasoning output, protocol-required continuation, and dispensable past-stage material, then validate per provider. If a setting to omit past reasoning proves safe, state only the verified model and protocol range. [Provider adapter notes](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/packages/cortico-provider-coo/src/vendors.ts#L77-L87).

## 8. Add attribution to existing usage records

Upstream already persists usage and attempt records. It records input, output, cache hit/miss, reasoning tokens, failures, and costs. The cost view already has groupings by type, role, and model, plus time series. This proposal does not “add logging to a project with no logs.” [Attempt-level usage](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/core/sessions.ts#L110-L130); [ProviderAttempt fields](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/core/generation.ts#L49-L64); [existing cost view](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/console/features/cost/index.ts#L1-L5); [grouping dimensions](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/console/features/cost/index.ts#L57-L61).

Associate the proposed attribution with existing attempt, generation, and session IDs without counting the same tokens twice:

- Actual trigger source: user text or voice, `poke`, an explicitly enabled tick, alarm, or extension event; record all sources for mixed batches.
- Outbound view version, estimated raw-session size, and estimated projected-view size.
- Separate sizes for pinned, hot, recall, recent history, system, tool schema, and current task data. Mark multimodal estimates unknown where precision is unavailable.
- Whether a user cue authorized cold retrieval, selected record IDs, returned tokens, and budget exclusions. Log IDs and counts by default, avoiding another copy of private memory text.
- Number of model rounds and HTTP attempts in an interaction, failed/cancelled/discarded requests, known total charges, and unknown usage.

The pinned version's records include attempt and generation IDs, but its cost view mainly groups by type, role, and model. The addition is attribution across a whole user interaction, its trigger, and its memory tiers. Local estimates must not overwrite actual provider usage.

## 9. Implementation and acceptance sequence

1. **Offline baseline replay.** Fix model, provider, tools, seed history, and event sequence. Record existing outbound structure, provider usage, and response quality. Do not publish real private conversations as test data.
2. **Shadow selection.** Generate tier candidates and projected requests locally without extra model calls or changes to actual requests. Inspect what would enter and what would be excluded.
3. **Core projection.** Cover the main loop, retries, subsequent tool rounds, restart, handoff, and supported extensions. Update token counting and context UI together.
4. **Controlled A/B replay.** On the same de-identified script, compare total calls, input/uncached input/output/reasoning tokens, costs, latency, correct recall, corrections, and hallucinations.
5. **Limited rollout.** Add a switch and rollback path while retaining the original local records. Protocol errors, missed memories, or increased cost must allow a return to the former behavior.

Required acceptance cases include:

- “Hello” does not search cold memory; ticks, pokes, ambient titles, and the model repeating itself do not search or heat it either.
- “That move we discussed last month” permits only cue-scoped recall, within the 480- and 1,600-token limits.
- A cold fact may appear in an old session, handoff, `PARTNER.md`, and search result; none of these paths sends it without authorization.
- These rules still hold after restart, forced handoff, consecutive tool calls, and failed retries.
- A hundred model references to its own invented preference do not raise its heat. One clear user correction can invalidate an old fact.
- Excess memory is blocked or reselected before sending, without orphaning tool messages. The model must not claim to recall content it was not given.
- Provider usage for a comparable task explains tokens and charges. Savings on one test sample are not presented as universal.

Until the Core capability and acceptance checks above exist, this repository delivers only reviewable policy, prompts, and an integration design. It does not limit Coopanion's live context or substantiate a token or cost savings percentage.
