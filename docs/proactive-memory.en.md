# Proactive recall and a separate reflection area (v0.4)

The goal is to respond to “I'm so tired today” with some sense of shared history. Remembering something does not mean it must be mentioned, and mentioning it must not turn a guess into a fact.

This optional extension awaits implementation. See its [profile](../config/proactive.example.json) and [cases](../examples/proactive-scenarios.en.json). It has no runtime integration or measured gains in believability, accuracy, or savings. The default remains the [base design](design.en.md)'s user-cue mode. Pasting a prompt does not enable a working memory controller.

## 1. Decide whether retrieval is allowed, then whether to mention it

Only an explicit user opt-in to proactive companionship, with a chosen association scope, lets an authenticated current user interaction provide a situational or emotional entry point. “Tired” may lead the host to search permitted routine work or sleep records. It does not establish overtime as today's cause. Without a discriminating situation or relevant evidence, respond to the present message rather than scanning the archive for topics.

The host first applies owner, scope, validity, and user restrictions, then runs one bounded retrieval within the permitted topics and categories. Emotion cannot authorize access to a record marked “only when I ask”; explicit recall cues still follow the base rules. Sensitive matters require an explicit cue, never an inferred mood. Timers, assistant self-talk, tool outputs, and screenshots are not authenticated user triggers. Disabling this extension immediately restores the base gate.

Keep three decisions separate:

1. **Find relevant records.** Use the [vector extension](vector-ranking.en.md)'s semantic search, exact cues, date filters, and deduplication, returning at most 12 fused candidates. Emotional similarity is only a relevance signal. Old events get no age penalty; whether a fact still applies now is checked separately.
2. **Check what the evidence supports.** Verify the original speaker, time, validity, and source. “Overtime exhausted me last week” establishes that past report, not today's cause. Unsourced, invalidated, or reflection-only candidates cannot become experienced facts.
3. **Choose use / ask / silent.** Use a record naturally when current evidence is clear and helpful. Ask gently when there is only a historical association. Stay silent about the old event when evidence is weak, causes conflict, the topic was repeated, or the moment is unsuitable. Silent still allows a normal response to the present message.

A fictional example:

> User: I'm exhausted.
> Companion: Is it work again? You said that stretch of overtime was tiring last time.

This is a source-backed question, not an assertion that the user worked overtime today. If the user says no, correct the current understanding and stop pursuing overtime. Decide expression in the same main response generation, without a separate large-model call for every candidate. Host code grants retrieval and enforces restrictions; the model only proposes how to express it.

## 2. Be proactive without repeatedly reopening old topics

A proactive permission is bound to its originating user interaction, identity, and topic. It lasts at most five follow-up user interactions or 30 minutes, whichever ends first, and expires immediately on a topic change. Assistant replies, idle time, and an unqualified “continue” do not renew it. After a proactive retrieval attempt, apply a 24-hour cooldown per user/topic, including attempts that led to silence. Do not keep retrying in the background. A new explicit recall cue can use the base route independently of proactive cooldown.

After the user rejects the association or asks to stop, suppress that proactive topic until the user reintroduces it or explicitly lifts the restriction. “Do not bring this up unless I ask” also writes the appropriate recall_only restriction; a one-day cooldown alone is insufficient. Reading, guessing, mentioning, silence, retries, and automatic summaries do not add qualifying mentions, heat, or familiarity. Only a substantive real-user confirmation or addition may count under the base rules.

## 3. Keep experienced events and the character's thoughts separate

The independent reflection area uses `kind=reflection` or `kind=hypothesis`. It is neither an alias for the event archive nor a hidden reasoning log. Store bounded, inspectable character observations or guesses, such as “If they want to talk next time, ask how work is going,” not “They worked overtime today.”

Each record needs `id`, `owner_id`, `scope_id`, `kind`, short content, `source_refs`, `generated_at`, `valid_until`, `status`, and derived-source dependencies. Use `unverified` for an unconfirmed thought, `superseded` for a correction, and `deleted` for deletion. Expiry removes it from the visible set. The illustrative validity limit is seven days, awaiting calibration. The fact that the system generated a thought does not establish the external situation described by that thought.

Reflection may use only a small, host-authorized set of recent real events and current anchors. Background work must not search the cold archive. An older event can become a new permitted recent input only after lawful recall in a current user interaction. A reflection cannot become evidence for another factual record; repetition does not raise confidence or automatically strengthen stable personality or relationships. When the user later confirms something, derive or correct a fact from that new user source. Do not relabel the old guess as an experienced event.

Storage lifetime and current visibility permission are separate. Recalling an old event does not turn it into a new recent fact. Before background input or each reflection injection, recheck the original sources' permissions, current topic, and permit expiry. Cold-derived content becomes invisible after its original permit expires or the topic changes, unless a current user interaction grants a new valid permit. Seven days is only the maximum reflection-record lifetime, never seven more days of cold-access permission.

When a source is corrected, restricted, or deleted, withdraw dependent reflections, summaries, vectors, and request caches and recheck other derived state. Emotional continuity cannot revive invalid material or copy reflections into resident personality state.

## 4. Preserve personality and recent detail within the same allowances

The [continuity extension](persona-continuity.en.md) separately preserves the character foundation, relationship baseline, recent affect causes, and unfinished matters. Transient mood may settle and good news may change the tone. A new day neither resets familiarity nor resolves unfinished matters. Proactive recall and reflection cannot evict current state or rewrite the base persona.

- The memory snapshot remains at most 1,600 tokens: pinned 160, hot 960, recalled 480, with no borrowing.
- Hot remains state 320/one record, recent detail 320/three, topical facts 320/four: at most eight total and two ordinary event records per topic.
- A proactive output contains at most two event records. Reflection is not injected by default on every turn: optionally include at most one record and 96 tokens including its wrapper. Events, candidate cards, and reflection **together share three items/480 tokens**. Reflection deducts from those 480, not an extra unlimited bucket.
- Deduplicate event bodies across recent history, hot, recall, and reflection references. Source pointers may remain, but bodies do not repeat. If a valid thought cannot fit within 96 tokens, omit it instead of cutting negation or source qualifiers.

## 5. Do not let background work idle-loop

Illustrative background limits, untested with real users:

| Component | Ceiling |
| --- | ---: |
| Total maintenance call attempts per day | Two, including failures and retries |
| Total input / output per call | 2,000 / 300 tokens |
| Total input / output per day | 4,000 / 600 tokens |
| Minimum interval between maintenance attempts | Six hours |
| Proposed reflection changes per call | Two records |

Enqueue only after new completed interactions, coalesce nearby new events, and execute when idle. With no new event, skip. Use Asia/Shanghai day boundaries and continuous elapsed time for the six-hour interval; midnight cannot allow an immediate extra run. Model, embedding, classification, reranking, and retry calls all consume the same maintenance call/token ledger. If an embedding takes one call, it cannot become a free step outside reflection accounting. Local rules and index arithmetic also incur recorded time and compute costs, though they are not model calls.

Before dispatch, use a reliable counter to reserve input and output allowances, including instructions, state, sources, and protocol framing. Enforce a hard output cap and reconcile against provider usage. Do not dispatch without capacity. If a failed call's usage is unknown, conservatively keep its reservation charged. Without reliable input counting or output control, disable background model maintenance and retain rule-based archiving; character estimates do not establish a hard cap. Main conversation, online retrieval embeddings, and all auxiliary costs belong in the interaction total as well; being outside the background allowance does not make them unreported.

Persist daily call/input/output reservations and usage, source watermarks, queue idempotency keys, and the latest attempt time. Use a single-worker lock or atomic version check. Restarts, concurrency, timeouts, and interaction retries cannot reset the ledger or consume source events twice. Disablement, revocation, and deletion cancel related pending jobs. Proactive memory does not require installing a new scheduled system service.

## 6. Merging compresses; time management determines what still applies

Merge duplicate representations of one event or fact, not two instances of tiredness with different causes. Preserve speaker, date, negation, distinct causes, and sources. Derive a valid summary from original events rather than repeatedly summarizing guesses into experienced facts. Important events may retain independent detail while still obeying permissions, corrections, and budgets. Keep original dialogue archived unless explicitly deleted.

Time still governs current-state applicability, unfinished-item review, and reflection expiry. Merging cannot replace all temporal management. An old event remains retrievable by its clues, while old work stress does not automatically explain today. See the [proactive cases](../examples/proactive-scenarios.en.json): expected values are requirements to execute, not passed runtime tests.
