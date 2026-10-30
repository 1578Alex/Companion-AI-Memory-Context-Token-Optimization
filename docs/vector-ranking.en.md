# Vector relevance, time decay, and Top N (v0.3)

Default permission comes from a user cue. Optional [proactive mode](proactive-memory.en.md) changes only host-validated trigger conditions, preserving provenance, age-neutral cold ranking, strong exact evidence, and budgets. Emotional similarity does not confirm the present cause.

The idea is simple: **find what matters to this conversation, then spend the limited context on the few records that help most. An old event must not lose its chance of retrieval because of its age, and the character's personality must not compete with ordinary events for survival.**

This is an optional specification awaiting implementation, with a [profile](../config/vector-ranking.example.json) and [acceptance cases](../examples/vector-ranking-scenarios.en.json). The embedding model, thresholds, weights, and half-lives are initial engineering choices without real companion-conversation results. These formulas are not a model of the human brain or measurements of feelings.

## 1. Give each event a semantic coordinate

An event unit remains a source-backed event card. “Bought a film camera while visiting Xi'an,” for example, keeps its time, place, people, object, topic, and source IDs, and gains a semantic vector. “That machine that took film” can then retrieve a candidate even without the word “camera.”

The host stores an embedding descriptor for each event:

| Field | Purpose |
| --- | --- |
| `embedding_ref` | Reference into host-managed vector storage; never an array pasted into the prompt |
| `embedding_model` / `embedding_revision` | Exact generating model and revision |
| `embedding_dimension` / `embedding_normalized` | Dimension and whether unit-normalized; validate before comparison |
| `embedding_content_hash` | Version of the source-verified event text, preventing use of an obsolete vector after correction |
| `memory_id` / `source_refs` | The same event and its original dialogue; multiple indexes do not duplicate the source |

A vector represents meaning, not importance or the depth of a feeling. An event has no permanent single “relevance level.” On each query, embed **the current user question and a bounded, valid recent referent**, then compare it with candidate events. Do not compute every event-to-event pair or send the whole archive to a model to form a query.

For unit-normalized vectors from the same model, revision, and dimension, use `S = clip(cosine(query, event), 0, 1)`. Negative similarity becomes 0. S is a ranking value, not a confidence probability. Numbers from different embedding spaces cannot be compared.

## 2. Heat and current relevance have different jobs

The [base design](design.en.md) keeps `heat = 0.45F + 0.35R + 0.20O` for ordinary hot eligibility and gradual cooling. A vector score cannot turn the cold archive into resident context or refresh a user-mention timestamp.

This extension replaces only the ordering of ordinary events **within the same priority class and event subbudget**. Permissions, validity, user restrictions, current relevance, necessary compact continuation of unfinished work, reserved continuity types, and budgets come first. An event unrelated to this question cannot enter merely because it was mentioned yesterday.

```text
F = min(qualifying user mentions in 30 days, deduplicated and daily-capped / 6, 1)
R = 2 ^ (-continuous days since the last qualifying user mention / 7)
O = 1 if user-confirmed ongoing and before its review deadline, otherwise 0

hot_score = 0.65*S + 0.20*R + 0.10*F + 0.05*O
```

This is a separate per-request ranker, not another recency term added to heat. While vector ranking is available, heat handles admission and lifecycle transitions; fallback restores the base hot ordering. R uses the last qualifying user mention, not the event date, embedding date, or retrieval date. A recall query alone, assistant repetition, retries, and summaries do not refresh it. Use continuous elapsed time; midnight, restart, and a new session do not reset it.

The example semantic threshold is `S >= 0.55` and needs calibration for the chosen model. A source-backed exact lexical/metadata match relevant to the question keeps an independent route and is not rejected because its embedding is missing or below this semantic threshold. Do not add incomparable raw scores from different routes. Without reliably comparable vectors, use the fallback below rather than inventing S.

## 3. Cold recall does not mean “newer is better”

For “Do you remember buying that camera in Xi'an years ago?”, first obtain the existing user-cue permission, then narrow the archive using Xi'an, purchase, camera, and other supplied clues. **An event from ten years ago may rank first. Neither its age nor the time since its last mention incurs decay.** Unrelated events from yesterday are excluded first.

```text
cold_score = 0.80*S + 0.20*C
```

C is verifiable matching of the user's current clues. Check only informative, supplied types, times, places, people, objects, topics, an explicit event name, or an original quote: a verified match is 1, unknown is 0, and C is their mean. Verified aliases or a location hierarchy satisfying the requested place may count as 1; the model must not guess missing evidence. C is not a probability. In default mode, with no sufficiently locating cue, stop at the cue gate rather than using zero scores to scan everything.

With a valid host permit in v0.4 proactive mode, an emotional entry such as “I'm exhausted” may supply none of these verifiable cue dimensions. Explicitly set `C=0` in that case, rank within the permitted topics using the same formula, and do not renormalize. Neither the permitted scope nor candidate content can become user-supplied clues or a strong exact match. Without a valid permit, do not search.

Explicit user constraints are filters, not penalties that high semantic similarity can overcome. Exclude known conflicts. “The 2016 Xi'an trip” uses the event's occurrence time, never the last chat time. With a precise year, an unknown event date is not a confirmed match; after a miss, ask whether to relax the date. “Years ago” does not invent a year, and a place alone can start retrieval. Unprovided dimensions are not missing-data penalties. These rules follow the [logical-index design](cold-storage-index.en.md).

Cold ordering must not use event age, time since mention, frequency, or heat as bonuses, thresholds, tie-breakers, or pre-truncation rules. Apply the strong-exact group below first, then score within each group; break equal scores by stable ID. Accurate old events therefore are not suppressed by time decay. Approximate search, missing tags, and ambiguity can still cause misses, so this is not a guarantee of successful recall every time.

## 4. Bounded candidates, then Top N

1. **Check scope first.** Filter owner, scope, status, and user restrictions; cold search needs current user-cue permission or a bounded current-interaction permit from explicitly enabled proactive mode. Pre-filter in the index where possible and recheck before returning text. Vectors cannot bypass access gates.
2. **Two bounded retrieval routes.** Within the authorized scope, semantic search and metadata/lexical search may each return at most 12 internal candidates. The host bounds index work and timeouts. This is not 12 per place or partition and not all-pairs computation. Internal index probes are not extra model calls and do not authorize reading every body.
3. **Fuse without preferring new events.** The routes produce at most 24 internal references. Deduplicate by ID/source and return **at most 12 candidates globally** to body selection. Select the evidence-verified `verified_strong_exact_match` group before ordinary candidates; use cold_score or cold fallback ordering within each group. A strong exact match needs a user-supplied explicit event ID, a uniquely locating original quote, or at least two independent, discriminating user-supplied cue dimensions all verified without conflicts. One broad city/topic, or C=1 alone, is insufficient. Preserve this group priority both at the fused 12-candidate cap and final cold Top N, so higher semantic scores alone cannot evict a strong exact candidate with lower S. If more than 12 strong exact matches remain, set `truncated=true`, return only the permitted tiny candidate view, and ask for another clue; never present Top 1 as a unique answer. Never take the “12 most recent” before semantic search or give two lists of 12 to the main model. If any cold candidate has a missing or incompatible vector, uniformly switch the batch to deterministic cold fallback ordering (strong exact group first, then cue match, one normalized host lexical score, stable ID). Do not assign missing vectors zero or mix vector scores with lexical scores. Strong-exact priority cannot bypass permissions, explicit filters, or ambiguity checks.
4. **Verify and return little.** Check source, negation, time, and identity. Inject one representation per event, not three paraphrases filling three slots. Ordinary hot events remain capped at two per topic; do not merge distinct events merely because they occurred at different times on a similar topic. When several events remain ambiguous, show the existing bounded candidate cards and ask for a cue rather than treating Top 1 as a confirmed answer.
5. **Top N within budgets.** N is a ceiling, not a quota: at most 8 hot records / 960 tokens, and 3 recalled records / 480 tokens, including labels, sources, and framing. Use verified short summaries first, then fewer records; never cut negation or factual qualifiers. Select in the applicable order among records that fit completely. A later, valid short record may replace an oversized one, with the reason recorded. Empty space cannot be borrowed by another bucket. Give the main conversation model only necessary text, not vectors, the directory, or all candidates.

Hot ordering scans only the bounded hot candidate pool of at most 64 records. An ordinary greeting needs no cold index query. The two cold search routes belong to one bounded retrieval flow, not automatic filter relaxation or repeated investigation. Approximate vector search provides Top N among its candidates; it does not prove a mathematical global optimum across the archive.

## 5. Personality and recent state keep their own place

A companion's continuity cannot depend solely on event scores. Its stable personality remains in the role prefix. With the [continuity extension](persona-continuity.en.md), the 960 hot tokens are divided as follows:

| Content | Independent allowance | Can vector-ranked events evict it? |
| --- | --- | --- |
| Current personality, interaction habits, and recent-state continuity | 1 record / 320 tokens | No; restore the current committed version |
| Recent episode details | Up to 3 records / 320 tokens | Select within this portion, preserving necessary recent continuation anchors |
| Current topical facts | Up to 4 records / 320 tokens | Select within this portion |

The total stays at most 8 records / 960 tokens, with no borrowing and deduplication across portions. Select current state by its typed exception, outside ordinary similarity, heat, 14-day inactivity cooling, and pool eviction. It cannot hide old episodes to bypass cold gates.

Before the first reply after restart or on the next day, restore the current state with a compatible authorized persona version. Familiarity and slowly changing style do not return to first-meeting defaults each day; short-term mood gradually moves toward the current relationship baseline over continuous elapsed time. Embedding updates, a changed Top N, or a single recall cannot rewrite personality. Preserve natural emotional change without inventing shared experiences or forcing old events into every reply for realism.

## 6. Missing data, updates, and cost

- **No embedding must not mean no recall.** Keep keywords, aliases, and logical indexes as a fallback. When vectors are unavailable, missing, or incompatible, mark the degraded mode and uniformly switch the candidate batch rather than mixing scores. Hot events first follow existing relevance, task continuity, type, and subbudget rules, then restore the base within-priority ordering: descending heat, descending last qualifying user mention, ascending ID. Cold events keep the strong-exact priority group, then sort by C, uniform lexical score, and ID, still without recency. Do not discard accurate exact matches, delete the archive, or expand permissions.
- **Process increments.** Embed new or corrected source-backed event text. Retrieval does not re-embed the archive. Compute the query vector once per valid query, with short-lived caching that does not cross permissions. Clear it on expiry or revocation.
- **Do not mix versions.** Comparable vectors require an explicitly configured model. Reject direct comparison across model, revision, or dimension mismatches. Build a new index incrementally/offline under a host-controlled budget and switch atomically after validation. Keep lexical fallback available and preserve originals. Never merge scores from two embedding spaces into one ranking during migration.
- **Corrections and deletion apply immediately.** Exclude superseded versions by default and invalidate stale vectors, indexes, derived summaries, and query/result caches. Deletion also removes corresponding vector data and originals. Historical access still follows the base validity and user-permission rules; an old embedding cannot resurrect it.
- **Count the whole cost.** Local index computation does not consume main-model input tokens, but embeddings, reranking, storage, latency, and auxiliary model charges belong in the evaluation. A shorter main request does not prove a lower total bill.

Acceptance must check that decay does not bury old events, freshness cannot admit irrelevant ones, the same character continues across days, and actual input stays bounded. The cases' expected fields describe requirements, not completed model evaluations.
