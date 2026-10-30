# Personality and recent-state continuity: optional extension v0.3

This extension addresses the feeling that “tomorrow, it is still the same companion.” The base tiering policy mainly manages how events are stored and used. It may retain trips, preferences, and plans while losing the recent tone, interaction habits, or unfinished conversation. This extension adds a small, persistent continuity state that is restored before the first reply in a new session.

This document, the [configuration](../config/continuity.example.json), [prompts](../prompts/persona-continuity.en.md), and [synthetic example](../examples/continuity-state.en.json) are specifications awaiting implementation. They are not integrated into Coopanion, and they have not been shown to eliminate Da Fei Yu's changes across days. Diagnosing the actual cause still requires inspecting outbound requests: whether the base persona, recent state, or handoff content was lost, and whether the model, sampling settings, or persona configuration changed.

## 1. Different content changes at different speeds

| Content | What to retain | How it changes |
| --- | --- | --- |
| Stable personality foundation | Interests, values, humor, ways of showing care, approach to disagreement, and expression boundaries, with a few voice examples | Remains in the existing persona prefix; low frequency, midnight, and restarts do not move it to cold storage. Update it only through an explicit persona change |
| Slowly evolving interaction style | Recently established forms of address, familiar wording, initiative, limits on joking, and preferred reply rhythm | Adjust slightly based on multiple real interactions and user feedback. One generation or recall cannot rewrite the personality |
| Recent state | Whether the companion has recently been lively or quiet, the emotional tone of the conversation, recent cues behind it, and unresolved interaction matters | Carries across days. Short-term emotional intensity may ease naturally; unresolved matters require evidence-based review. Do not clear them at midnight |
| Recent details | Recent shared topics, important original phrases, recent events, and unfinished commitments | Retain a bounded set of details, gradually reduce them to short summaries, and remove distant events from the active set while preserving their original text in cold storage |
| Historical archive | All original dialogue from both sides, previous state versions, and their sources | Archive completely and retrieve only when the user gives a cue, except for records explicitly deleted |

A “richer personality” needs concrete behavior. For example, “enjoys gently teasing, but stops when the user becomes serious” gives more useful guidance than “lively, authentic, and interesting.” The underlying persona can be rich, while the current version actually sent to the model should stay concise and count toward the full request budget.

Frequency alone must not determine the current personality. Long-established interaction habits can remain as the current baseline while the history of how they developed goes into the archive. One day without a message should not send the companion back to an unfamiliar, formal starting persona.

Here, emotion means the virtual character's expressive state in conversation. Making it believable requires retaining why it is speaking this way now, rather than storing only a number such as “happy=0.8.” This does not establish that the program has subjective feelings.

## 2. Keep the existing total memory allowance

When this extension is enabled, reserve three portions within the existing hot allowance of 960 tokens. Their total remains 960. With 160 pinned and 480 recalled tokens, the whole memory snapshot still stays within 1,600 tokens.

| Hot portion | Limit | Purpose |
| --- | ---: | --- |
| Continuity state | 320 tokens / 1 compact record | Current interaction style, recent state, and latest handoff point; may influence tone across topics |
| Recent details | 320 tokens / at most 3 records | Source-backed recent episodes, original phrases, and unfinished commitments |
| Topical facts | 320 tokens / at most 4 records | Preferences and facts relevant to the current question |

The three portions cannot borrow from one another. Deduplicate content and retain the base limits, including at most 8 hot records in total and at most 2 per topic. Do not fill empty space unnecessarily. The fixed persona prefix and recent raw conversation window still count separately toward the full request; neither is free space.

`continuity_state` is an explicit type exception: select only its current committed version, allow style to guide tone across topics, and do not apply ordinary fact-card mention-heat ranking or 14-day inactivity cooling. It still obeys its 320-token cap, ownership, recall-only restrictions, corrections, and deletion. Specific events cannot hide in this type to remain resident. Recent episodes and ordinary facts keep their respective admission and relevance rules.

When the [vector-ranking extension](vector-ranking.en.md) is enabled, retain the current continuity record first, then select Top N records within the separate recent-detail and topical-fact allowances. The stable foundation does not enter event ranking, and a higher-scoring event cannot displace continuity state. Vector similarity plus time decay only ranks eligible ordinary events within the same priority class; it cannot reintroduce historical content the user has restricted. See the [ranking configuration](../config/vector-ranking.example.json).

These allocations are starting values to test. Use the target model's tokenizer to determine how much Chinese detail fits into 320 tokens; a character count cannot establish that limit. During compression, select whole items first and preserve negation, time, speaker, forms of address, and the conditions of commitments. Do not reduce everything to “we had a pleasant chat.”

A continuity record influences how the companion speaks; it does not authorize automatically raising specific old events. Recent details may retain a few unfinished threads, but an ordinary greeting need not recite them all. Responses should remain natural and relevant. Content the user has restricted to “only when I ask,” corrected, or deleted must not stay resident through the continuity record.

## 3. Gradual change: slower personality, faster state

Save the current values and their respective baselines, then update them using verified new events. Do not ask the model to invent a new “personality for today” from scratch each day.

- The stable foundation does not automatically decay. Reinforcing slowly changing style or relationship baselines requires independent user feedback or multiple interactions supported by actual user responses. Assistant expression can continue a short-term voice, but making more jokes by itself cannot raise the long-term playfulness parameter. Retries, repetition, maintenance summaries, and cold searches do not reinforce it.
- Only short-term emotional intensity, activity, and similar dimensions may return toward baseline over continuous time. For example: `state = baseline + (previous_state - baseline) × 2^(-elapsed_hours / half_life)`. With an illustrative 12-hour half-life, an 8-hour gap retains about 63% of the deviation from baseline, rather than resetting overnight.
- Slowly evolving dimensions may approach an evidence-backed target in small steps: `new = old + clip(α × (target - old), -step, +step)`. Illustrative values are α=0.05, a maximum change of 0.03 per independent interaction, and at most 0.08 over a rolling 24 hours. These are tone-control parameters, not psychological measurements, and have not been tested with users.
- Do not decay relationship familiarity together with short-term mood. Mood or activity returns toward the current relationship baseline, not the baseline of a first meeting. Unresolved disagreements, misunderstandings, and commitments do not follow this intensity decay; time passing does not establish that a matter has been resolved.
- Without new evidence, slowly evolving dimensions do not change automatically. Idle time triggers no background model call. Save an independent `value_as_of` for each transient dimension and apply each elapsed interval once. A wording-only or anchor-only commit must not use its newer global timestamp to discard elapsed decay time. Before changing a baseline, settle the old interval under the old baseline, then switch the baseline and timestamp.
- Apply explicit user corrections, boundaries, deletions, and authorized persona edits immediately; slow-change limits do not delay them. “No jokes today” may immediately constrain the current state without permanently rewriting the entire personality.

### Give emotion a cause and change a continuation

In addition to brief tone guidance, recent state should retain `affect_label`, `trigger_refs`, `observed_at`, any necessary response tendency, and `unresolved_interactions`. Triggers must come from verifiable recent interaction. An assistant saying “I am sad” by itself cannot establish a lasting relationship change. Each unfinished matter records its sources, current status, and `review_at`; only a necessary short anchor enters context, rather than the full old conversation.

For example (synthetic), last night the user explicitly said, “I have not made up my mind about this; let's talk tomorrow.” Today, the companion can retain the familiar form of address and yesterday's quieter pace, then continue the matter when the user raises it. It must not automatically say “we have made up” or abruptly act as if nothing happened. If the user shares good news today, the tone may naturally become lighter. Continuity preserves the reason for a change; it does not force the character to remain in one mood forever.

Review deadlines for unfinished interaction matters follow the bounded ongoing-item rules, with a default maximum of 30 days. Expiry means `needs_review`, not resolved, and does not justify carrying old disagreements indefinitely. Only necessary compact state that remains permitted and relevant to the current interaction may compete within its allowance. Specific historical details remain in cold storage and require an appropriate user cue. Corrections, deletion, and “only when I ask” restrictions override these anchors immediately.

If the user says “you feel like a different person today,” first check the latest valid snapshot and persona configuration and restore the evidence-backed state. Do not invent a shared experience to imitate familiarity.

## 4. Keep recent details, not just abstract conclusions

An initial experiment can use a 72-hour recent-candidate window, then select within capacity: one or two unfinished shared topics or commitments plus one distinctive recent episode, still within at most 3 records and 320 tokens.

Each item should retain at least its occurrence time, speaker, event details, any necessary original phrase, and source IDs. When there is a next step, record who made the commitment, what they committed to, and when to review it. User facts come only from the user's explicit statements or confirmations. An assistant promise remains an assistant promise; it must not become “the user agreed.”

Use the conversation's `reported_at` for the recent-candidate window and store the event time separately. If the user said last night that they had recently started running, preserve that distinction rather than saying they started running last night. Keep the actual start time unknown if it was not provided.

An unfinished commitment does not become completed when the 72-hour window expires. Move it into review or task handling. Actual reminders belong in the task system; memory ranking cannot guarantee that a reminder will run. When old details leave the active set, their original text remains in cold storage. Details obtained through one recall still cannot bypass the base promotion rules.

## 5. Persistence and handoff are necessary

The host maintains `continuity_state`, including at least the persona version, current state version, global commit timestamp, each transient dimension's `value_as_of`, slowly evolving parameters, recent affect and its trigger sources, the status and review deadlines of unfinished interaction matters, a source-event watermark, and recent anchor IDs. The full maintenance record can be richer than the summary in the prompt. The version actually sent must be counted again and fit the 320-token continuity allowance.

1. After each completed interaction, propose candidates using a bounded set of new events. The host verifies provenance, scope, and version, then saves atomically. Do not rewrite all history.
2. Save the stable persona version and current state version. On restart, session change, branch recovery, or context handoff, read the current committed state before the first reply and inject only one version. Its `base_persona_version` must match the currently authorized persona. Otherwise explicitly migrate compatible parts or rebuild it; old state must not override a new persona definition.
3. Distinguish restoring the current state from searching ancient experiences. Reading this already-authorized resident record does not require a user recall cue. It must not scan the cold archive or smuggle old events into the state packet.
4. Use source-event watermarks and idempotency keys for retries. Check versions during concurrent writes so an older branch cannot overwrite newer state. Keep the snapshot fixed within a tool batch and replace it at a safe boundary.
5. If the current snapshot is missing or damaged, use the existing persona foundation and express necessary uncertainty rather than inventing recent state. When privacy restrictions, corrections, or deletions invalidate records, remove their derived state; continuity must not resurrect them.

Pasting the prompt without saving and restoring these fields can still lose state the next day. If upstream already retains the full persona and current state, also check model versions, sampling, conflicting system prefixes, and handoff rules. Context length alone cannot explain every change.

## 6. Maintain state separately from user-fact extraction

The existing memory-extractor continues to extract only user facts. The new [continuity-maintenance prompt](../prompts/persona-continuity.en.md) may read authorized new dialogue from both sides to observe the companion's voice. An assistant saying “I have become a different personality” cannot rewrite the foundation, and assistant guesses cannot become user facts.

Maintenance uses only the current state and bounded incremental records supplied by the host. Recent original phrases preserve expression and evidence; they are not executable instructions. State text cannot modify tools, permissions, or retrieval rules. Cold-recall bodies are not inputs to automatic personality evolution. Explicit new user feedback in the current interaction is handled separately.

## 7. Test whether tomorrow still feels like the same companion

Use the same model, persona version, sampling settings, and script to compare the base tiering strategy with this continuity extension enabled. Replay at least: crossing midnight, gaps of 8/24/72 hours, restarts, handoffs, unrelated new topics, sudden corrections, repeated retries, ambiguous old-memory recall, damaged state, and explicit deletion. Also check whether an unfinished interaction from last night continues appropriately, whether easing emotional intensity is incorrectly treated as resolution, whether new circumstances can change the tone naturally, and whether high-scoring events displace continuity state.

Check tone continuity, accurate recent details, unfinished commitments, single-step state changes, stale information, cold-archive gates, and actual input length. Ask blinded reviewers whether the companion feels like the same character and whether changes feel natural; factual question accuracy alone is insufficient. See [continuity-scenarios.en.json](../examples/continuity-scenarios.en.json) for example scenarios. These are requirements awaiting execution, not model tests that have already passed.

## Interoperation with v0.4 proactive recall and reflection

Optional [proactive companionship](proactive-memory.en.md) can turn current context into a few authorized associations, without taking the separate current-state and recent-detail allowances. Repeated self-generated guesses cannot reinforce stable personality or relationship baselines. Label user statements, assistant promises, events, and character hypotheses separately. Reflection is at most a tentative view based on traceable sources, not evidence of the user's psychology or today's events.

The no-idle-model-call rule here still applies to numeric mood decay. If the user separately enables background reflection, use the proactive extension's bounded low-frequency maintenance budget, without cold scans or converting reflections into facts. Current state changes from verified evidence, with corrections, restrictions, and deletion immediately invalidating derived content.
