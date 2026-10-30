# Logical Partitions and Progressive Recall of Distant Memories

This file describes default cued retrieval. After explicit opt-in to [proactive mode](proactive-memory.en.md), the host may grant bounded access for the current interaction. The same indexes, permissions, provenance, and budgets apply. Background reflection has no cold-scan permission.

Keep the complete user-and-assistant dialogue in a host-managed cold archive, then build composable indexes over its messages or episodes by **type, time, place, person, object, and topic**. Every conversation remains stored even when it has no useful facet or separate fact card. Any sufficiently discriminating cue from the user can serve as the first entry point. Later turns can add constraints to locate the exact event. The user does not need to provide every field at once.

This is a specification to implement. Logical partitions can be database indexes and metadata filters; no physical disk partitions or immediate vector database adoption are required.

## 1. One body, multiple entry points

Consider a fictional exchange: “In October 2019, the user bought a film camera while traveling in Xi'an.” Store the original dialogue once, link a compact memory card to its message IDs, and let all facet indexes point to that same archived source or card. The archive keeps other conversational turns too, even those that never become cards.

| Entry point | Example index | User cue it can answer |
| --- | --- | --- |
| Type | experience / purchase | “when I bought something,” “that trip” |
| Time | 2019 / 2019-10 with month precision | “in 2019,” “that October” |
| Place | China / Shaanxi / Xi'an | “in Xi'an,” “in Shaanxi” |
| Person | Person ID or alias explicitly present in the source | “when I went with that person” |
| Object | camera / film_camera | “my camera,” “the film camera” |
| Topic | travel / photography | “on the trip,” “the photography thing” |

The dimensions have no fixed search order. “That time in Xi'an” starts with place; “my old camera” starts with object; “the 2019 trip” combines time and topic. The implementation can choose whichever available index narrows the scope most, without forcing every query through a type-then-time-then-place hierarchy.

Do not duplicate an entire conversation for each category, or send every tag, directory tree, and index entry to the conversational model. Indexes reside in host storage; the model receives only a few selected candidate cards or source excerpts. An untagged turn is still stored; authorized full-text or time-bounded search can find it when facet extraction missed it, within the same global candidate and recall limits.

## 2. Field contract

Add `facets` to each record; see the [synthetic data](../examples/memories.en.json). Every field requires source evidence. An optional extractor may propose fields; the host verifies them before indexing.

| Field | Rule |
| --- | --- |
| types | A bounded set of types per record, such as experience, purchase, preference, project, relationship, routine, or fact |
| occurred_time | When the event happened; inclusive ISO `start`/`end` dates with `precision=day/month/year/unknown`; use null for unknown and never infer it from record creation |
| places | Canonical place ID, readable name, hierarchy, and aliases; only supported places, retaining uncertainty for “nearby” or unknown locations |
| people | Person/entity ID and original name; do not merge different people just because their names match |
| objects | Object category or entity ID, such as camera; “camera” does not establish brand or model |
| topics | A few topic labels; canonical terms can map to aliases from the user's wording |

Allow at most four tags or entities per dimension and manage alias lists separately in the host. Do not turn every noun into an unbounded tag. `created_at` marks record creation and `last_user_mention_at` drives heat; neither is `occurred_time`. A query for “something that happened in 2019” uses event time, not the date on which the user mentioned it again.

Resolve relative time such as “last year” against the current user event's time and timezone. “Back then” or “a long time ago” does not identify a year: leave it vague rather than inventing a year filter. Put unknown event times in an explicit unknown bucket so place-only queries can still find them. With a precise year filter, unknown-time entries are not definite matches; after no result, ask whether the user wants to broaden the time range.

`owner_id`, `scope_id`, and `status` are permission and validity filters that must never be relaxed simply because no result was found. Other facets are query conditions. Change a user-supplied explicit condition only after a new user cue or explicit consent.

## 3. “Do you remember that time in Xi'an years ago?”

This is a fictional example of expected behavior, not an observed model run:

```text
User: Do you remember that time in Xi'an years ago?
  Cue: place=Xi'an; time=vague; type/object/topic=unspecified.
  Host: Filter valid records for the same owner and allowed scope,
        then query the Xi'an place index.
  Do not: Invent 2019 as the year, demand a year first, or read all Xi'an bodies.

If one strong match exists: Return only its necessary summary, preserving uncertainty.
If several matches exist: Return at most three very short candidate cards,
                           still within 480 tokens altogether.
  Card 1: 2019, Xi'an, purchased a film camera while traveling.
  Card 2: 2021, Xi'an, visited a photography exhibition with a friend.

Assistant: Do you mean the camera purchase or the photography exhibition?
User: The camera purchase.
  Host: Keep the unexpired place cue; add object=camera and type=purchase.
        This new cue permits one new query. The global candidate limit remains 12,
        not 12 per partition.
  Result: Return the matching camera-event summary and its source.
          Do not invent the camera model.

When the user changes topic: Remove cards and body excerpts from later outbound
context. The original record stays in cold storage.
```

An “explicit cue” needs to narrow the search meaningfully, not identify one unique event. A place cue can start the search. Only “Do you remember that time?” with no usable reference requires a cue before searching. If many candidates match, the host sets `truncated=true`; do not present three displayed cards as the full set. Ask briefly for a year, person, object, or event type instead of paging through the whole directory.

A candidate card contains only a memory ID, short title, known time, and the type or object needed to distinguish it. It uses the recall budget. Show only candidates that match the user cue and pass permission filters; do not invent options when there are no results. If an otherwise precise query gives conflicting answers, express uncertainty and ask for a clarifying detail.

## 4. Retrieval flow and hard limits

1. **Verify source and permission.** An actual user cue or a valid follow-up within the same topic can authorize recall. Background events cannot.
2. **Parse available cues.** Use rules and known entity aliases first; an optional cue router may process bounded input when necessary. Leave vague terms unset rather than manufacturing exact conditions.
3. **Filter indexes.** Apply owner, scope, and status first. Then combine supplied time, place, type, person, object, and topic facets. Verified synonyms within one dimension may be ORed; distinct explicit dimensions usually combine with AND.
4. **Rank bounded candidates.** Use database or inverted indexes. If semantic search is needed, confine it to the authorized scope and recheck filters before reading bodies. At most 12 candidates **globally** reach body selection; never take 12 from every partition and concatenate without limit. For a broad match, the host may retain a count or truncation flag, but never send the full list to the model.
   With the [vector-ranking extension](vector-ranking.en.md), the authorized semantic and exact-index routes may each return at most 12 host-internal references, or 24 combined. Fuse, deduplicate, and protect the strong-exact-evidence priority group before passing a global maximum of 12 candidates to body selection. Internal references do not mean reading 24 bodies or sending them to the model. If the priority group still exceeds the cap, mark truncation and ask for another cue. Cold ranking follows the current cue, with no event-age or last-mention decay and no newest-12 prefilter. Match explicit years against event time. Missing embeddings still allow exact-index retrieval without relaxing explicit constraints.
5. **Determine whether the event is unique.** Read necessary body content when the scope is clear. Otherwise show at most three very short candidate cards to invite another cue. Cards and bodies share the three-item / 480-token budget; they cannot each consume a separate allowance. Replacing a card with a body replaces the item rather than appending a duplicate ID.
6. **Insert a bounded summary.** Use atomic, source-backed summaries; count their time and ID wrappers. Drop entries first when over budget. A request for lengthy material must be a separate, explicitly requested task; do not automatically insert an entire ancient conversation as ordinary memory.
7. **Expire.** Remove temporary content after permission expires, a topic change, deletion, or correction. Qualifying mentions and heat still follow the main design; retrieval reads add no heat.

Run only one retrieval flow per explicit cue by default. That flow may use limited index lookup and ranking, but must not try unlimited combinations of relaxed filters automatically. A new user detail permits the next query. A miss never permits searching across owners/scopes or silently dropping an explicit place or time filter.

For a deliberately broad request such as “Give me a detailed review of all my Xi'an trips,” start a separate, visible paginated task. Each page still has a limit and reports what was covered and left unread. One broad authorization must not become permanent permission to browse the archive in later conversations.

## 5. Where token savings can occur

Database index filtering does not itself require sending bodies to the language model. The potential saving comes from narrowing scope first, reading a few results, then inserting only necessary summaries. Sending the full directory to the model to choose a partition, or calling a model for every partition, could instead increase token use and latency.

Keyword queries and local indexes do not directly spend main-model input tokens, but they do consume storage and compute. Count cloud embeddings, reranking, classification, and any model-based maintenance in the evaluation. The structure constrains what gets read; it does not establish a measured percentage reduction in cost.

## 6. Additional acceptance checks

- A place-only cue reaches the appropriate index; “years ago” is not silently converted into an exact year.
- Time, place, person, object, type, and topic can each start retrieval. Missing dimensions do not block available ones.
- Additional cues combine filters. Unrelated bodies from other partitions and the full index never enter the final request.
- A record found through multiple entry points keeps one `memory_id`, provenance set, and heat counter and is counted once against the budget.
- Aliases, identical names, unknown times, and unknown places are handled without guessing after a miss.
- Expected behavior for S23–S28 is in the [scenario data](../examples/scenarios.en.json); a structured example is in the [recall plan](../examples/recall-plan.en.json).
