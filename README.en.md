# Companion AI Memory Context and Token Optimization

**中文：[陪伴 AI 记忆上下文 token优化](README.md)**

I recently came across Da Ha Yu ([Coopanion](https://github.com/Pal-AI-Lab/Coopanion)) and the line “爸爸想给你完整的生命” — “Dad wants to give you a complete life.” It stayed with me. As conversations grow, old messages take up more context, some moments can become harder to find after a handoff, and input tokens keep piling up. I wanted a way for an AI companion to keep the whole conversation while using fewer tokens for each reply.

## How it works

Think of human memory alongside a computer's RAM and disk. People don't replay their whole lives every time they talk, and an AI needn't do that either. **Under this design, the original text of every conversation would go into a cold archive** (except anything the user explicitly asks to delete). A small set of recent or often-mentioned memories stays close at hand. Older moments are indexed by type, time, place, and other clues. When the user brings up a person, place, or event from the past, the AI can follow that clue to retrieve just the relevant pieces.

“Forgetting” here means leaving an old conversation out of the current context for now, not erasing its original text. The goal is to send fewer tokens with each request; the real saving still needs to be tested after integration.

## Three ground rules

1. **Keep every conversation; cool down low-frequency memories.** Archive every original turn. Memories the user seldom mentions leave the active set over time. Folding an index away is not enough: old message bodies and expired recalls must also be removed from the actual model request.
2. **Bring old memories back when the user gives a clue.** “That café in Hangzhou last year” can start a search. An ordinary greeting, a timer, a tool result, or the model's own association cannot. When the clue is too vague, ask the user for one more detail.
3. **Even frequent memories have limits.** Count only qualifying user mentions. A retry, an assistant repeating itself, or an automatic summary does not make a memory hotter. Even a frequently mentioned topic enters the request only when relevant and within the item and token limits.

## Finding older memories by clue

The cold archive keeps the full text of every conversation, including everyday chat that never becomes a separate memory card. It then builds logical indexes by **type, time, place, person, object, and topic**. The original conversation is stored once; each category is another way to find it.

Suppose the user suddenly asks, “Do you remember the bowl of ganmianpi (a Shaanxi noodle dish) we had in Xi'an years ago?” The host first searches for “Xi'an” and “ganmianpi.” A future interface could make that brief search feel like taking a moment to remember something from long ago; no such delay is implemented here. If two relevant conversations really exist, the AI can offer a couple of brief choices: “The time near the Giant Wild Goose Pagoda, or the one in the old town?” The user's next clue narrows the search to the original exchange. The AI must not make up a restaurant or an event.

The full directory and every Xi'an memory stay out of the prompt. Candidate cards and final excerpts share one recall allowance: at most 3 items and 480 tokens. See the [cold archive indexing design](docs/cold-storage-index.en.md); a separate [structured search example](examples/recall-plan.en.json) follows a camera purchase in Xi'an.

## Initial budget suggestions

These are starting values to try, not Coopanion's current settings or proven best limits. I haven't run a live test yet. Feedback is welcome, and I'll keep improving the plan. If you find it useful, you can also follow me on Bilibili or other platforms as “原无奇变.”

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
| [docs/design.en.md](docs/design.en.md) | States, heat, budgets, recall, and corrections |
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

The main design, integration, evaluation, references, and prompts have matching Chinese versions. Four JSON examples also have matching `.en.json` versions. The structured policy uses English field names shared by both languages.

## Development roadmap

- [x] Write the design rules, prompts, example data, and acceptance checklist.
- [ ] Implement a complete conversation archive, source indexes, qualifying user-mention counts, and state updates.
- [ ] Integrate with Coopanion/Cortico request construction so old history and general file tools cannot bypass the cold-storage policy.
- [ ] Add a budgeter that preserves the protocol structures required by active tool calls.
- [ ] Replay the same conversation logs against current Coopanion, a recent-window-only version, and this plan.
- [ ] Publish measured cost, cache, latency, and recall-quality results before considering an extension release.

## Related work and project status

[MemGPT](https://arxiv.org/abs/2310.08560) has bounded context and a searchable conversation history; [MemoryBank](https://github.com/zhongwanjun/MemoryBank-SiliconFriend) studies time-based decay and reinforcement for companion memory; [MemoryOS](https://github.com/BAI-LAB/MemoryOS) uses tiers and heat; and [Zep](https://help.getzep.com/episodes) retains ingested source text and derives searchable facts. See [References](REFERENCES.en.md) for what this plan borrows and how it differs.

This is an independent proposal for Coopanion, not an official Pal-AI-Lab plan or endorsement. The source review is pinned to revisions checked on 2026-10-25; later revisions need another integration review. This package contains original documentation, prompts, and fictional data, not copied upstream source. Each upstream project retains its own license. The publisher can add a LICENSE after choosing distribution terms; this package does not choose a license on the publisher's behalf.
