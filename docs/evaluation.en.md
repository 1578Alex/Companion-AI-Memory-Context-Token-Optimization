# Acceptance and cost evaluation plan

This repository specifies tests to run. It has not run Coopanion, measured model calls, or measured savings. The `expected` fields in [the scenarios](../examples/scenarios.en.json) are requirements, not passed test results.

## 1. Measure the requests actually sent

In a test environment, capture a redacted snapshot of each completed request immediately before it is sent to the provider. Aggregate every model call, tool round, failed retry, maintenance extraction, and retrieval call by user interaction.

Record at least:

| Category | Fields |
| --- | --- |
| Environment | Coopanion and Cortico commits, model, provider, policy version, token-counting mode |
| Correlation | Anonymized owner and scope IDs, `interaction_id`, `request_id`, `attempt_id`, trigger source |
| Input composition | Stable persona, tool schemas, recent window, pinned, hot, recalled, current message/attachments, active tool transaction, protocol overhead |
| Provider usage | Actual input/output, cache hits/misses, reasoning tokens where available, failed-attempt usage and billing evidence |
| Maintenance | Extraction, reranking, embedding calls and tokens; retrieval latency; storage growth |
| Behavior | Cue source, selected record IDs, missed and ambiguous searches, tier changes, budget overflow behavior |

Locally estimated component tokens help attribute cost; provider usage is the source to reconcile billing. Do not force imperfect estimates to sum to a provider invoice. Images, screenshots, multimodal content, and protocol wrappers need the appropriate counting method. Public reports should default to IDs, lengths, and categories rather than private memory bodies.

## 2. Behavioral acceptance criteria

| Requirement | Synthetic scenarios | Inspect |
| --- | --- | --- |
| No cold body read or directory exposure without a user cue | S01/S03/S04/S12/S15 | Retrieval calls, tool access, final outbound request |
| A cue can recover a record; no hit is reported honestly | S02/S10 | Provenance and answer accuracy |
| Expired recall does not survive topic changes or session restoration | S05/S13/S22 | Difference between stored raw log and outbound projection |
| Model repetition, retries, and duplicate extraction do not raise heat | S06/S20 | User mention count and tier changes |
| High-heat records still obey item, topic, and token limits | S07/S14 | Fully serialized snapshot and visible pin overflow |
| Cooling preserves sources; explicit deletion prevents future retrieval | S08/S11 | Archive, index, derived summaries, cache, re-import |
| Correction, isolation, and low-trust content rules hold | S09/S17/S18 | Version and permission filters, instruction-in-data handling |
| Tool transactions stay valid without silent over-budget requests | S16 | Provider message constraints and budget handling |
| Promotion needs real evidence; “only mention when I ask” has priority | S19/S21 | Effective user mentions and `recall_only` |
| A single place cue locates a range, then a new cue narrows it | S23/S24/S26 | Facet index, candidate cards, actual query conditions |
| Event time differs from storage time; facets do not copy records or multiply budgets | S25/S27/S28 | Date filter, ID deduplication, global candidate and card cap |
| Visibility restrictions take priority; restriction removal and task completion are expressible | S29/S30 | `recall_only`, pinned, ongoing status, and source |
| Keep both sides of every conversation, including turns without a card; clear originals and indexes on explicit deletion | S31 | Cold archive, uncarded chat, outbound request, deletion result |

Additional edge cases: exactly reaching a token cap, one oversized record, scores on either side of a heat threshold, 7/14/30-day boundaries, cross-timezone dates, an oversized current user message, extractor failure, idempotent retries, deletion during an active tool exchange, and expiry of an ongoing task's review date. Use a controllable clock instead of real waiting.

Only a reliable model-specific tokenizer applied to the actual outbound request can support a claim that a hard token cap passed. Otherwise report item limits separately from estimated token use that still needs calibration. Synthetic data checks cannot prove semantic recall quality on real conversations.

## 3. A/B comparison

Compare identical anonymized or synthetic conversation scripts with the same model, tools, output limit, and sampling settings:

- **A: current Coopanion.** Pin its version, preserve defaults or documented user settings, and record actual handoffs and cache behavior.
- **B: bounded recent history only.** Keep the same persona and tools, but restrict the history window without tiering or cue retrieval. This separates savings caused by shortening history from additional effects of the memory policy.
- **C: this proposal.** Bounded recent history, selected hot memories, recoverable cold archive, user-cue gating, and incremental maintenance.

Within C, separately disable heat ranking, cue gating, or protection against model-driven reheating to estimate each component's contribution. Do not change to a cheaper model, remove tools, or disable reasoning at the same time and attribute the entire difference to memory.

Measure both cold-cache startup and steady operation with warm cache. Record the cache conditions that can be controlled. If provider cache state cannot be held identical, report that limit and repeat the replay. Interleave groups and report sample counts, averages, and distributions so that a single conversation does not determine the result.

## 4. Metrics and billing

```text
total cost per user interaction = provider charges for every request in that interaction
                                + attributable extraction/embedding/reranking/maintenance charges

cost change = (baseline total cost - proposal total cost) / baseline total cost
```

If tokens are available but verifiable prices are not, report token changes, not a monetary reduction. Separate failed attempts with unknown billing instead of assuming all are free or all are charged. Apply the provider's current rules to cache discounts; a high cache-hit rate does not mean zero cost.

| Dimension | Metrics |
| --- | --- |
| Cost | Total input and output, request count, actual charges per interaction; maintenance cost share |
| Long-session growth | Snapshot and complete-input length after 10, 100, and 1,000 user interactions; archive storage growth |
| Recall quality | Retrieval accuracy on labeled answerable questions, factual accuracy, unsupported detailed answers |
| Unprompted intrusion | Rate of mentioning cold events without a user cue; wrongly associated events |
| User burden | Additional clues requested, unnecessary clarification, old-topic residue after a topic switch |
| Performance | Median and P95 time to first useful response and to completed response |
| Persistence | Cold-archive integrity, correction success, residual or re-imported records after deletion |

Set acceptable quality thresholds before comparing cost. Hard constraints include zero unauthorized cold lookups, zero returns of deleted content, no pinned/hot/recalled budget violations, and zero tool-protocol errors. Set targets for semantic recall accuracy and clue requests using labeled samples; do not promise percentages before data exists.

## 5. Report template

| Measure | A: current | B: recent window | C: tiered memory |
| --- | --- | --- | --- |
| User interactions measured | Not measured | Not measured | Not measured |
| Total input/output per interaction | Not measured | Not measured | Not measured |
| Actual cost per interaction | Not measured | Not measured | Not measured |
| Cache hits and maintenance overhead | Not measured | Not measured | Not measured |
| Old-event recall accuracy | Not measured | Not measured | Not measured |
| Additional clues requested on average | Not measured | Not measured | Not measured |
| Unauthorized cold lookups | Not measured | Not measured | Not measured |
| P95 completion latency | Not measured | Not measured | Not measured |

Every report must include versions, data sources, replay method, cache conditions, and missing data. This policy may reduce the memory inserted into requests, but total tokens and cost also depend on tools, reasoning, number of rounds, and caching. The experiment determines the real result.

## 6. Additional continuity comparison

When using the [continuity extension](persona-continuity.en.md), compare base C with C2 using this profile, keeping the same 1,600-token memory allowance, model, character version, tools, and sampling settings. Record hot subbudgets, the smaller topical-fact allowance, and maintenance costs too.

Replay [P01–P14](../examples/continuity-scenarios.en.json) for cross-day restoration, restarts, gradual change, recent details, corrections, deletion, and concurrency. Numeric state transitions can be tested deterministically; use separate blind evaluation for the model's character continuity and detail accuracy. A controller calculation is not a measured model result.

Record whether the first reply restores the current version, preserves recent wording and unfinished matters, changes tone abruptly, respects immediate boundaries, or changes personality after one cold recall. Measure actual tokens and costs. The goal is natural continuity with change, not identical tone in every situation. The sample parameters and intended behavior have not been validated with real-model experiments.

## 7. Additional graduated-cooling acceptance

Use [cooling cases C01–C06](../examples/cooling-scenarios.en.json) to check detail, summary, and cold-stage order; necessary continuity for unfinished matters; summary accuracy; capacity compression and exclusion reasons; lazy evaluation after long absence; and priority for corrections, deletion, and recall-only restrictions. Keep the existing S08 expectation of cold storage after 15 days. A gradual transition must not weaken the cold-access gate.

Beyond deterministic state-machine checks, blind evaluation with real model replies must assess abrupt loss of recent details, whether summaries support natural continuation, and whether the next day feels like the same companion. Consistent documents and parameters do not establish improved experience.
