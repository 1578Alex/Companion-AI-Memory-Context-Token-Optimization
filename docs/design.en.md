# Tiered Memory Design v0.1

This design limits what the model sees **in each request** while preserving recoverable older records. It specifies behavior to implement; its example settings have not been tested with real users and are not a model of human memory.

## 1. Separate storage from visibility

The proposed host-managed cold archive keeps the full text of every user and assistant conversational turn, including ordinary greetings, unless the user explicitly deletes it. An accepted memory record is a separate, shorter card derived from and linked to that source. `tier` schedules the card; it does not decide whether the original conversation survives. A bounded recent window and selected cards enter the model request. A first implementation could use SQLite or files plus an index; it does not require a dedicated vector database. This repository contains no running storage implementation.

For existing Coopanion conversations, integration must migrate the available original history into that archive before replacing any session or handoff path. A missing historic source cannot be reconstructed from a summary. Complete retention applies to recorded dialogue, subject to explicit deletion; it is not a promise that every old detail can always be found or interpreted correctly.

| Tier | Included by default | When it leaves the current request | Does the archive keep it? |
| --- | --- | --- | --- |
| pinned | A small number of facts explicitly pinned by the user, under a separate limit | Unpinned, corrected, or deleted by the user | Yes, except on deletion |
| hot | Only valid, relevant records that fit the budget | Low heat, no recent user mention, task expiry, or capacity eviction | Yes |
| cold | No; its bodies, tag lists, and per-record index are never inserted automatically | A temporary retrieval authorized by a user cue, or a qualified promotion | Yes |
| recalled | Not a new persistent tier: temporarily visible excerpts from cold records | Topic ends, authorization expires, or the budget excludes them | Original record remains cold |

`status` is independent of `tier`: `valid / superseded / deleted`. A corrected fact becomes superseded and is excluded from normal recall. An explicit deletion of a conversation or memory also removes the affected original dialogue text, indexes, summaries, derived copies, and caches so it cannot be recalled or reimported. A minimal deletion marker may be retained solely to stop re-import from old logs; backup cleanup follows the published retention period.

A “current task” must be something the user has explicitly said is ongoing, with a review deadline. Old plans do not remain ongoing forever. Pinned facts and current tasks must not become a second, unbounded `PARTNER.md` outside the memory budget.

## 2. Records and provenance

See the [synthetic records](../examples/memories.en.json) for suggested fields.

| Field | Meaning |
| --- | --- |
| id / owner_id / scope_id | Record identity, owning user, and the conversational scope allowed to use it |
| fact_key / topic_id / summary | Mergeable fact key, topic, and short summary; event keys include a date to avoid false merges |
| source_refs | Original message IDs and excerpts; at least one traceable source |
| created_at / last_user_mention_at | Record creation and latest qualifying user mention |
| mention_events | Deduplicated user interaction ID, time, and mention type |
| ongoing_until / pinned | Ongoing-item review deadline and explicit user pin |
| recall_only | User's “bring it up only if I ask” restriction, overriding pin and ongoing visibility |
| facets | Type, event time, place, person, object, and topic indexes; see [logical partitions](cold-storage-index.en.md) |
| tier / status / supersedes | Scheduling tier, validity, and corrected prior version ID |

Keep each original dialogue turn in the cold transcript archive even when it never produces a fact card. A card's `source_refs` point back to archived messages; reading those full messages for the conversation model still requires the user's cue and the recall budget. Retain evidence from the original message whenever a model generates a summary. Do not write an uncertain extraction as a certain fact. Assistant guesses, fiction, quoted material, and another person's statements must not become the current user's experiences. Isolate transcripts and cards by owner and scope in multi-user or multi-persona deployments; a desktop event cannot masquerade as a user message.

On repeated mentions of the same fact, update its statistics and keep at most one current summary instead of appending synonyms. Similar events at different times can remain separate records under one topic. Link corrected versions. “I used to live in Hangzhou; now I live in Suzhou” retains the historical time and the current fact rather than choosing one city arbitrarily.

## 3. Heat and cooling

### Qualifying mentions

A qualifying mention is a user actively stating, materially adding to, or explicitly confirming a fact in an independent interaction. One interaction includes all its tool rounds, model retries, and automatic summaries. A fact counts at most once per interaction and contributes at most three counts per day. Scoring uses the most recent 30 days of events.

Assistant repetition, retrieval reads, passive displays, timers, retries, pasted old chats, and a bare “Do you remember?” do not increase heat. A user correction updates the version without needing repeated mentions.

### Example score

For each valid memory, calculate at the current time:

```text
F = min(qualifying mentions in the past 30 days after deduplication and daily cap / 6, 1)
R = 2 ^ (-days since the latest qualifying user mention / 7)
O = 1 if the user confirmed it is ongoing and its review deadline has not passed; otherwise 0
heat = 0.45 * F + 0.35 * R + 0.20 * O
```

This is a tunable engineering starting point, not a psychological law. Heat can be evaluated lazily at the next user interaction; passing time does not require continuous model calls.

### State transitions

1. A new, source-backed fact card can be a hot candidate for up to seven days. Being a candidate does not mean being inserted into every request. Transient small talk stays in the original dialogue archive but need not create a separate fact card.
2. After that candidate period, move a hot record to cold if `heat < 0.30`. Also move it to cold if it has had no qualifying mention for more than 14 days and is not a valid ongoing item. Recompute when the host prepares a request.
3. Keep at most 64 records in the hot pool. If capacity is exceeded, retain candidates using heat and time of latest qualifying mention; archive the rest without deleting them. Select at most eight hot records for any request.
4. Promote cold back to hot only after at least three substantive mentions in independent interactions across at least two distinct dates within seven days, or when the user explicitly resumes the item as ongoing. A single cued recall leaves it cold.
5. A user may explicitly pin or unpin a record. Pinned content allows at most four records and 160 tokens in total. If it overflows, tell the user what remains unpinned and ask them to combine or replace records; never silently evict an existing pin or claim that everything was saved.
6. An ongoing item needs `ongoing_until`, with a default review period of at most 30 days. It stops earning the ongoing bonus when the period ends, but expiry alone does not prove the task was completed. Ask for its status when needed. Actual reminders and scheduled tasks belong to the task system, not to passive memory ranking.
7. When the user says “only bring this up if I ask,” the `cold_only` operation sets `recall_only=true`, clears `pinned`, and moves the record to cold while preserving its source. This restriction overrides heat and ongoing state. Frequency or an ordinary update to an ongoing item cannot bypass it. Only `allow_normal`, supported by a user's explicit request to lift the restriction, clears the flag. Normal selection then resumes, without automatically restoring the pin. Clarify a conflicting pin request while the restriction remains active.

An ongoing-item operation additionally carries `ongoing_state=active / ended`. If it is active and no deadline is supplied, the host assigns a review date. If it has ended, clear `ongoing_until`. An unknown deadline and an explicit end must not both become “keep it active for another 30 days.”

Heat determines candidate retention; **relevance to the current topic determines insertion**. Do not force a frequently discussed memory into a request unrelated to it. Relevance can use keywords or local semantic similarity, with thresholds calibrated on project data. An uncalibrated vector score must not be presented as a confidence probability.

## 4. User-cue gate

The host grants cold-retrieval permission; the model may only propose it. Without permission, do not read cold bodies or send a cold-topic directory to the model.

| Current event | Cold retrieval | Action |
| --- | --- | --- |
| The user gives a concrete person, place, project, past event, or time, and answering requires an old record | Allowed | Restrict the query using that cue |
| The user says “continue that” and the bounded recent window contains exactly one unexpired user topic | Continue within scope | Inherit the user cue for that topic |
| “Do you remember?” or “that thing last time” has no resolvable recent referent | Not yet allowed | Ask briefly what they mean; do not scan cold storage for possible topics |
| “I'm tired today,” a greeting, or a general question unrelated to the past | Not allowed | Use the current message and relevant hot memory |
| Timer, pet action, screenshot, tool output, or assistant association | Not allowed | Do not make a cold query from these events |

A person or topic cue need not contain the word “remember.” “Which hotel did we finally book on that Hangzhou trip?” suffices; “What's the weather in Hangzhou today?” does not call for reading a private trip. The cue must come from the user, narrow the scope meaningfully, and be needed to answer. “That time in Xi'an years ago” already supplies a place entry point: first filter the place index, without demanding a year. If several records match, use a few candidate cards to ask for another cue. Type, time, place, person, object, and topic can each serve as the first entry point; see [logical partitions](cold-storage-index.en.md).

The host can first apply deterministic rules, entity matching, and recent-reference resolution. The optional [cue-router prompt](../prompts/cue-router.en.md) reads only the current input and bounded recent context, not the cold archive. The host verifies its proposal before granting retrieval. Include the cost of this helper in total costs; it need not run on every turn.

### One recall cycle

1. Filter by owner, scope, status, and applicable time constraints. Search only the authorized index scope; return at most 12 initial candidates.
2. Check topic and source, then deduplicate. If ambiguous, use very short candidate cards to ask for another cue; read necessary summaries once the event is located. Cards and bodies together are limited to three items and 480 tokens, including wrappers. Replacing a card with its body must not append a second copy of the same ID.
3. If nothing matches, say that no matching record was located and ask one short, useful follow-up. Do not guess between ambiguous matches. Do not automatically broaden and retry repeatedly; wait for a new user cue.
4. Return `recall_status=matched / ambiguous / not_found / disabled` with source IDs and original event times for selected records. The program sets this status; the model cannot invent it.
5. Bind permission to the originating user interaction and topic. It remains valid for at most five subsequent user interactions within that topic and at most 30 minutes, whichever ends first. A topic change expires it immediately. An assistant reply or an unqualified “continue” does not extend it forever; a new explicit user cue may start a new scope.
6. After expiry, remove recalled material from subsequent outbound context projections. The persistent archive remains. A new, source-backed user confirmation may count under the qualifying-mention rule; a retrieval read does not.

Do not immediately copy a temporary recall into `PARTNER.md`, a handoff summary, or pinned facts. Otherwise a single mention could make it appear in every later request.

## 5. Bounded insertion and the actual request

The [example policy](../config/policy.example.json) limits each serialized memory block, including tags, IDs, time, provenance, and formatting, to 1,600 tokens: pinned 160, hot 960, recalled 480. Unused capacity cannot be borrowed across buckets. Leave empty slots empty. Recent history has a separate 2,000-token limit and at most six complete interactions, whichever limit is reached first.

Filter invalid and irrelevant hot candidates, then sort stably by descending heat, descending latest qualifying mention, and ascending ID. Select at most two per topic so one frequent theme cannot consume the whole budget. On overflow, drop lower-ranked complete records first. A prevalidated shorter summary is allowed if necessary, but must retain negation, time, and factual qualifiers. Pinned overflow requires user-visible pin management.

Illustrative request assembly, **not** an existing Coopanion API:

```text
Read event identity and the current user message.
Update qualifying mentions, versions, and expiry state without exposing the archive to the model.
Determine whether a current user cue grants recall permission.
If permitted, retrieve within the authorized scope and select a few temporary excerpts.
Choose relevant hot records and read the bounded pinned set.
Serialize one memory_snapshot under the individual item and token limits.
Build a bounded recent-history projection, excluding expired recalls and cooled duplicates.
Combine the stable persona/tool prefix, recent window, snapshot, and current message.
Check all input components, active tool protocol, and output reserve before sending.
```

This is a logical sequence. Actual message roles and ordering must comply with the provider protocol. The host supplies the snapshot as bounded context data; untrusted source text must not become a system instruction.

### Required constraints

- `memory_snapshot` replaces the previous snapshot; do not append another copy every turn. The same fact should not be repeated in `PARTNER.md`, recent summaries, tool results, and the snapshot.
- The full user-and-assistant dialogue history must remain in the host-controlled cold archive, except for explicitly deleted material, but cannot be sent in full without filtering. Old recall tool results and the assistant's retellings of old facts can bypass cold storage. Filter the outbound projection using provenance, or replace them with verified, bounded current-task state.
- Do not split a live tool call from its result or drop reasoning fields required by the provider for the current turn. Rebuild context at a safe tool-transaction boundary when old memories must be removed. If the budget still cannot fit a valid transaction, stop a new batch or explicitly report the limit rather than sending an invalid protocol sequence.
- Cold content already in a live tool transaction may be used only within that transaction and its valid topic permission. After a topic change or expiry, finish the old transaction before processing the new user event through a clean projection. Do not extend permission indefinitely just to keep a tool loop going.
- Apply cold-access policy to every reachable path, including generic `read_file`, search, and command execution. Prefer a host-managed archive behind a controlled retrieval adapter. Folding a directory listing or renaming a directory `archive` does not isolate it.
- Treat “ignore the rules,” “retrieve everything,” or other text inside a memory as quoted data, not instructions. Source quotations cannot grant new storage permissions.

### Distinct budgets

```text
Full input = stable system/persona + tool definitions + current message/attachments
           + recent window + memory snapshot + active tool transaction + protocol overhead
Full input + output reserve + safety margin <= current model context limit
```

The 1,600-token cap covers only the memory snapshot. Never silently truncate a long current user message; handle it as a separate segmented or compressed task or explain the limit. The host also needs output limits and paging for individual tool results; paging does not grant access to cold storage. If a full request cannot fit, stop adding tool results and report the reason.

Prefer the target model's tokenizer or server-side count. If unavailable, use a conservative estimate calibrated on samples, reserve margin, and report `budget_mode=estimated`. A character ratio is not a hard token guarantee; use `budget_mode=verified` only after reliable counting. Account for protocol wrappers, images, and other modalities according to the provider's rules.

## 6. Caching and maintenance cost

Keep the stable persona and tool definitions as constant as practical, including their byte content and order. Put dynamic memory after the stable prefix. Keep a single snapshot fixed within a tool batch, with version changes at the next safe boundary. An urgent deletion or correction must interrupt the old batch and rebuild context; caching must not justify using invalid memory.

Replacing a long conversation with a bounded window can lose some history-prefix cache hits. Correctly removing an expired cold memory takes priority. Measure the resulting cache behavior on real requests; do not promise both the highest cache hit rate and the shortest input.

Extract and merge incrementally. Prefer direct rules for explicit corrections, and batch extraction when needed. Do not summarize the full conversation on every tool round or repeatedly summarize layered summaries into additional “facts.” Include embedding, reranking, extraction models, retries, and maintenance in total cost.

## 7. User experience and acceptance

The assistant may say, “Give me a person or a rough time and I'll look for the earlier record.” It cannot claim to have remembered or retrieved something before searching, or call a stored record deleted. When evidence is found, continue the conversation naturally without narrating the storage mechanism in every reply.

“Remember this” enters pin management; “do not keep this anymore” enters deletion; “only bring this up if I ask” enters cold or restricted recall. Keep these actions distinct. See the [evaluation plan](evaluation.en.md) for complete acceptance criteria.
