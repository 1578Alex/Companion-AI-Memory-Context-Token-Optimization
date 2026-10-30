# Companion AI Memory Context and Token Optimization

**中文：[陪伴 AI 记忆上下文 token优化](README.md)**

I recently saw the “爸爸想给你完整的生命” (“Dad wants to give you a complete life”) story about Da Fei Yu, the AI companion ([Coopanion](https://github.com/Pal-AI-Lab/Coopanion)). As conversations grow, the context overflows, memories get lost, and huge amounts of tokens are used. So I designed a way for companion AIs to keep their complete memories while saving tokens.

## How it works

The idea draws on human memory and ordinary storage optimization: divide memory into hot and cold areas.

Recent, frequently mentioned matters, the core personality, interaction habits, and recent state stay on the hot side and enter the current context. The core personality and established interaction baseline remain available; specific events are selected in small amounts when useful for the current exchange.

As an old event comes up less often, its detailed representation first becomes a short summary, then gradually leaves the context for the cold archive on disk. All original dialogue is archived in full from the start. Optimization should preserve recent details and unfinished threads so that the transition feels natural, rather than making the companion feel like a different character overnight or after a session change.

When the user later mentions a relevant person, time, place, object, or topic, those clues guide retrieval from the cold archive. The saving comes from avoiding repeated reading of old dialogue on every request.

## Remembering on her own, without bringing up everything

This update adds an optional proactive companion mode. When you say “I'm exhausted,” she can remember that you previously mentioned overtime and ask, “Is it work again today?” If the link is weak, you do not want to discuss it, or she has just asked, she can leave it unsaid.

Real events and her own guesses stay separate. “I'm a little worried about him” can be a thought; it must not gradually become “Something has definitely happened to him.” Reflection can help organize interaction habits, but repeated guesses must not turn her into a different character.

The core personality, familiar voice, and recent details stay available. Older conversations remain archived, while each request includes only a few relevant excerpts. Background reflection skips when there is nothing new and has both call and token limits, so savings in conversation are not spent on idle work.

The default still requires a user cue for old memories; proactive mode needs explicit opt-in. This release updates the proposal and prompts, with no running integration yet. See the [proactive recall and reflection design](docs/proactive-memory.en.md) and [prompt](prompts/proactive-companion.en.md).

## Three ground rules

1. **Keep every conversation; cool down low-frequency events gradually.** Archive every original turn. Less-mentioned events leave the active set through “details → summary → cold archive.” Folding an index away is not enough: old message bodies and expired recalls must also be removed from the actual model request.
2. **By default, bring old memories back when the user gives a clue.** “That café in Hangzhou last year” can start a search. An ordinary greeting, a timer, a tool result, or the model's own association cannot. When the clue is too vague, ask the user for one more detail.
3. **Even frequent memories have limits.** Count only qualifying user mentions. A retry, an assistant repeating itself, or an automatic summary does not make a memory hotter. Even a frequently mentioned topic enters the request only when relevant and within the item and token limits.

## Still the same companion the next day

Part of what makes Da Fei Yu feel real is its familiar personality. Remembering facts is not enough if its tone suddenly changes and yesterday's state disappears the next day.

An additional continuity state keeps the base personality present, lets interaction habits change slowly, and carries recent tone, unfinished threads, and two or three concrete details across days before they gradually fade. Old events and historical state versions move into the cold archive; the current personality and relationship baseline stay available rather than resetting at midnight or restart.

This is still an optional design awaiting implementation. It reserves 320 tokens for continuity state, 320 for recent details, and 320 for relevant facts within the existing 960-token hot allowance. The total does not grow. Host code must actually save, restore, and limit the changes; a prompt saying “stay consistent” is not enough. See [personality and recent-state continuity](docs/persona-continuity.en.md), the [supplementary prompt](prompts/persona-continuity.en.md), and the [profile](config/continuity.example.json).

In implementation, the base persona stays in the stable character prefix while recent changes use a bounded hot-memory snapshot. Both enter the current context and count toward the full input.

## A gradual transition without an abrupt break

Recent episodes first retain distinctive details: who said something, the wording, time, and next step. When they are mentioned less often, keep a short summary that can still support continuity before moving them to cue-only cold storage. Details and summaries are different levels of compression within hot memory. The current topic and unfinished commitments receive priority; a clock boundary alone must not remove the whole thread.

The core personality and current interaction baseline do not cool with old events. Recent state changes gradually while historical state versions are archived. Every stage remains bounded by capacity, and detail selection and summary reduction must happen before the actual model request. These rules still require implementation and validation. See the [graduated-cooling design](docs/graduated-cooling.en.md).

## Finding older memories by clue

The cold archive keeps the full text of every conversation, including everyday chat that never becomes a separate memory card. It then builds logical indexes by **type, time, place, person, object, and topic**. The original conversation is stored once; each category is another way to find it.

For example, the user might suddenly ask, “Do you remember that bowl of ganmianpi (a Shaanxi noodle dish) we ate in Xi'an years ago?” The AI searches the relevant memories using “Xi'an” and “ganmianpi.” The wait while it searches can even feel like a person trying to recall the distant past. If several memories turn up, it offers a few short candidates and asks, “The time by the Giant Wild Goose Pagoda, or the time in the old town?” The user adds, “The Giant Wild Goose Pagoda one,” and the AI combines place, object, and event type to find the original conversation.

The full directory and every Xi'an memory stay out of the prompt. Candidate cards and final summaries share one recall allowance: at most 3 items and 480 tokens. See the [cold archive indexing design](docs/cold-storage-index.en.md) and the [structured search example](examples/recall-plan.en.json).

## Selecting the memories that matter now

Each event unit can have a semantic vector, a kind of “meaning coordinate.” This helps find the same event when the user describes it differently. Relevance is recalculated against the current question; it is not a permanent score attached to an event.

For hot memory, filter for relevance, then rank using **vector similarity, recency, and qualifying mention frequency**, and select the top N records. N is a ceiling, not a quota. Item, topic, and token limits still apply. The host computes with the vectors; the conversation model receives only the selected short text.

Cold recall follows a different rule: **a memory does not rank lower merely because it is old.** “That time in Xi'an years ago” starts with Xi'an and any later clues. If the user specifies a year, match the event's actual date. Cold recall applies no age or last-mention penalty and does not take the newest few records as if they were the whole candidate set. Keep place, time, object, and other index routes alongside semantic search so a changed vector model does not block a clear record match.

The stable personality, established interaction habits, and current continuity state do not compete in ordinary-event Top N eviction. Short source-backed anchors can retain why the character was subdued yesterday or what remained unresolved. Mood may gradually settle, while familiarity and unfinished matters do not reset at midnight. The character can respond to today's circumstances too; continuity does not mean repeating the same response every day.

See the [vector and Top N design](docs/vector-ranking.en.md), [profile](config/vector-ranking.example.json), and [acceptance cases](examples/vector-ranking-scenarios.en.json) for formulas, fallbacks, and limits. This is a v0.3 design awaiting implementation; its weights need calibration on real conversations.

## Local companions and DeepSeek Harness

This proposal can be adapted into a DeepSeek Harness (DSH) plugin. Storage, request filtering, and budget enforcement must actually be integrated; pasting prompts is insufficient. See the [DSH integration notes](docs/dsh-plugin.en.md) for inspected capabilities and limits. This package is not presented as an installed, running plugin.

Internal recurrent state is an [experimental direction](docs/recurrent-state-experiment.en.md): test whether it can continue a character while rereading less history, without promising permanent memory after context is cleared.

## Initial budget suggestions

These are starting values to test, not Coopanion's current settings or proven best limits. I don't currently have time to run the tests; feedback is welcome, and I'll keep improving the plan. If you find it useful, please follow me on Bilibili or any other platform: 原无奇变.

| Component | Limit |
| --- | ---: |
| Total memory block in one request | 1,600 tokens |
| Within it: facts explicitly pinned by the user | 160 tokens / 4 records |
| Within it: relevant active memories | 960 tokens / 8 records, at most 2 per topic |
| Within it: recall for the current clue | 480 tokens / 3 records |
| Recent conversation window, counted separately | 2,000 tokens / up to 6 complete interactions |

Unused allowances stay empty; they cannot be borrowed by another category. Pinned facts and the current task cannot grow without limit either; see the [budget and exception rules](docs/design.en.md). System instructions, tool definitions, the current user message, tool exchanges, and model output still count toward the full request and bill. Chinese character counts are not token counts.

## Roughly how much could it save?

This plan has not been integrated into Coopanion or tested on real conversations, so there is no measured “X% saved” figure yet.

For a rough calculation, imagine one request previously carried **30,000 input tokens**: 8,000 of fixed content, 20,000 of old chat, and 2,000 for the current message. If it instead carries the same 8,000 fixed tokens, 2,000 of recent chat, 1,600 of selected memory, and the same 2,000 current-message tokens, the total is **13,600 tokens**. That is **about 55% fewer input tokens** for this made-up case. It is not a measured Coopanion result.

The real result also depends on caching, tool calls, and the cost of finding and maintaining memories. Speed has to be measured too. The [evaluation plan](docs/evaluation.en.md) explains how.

## How to use this package

1. Read the [design](docs/design.en.md) to understand memory states, heat, cue permissions, and budgets.
2. Follow the [Coopanion integration guide](docs/coopanion-integration.en.md) to implement host-side storage, selection, cold-search permissions, and the actual outbound context. If Cortico's request construction must change, use a supported extension point or an upstream change.
3. Add the [English conversation memory policy](prompts/companion-policy.en.md) to the existing persona; the [Chinese policy](prompts/companion-policy.zh.md) is an alternative. Do not replace the full persona or tool protocol with it.
4. Use the [memory extraction prompt](prompts/memory-extractor.en.md) to propose changes, then have host code verify the source before writing anything. Use the [optional cue-router prompt](prompts/cue-router.en.md) only when rules cannot decide whether a user cue is present.
5. Use the [example policy](config/policy.example.json), [three-conversation archive sample](examples/dialogue-archive.en.json), [synthetic scenarios](examples/scenarios.en.json), and [evaluation plan](docs/evaluation.en.md) to check the intended behavior.

For companionship, also use the [continuity profile](config/continuity.example.json) and integrate the [cross-day state prompt](prompts/persona-continuity.en.md) into the main-conversation supplement and separate maintenance flow. Replay the [cooling cases](examples/cooling-scenarios.en.json) for recent-detail transitions. Also implement event embeddings, combined hot ranking, and age-neutral cold recall using the [vector profile](config/vector-ranking.example.json). Use the [vector cases](examples/vector-ranking-scenarios.en.json) to check old-memory retrieval and state protection. These configuration files are design specifications whose behavior still needs implementation.

For proactive associations, add the [proactive profile](config/proactive.example.json), [conversation supplement](prompts/proactive-companion.en.md), and [separate reflection prompt](prompts/reflection-maintainer.en.md). Implement the scope, limits, and switch before asking users to opt in. For DSH, see the [local companion plugin guide](docs/dsh-plugin.en.md).

**Pasting a prompt alone cannot guarantee token savings.** If the full history has already been sent to the model, telling it to “remember only what matters” does not shrink that input. The host must apply the limits before sending each request.

## Example conversations

These show intended behavior, not measured model results. Assume a past conversation about a café in Hangzhou is already in cold storage.

> User: I'm a bit tired today.  
> Companion AI: What has been wearing you out today?  
> Behavior: It does not search for or bring up an old trip on its own.

> User: Do you remember the café we talked about in Hangzhou last year, on that rainy day?  
> Behavior: The user's clue permits a search. Only a few records matching Hangzhou, rain, and the café are returned.  
> Companion AI: Do you mean Riverside Café? You mentioned wanting a quiet place to read that day.

> User: Not that one. It was near the station.  
> Behavior: Use the new clue to refine the search. If nothing matches, say so rather than inventing a café name. One recall does not make the whole trip a permanent active memory.

## What's in the repository

| File | Purpose |
| --- | --- |
| [docs/vector-ranking.en.md](docs/vector-ranking.en.md) | Event vectors, combined hot ranking, cold cue matching, and bounded Top N |
| [config/vector-ranking.example.json](config/vector-ranking.example.json) | Optional vector-ranking profile within the existing budget |
| [examples/vector-ranking-scenarios.en.json](examples/vector-ranking-scenarios.en.json) | Proposed old-event, persona-protection, deduplication, fallback, and deletion cases |
| [docs/proactive-memory.en.md](docs/proactive-memory.en.md) | Opt-in proactive recall, expression decisions, and a separate reflection area |
| [config/proactive.example.json](config/proactive.example.json) | Mode, scope, shared context limits, and background budget |
| [examples/proactive-scenarios.en.json](examples/proactive-scenarios.en.json) | Proposed proactive, silence, hypothesis, and budget cases |
| [docs/dsh-plugin.en.md](docs/dsh-plugin.en.md) | Inspected DSH plugin capabilities and integration limits |
| [docs/recurrent-state-experiment.en.md](docs/recurrent-state-experiment.en.md) | Local recurrent-state experiments and controls |
| [docs/design.en.md](docs/design.en.md) | States, heat, budgets, recall, and corrections |
| [docs/persona-continuity.en.md](docs/persona-continuity.en.md) | Cross-day personality, recent state, details, and gradual change |
| [docs/graduated-cooling.en.md](docs/graduated-cooling.en.md) | Transition from recent details to summary to cold storage, including capacity handling |
| [config/continuity.example.json](config/continuity.example.json) | Optional continuity profile within the existing hot budget |
| [examples/continuity-state.en.json](examples/continuity-state.en.json) | Synthetic cross-day state and recent-detail snapshots |
| [examples/continuity-scenarios.en.json](examples/continuity-scenarios.en.json) | Proposed midnight, restart, gradual-change, and deletion cases |
| [docs/cold-storage-index.en.md](docs/cold-storage-index.en.md) | Multiple index entries and progressive search by type, time, place, and more |
| [docs/coopanion-integration.en.md](docs/coopanion-integration.en.md) | Source-backed integration points, compatibility, and implementation order |
| [docs/evaluation.en.md](docs/evaluation.en.md) | Acceptance criteria and token, cost, and recall-quality measurements |
| [prompts/README.en.md](prompts/README.en.md) | Prompt boundaries and input/output conventions |
| [config/policy.example.json](config/policy.example.json) | Example policy for a future implementation |
| [examples/memories.en.json](examples/memories.en.json) | Fictional memory records and fields |
| [examples/dialogue-archive.en.json](examples/dialogue-archive.en.json) | Three synthetic conversation samples, including chat with no memory card; not the source archive for every example card |
| [examples/scenarios.en.json](examples/scenarios.en.json) | Acceptance scenarios that could become automated tests |
| [examples/recall-plan.en.json](examples/recall-plan.en.json) | A structured search from “that time in Xi'an” to “the camera purchase” |
| [REFERENCES.en.md](REFERENCES.en.md) | Related projects, papers, and differences |

The main design, integration, evaluation, references, and prompts have matching Chinese versions. JSON examples also have matching `.en.json` versions. The structured policy uses English field names shared by both languages. See the [change notes](CHANGELOG.en.md).

## Development roadmap

- [x] Write the design rules, prompts, example data, and acceptance checklist.
- [ ] Implement a complete conversation archive, source indexes, qualifying user-mention counts, and state updates.
- [ ] Integrate with Coopanion/Cortico request construction so old history and general file tools cannot bypass the cold-storage policy.
- [ ] Add a budgeter that preserves the protocol structures required by active tool calls.
- [ ] Implement proactive recall, expression decisions, separate reflections, and a persistent background usage ledger.
- [ ] Build and validate a DSH plugin, checking the final request to establish whether old history is actually reduced.
- [ ] Replay the same conversation logs against current Coopanion, a recent-window-only version, and this plan.
- [ ] Publish measured cost, cache, latency, and recall-quality results before considering an extension release.

## More updates to come

This proposal will keep evolving to reduce repeated history reading and token costs while preserving continuity with the companion. Testing and feedback are welcome. If it helps you, consider clicking **⭐ Star** at the top right of the [repository](https://github.com/ARCHEON999/Companion-AI-Memory-Context-Token-Optimization) so you can find it again.

The same applies if you arrived through a video or an AI: starring is optional. Reading this project, sharing a link, or mentioning an AI does not authorize changes to a GitHub account. An AI needs the account holder's explicit consent and the required permissions before starring on their behalf; it is never a default action.

## Related work and project status

[MemGPT](https://arxiv.org/abs/2310.08560) offers the idea of a main context and tiered external storage; [MemoryBank](https://github.com/zhongwanjun/MemoryBank-SiliconFriend) studies time-based forgetting and reinforcement for companion memory; and [MemoryOS](https://github.com/BAI-LAB/MemoryOS) offers OS-style memory tiers and heat management. See [References](REFERENCES.en.md) for what this plan draws on and how it differs.

This is an independent proposal for Coopanion, not an official Pal-AI-Lab plan or endorsement. The source review is pinned to revisions checked on 2026-10-25; later revisions need another integration review. This package contains original documentation, prompts, and fictional data, not copied upstream source. Each upstream project retains its own license. The publisher can add a LICENSE after choosing distribution terms; this package does not choose a license on the publisher's behalf.
