# How details cool down gradually

The goal is to keep recent, frequently discussed events specific, retain a usable short summary as they become less active, and only then remove them from current context. Original dialogue remains in the cold archive and can be found when the user gives a cue. These are proposed storage and context-selection rules, not implemented human forgetting or measured results.

## 1. Three representations, one original archive

| Stage | Content available to the current request | What remains stored |
| --- | --- | --- |
| detail | Distinctive quote fragments, concrete people/times, unfinished threads, and commitment conditions | Complete two-sided dialogue and provenance |
| summary | Event essentials, critical qualifications, next step, and sources; more than “we had a pleasant chat” | Original text and traceable derived representations |
| cold | Excluded by default; bounded retrieval only with a user cue | Original text, logical indexes, and permitted derived records |

`representation_stage` does not add a storage partition. Detail and summary are bounded derived views of the same record. Archive the original conversation once, send only one representation per ID in a request, and do not duplicate counts or append both versions. Ordinary detail/summary stages remain hot candidates; cold is outside hot. Eligibility does not guarantee injection: the current topic still matters.

The stable persona base, current committed `continuity_state`, and explicitly pinned records do not use ordinary event-cooling clocks. Their own update, recall-only, deletion, and budget rules still apply. Historical personality versions and old events can be cold while the compact current persona and relationship baseline remain available. See the [continuity design](persona-continuity.en.md).

## 2. Deterministic stage selection

Experimental starting values are in `lifecycle.representation_policy` in the [base configuration](../config/policy.example.json). They have not been tested with real users. Measure continuous elapsed time since the latest verified substantive user mention. Keep report time separate from event time. Midnight, restart, and summary-write time do not reset the clock.

For ordinary events, select in this order:

1. **Apply validity and user controls first.** Deleted content is unavailable; superseded content is not a current fact. “Only when I ask” immediately becomes cue-only recall. Corrections, deletions, and access restrictions do not wait for gradual stages.
2. **Apply existing type exceptions.** Current continuity state and pinned records follow their own rules. Keep active tool transactions protocol-valid and replace snapshots at a safe boundary.
3. **While cold conditions are unmet, prefer detail for the first 72 hours, then summary.** A fresh substantive user mention of a still-hot record can refresh its detail preference after updating provenance. Assistant repetition, retrieval, maintenance summaries, and retries do not refresh the clock.
4. **Ordinary time-based cooling requires a summary-eligible interval.** For an ordinary record without a valid ongoing marker, cold eligibility begins after 7 days since the latest qualifying mention with heat below 0.30, or after 14 days without a qualifying mention. At least 24 hours must have been summary-eligible. Valid ongoing matters first retain bounded, relevant compact state until review; neither low heat nor inactivity alone directly demotes them.
5. **Cold records do not promote themselves through background reads or writes.** Existing user-cue and promotion conditions remain in force. One recall serves the authorized current topic and does not make full detail permanently resident.

Illustrative host selection logic, not an existing Coopanion API:

```text
Filter deletion, invalid status, access, and recall_only first
Apply the dedicated rules for current continuity_state / pinned
An ordinary record already cold stays cold unless recall or promotion is authorized
Otherwise age = now - latest verified substantive user mention time
If valid ongoing: choose detail or summary by age; expiry becomes needs_review
Else if at least 24 hours were summary-eligible and either holds:
    (age >= 7 days and heat < 0.30) or age >= 14 days, target cold
Else: target detail for age < 72 hours, otherwise summary
Then validate the representation and apply relevance, item, and token limits
```

When tuning the profile, validate temporal ordering: normal cold grace should be at least the detail-preference period plus the summary-eligible period. The default 7 days exceeds 72+24 hours. The 7-day candidate grace neither guarantees 7 days of injection nor reserves a slot on every mention. Cooling is host selection of what the next request may see. Sending the whole original and asking the model to ignore it does not implement this policy.

## 3. Prepare summaries from sources, not from repeated summaries

While processing new events, prepare a source-backed detail view and shorter summary for the same record. Derive and verify both from allowed original evidence. Repeatedly summarizing old summaries can distort names, negation, commitments, and times. Changing representation neither copies the whole original nor creates a synonymous fact ID.

Preserve the speaker, important negation, report time, known event time, identity and distinguishing cues, the owner/conditions/review point of unfinished commitments, and source IDs. Keep a necessary short quote, such as “worried I won't keep it up”; do not turn it into “already gave up running.”

If a summary is unverified or cannot retain critical fields, record `summary_unavailable`. A still-hot-eligible record may temporarily use its last valid detail view if it fits the budget. Otherwise exclude or cool it with a quality/capacity reason while keeping the original. Do not manufacture apparent smoothness by dropping a crucial qualification.

Restoring detail for a cold record still requires existing cue or promotion permission. Representation changes must not become automatic full-text reads that bypass the cold gate.

## 4. Selection under budget pressure

Filter permissions, current relevance, and independent budgets first. Prioritize necessary compact continuation information for relevant unfinished tasks or commitments. Then apply the configured within-group ordering (base heat order or the vector extension) to other records. Protocol-required active tool content still belongs in the bounded task allowance; a commitment is not unlimited prompt space.

Without the vector extension, use the deterministic ordinary-record ordering: relevant unfinished-task essentials first, then heat descending, latest substantive user mention descending, and ID ascending. Use the final three keys within a priority group too. With the [vector extension](vector-ranking.en.md), replace only within-group ordinary-event ranking with the combined relevance score. Heat-based transitions, unfinished-item priority, and the separate continuity-state allowance remain unchanged. Item and topic overflow requires whole-record selection in this order; compressing text cannot bypass item caps.

On overflow, first try a verified summary instead of a low-priority detail view. If it still does not fit, drop low-priority whole records and record an exclusion reason such as `capacity`. The 64-record active pool, 8-record request cap, 2-record topic cap, and token subbudgets do not change. With continuity enabled, hot remains 320/320/320 and total memory remains 1,600. Pin overflow still requires explicit pin management rather than silently dropping existing pins.

The continuity profile's 72-hour recent-candidate window is separate from representation stage. An ordinary event outside that recent window may compete as a short summary in the topical-facts portion if it remains hot-eligible and relevant. It does not automatically reserve a slot, and old events cannot hide in resident personality state. Replace and deduplicate by the same ID when changing portions rather than occupying both. Unfinished matters follow valid-ongoing or review rules.

An unfinished commitment that outlives the recent window or review deadline becomes `needs_review`, not completed. Actual reminders belong to the task system. This protects essential task state; it does not guarantee that every original phrase stays in every request forever.

## 5. What gradual transition can actually promise

The normal path prefers detail, then summary, then cold. It reduces abrupt cuts but cannot guarantee permanent detail recall. Capacity pressure, missing validated summaries, explicit deletion, or recall-only restrictions can interrupt that preferred progression. Record the reason instead of silently exceeding limits.

Maintenance is evaluated lazily at the next interaction. If the user returns after 15 days, the resulting stage can already be cold. The summary-eligible interval elapsed in real time, but the user did not necessarily see a summary. Do not replay unrelated old memories merely to stage a transition, or invent interactions during idle time.

See [cooling-scenarios.en.json](../examples/cooling-scenarios.en.json) for proposed acceptance specifications. They are implementation targets, not passed runtime or model tests.
