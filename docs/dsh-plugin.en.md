# A memory plugin for DeepSeek Harness

**中文：[DSH 记忆插件接入说明](dsh-plugin.md)**

Yes, this can be built for **DeepSeek Harness (DSH)**. The goal is for a local companion using DSH to pick up yesterday's conversation: keep its character and recent details close, store older experiences, and bring back only what matters.

**This repository currently contains an integration investigation and a plan, not an installable DSH plugin.** Nothing has been installed in or changed in the user's DSH, and no real conversation trial has run. Pasting prompts cannot replace storage, retrieval, or request budgeting.

## What was checked

The desktop package inspected read-only on 2026-10-30 was `0.2.1-rc.3`, application ID `com.deepseek.dsh`. Its metadata declares build commit `be72761ee1a6736ebcdf4f3739363b083ae53c98`. It includes the Cordis plugin loader, official plugin-development documentation, sessions, storage, compaction, timers, and tool-policy modules.

The official [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) repository was also checked at public snapshot `ce7073d51254c26cd63b22a60835b8f948deb447`. The installed build commit could not be found in the public repository, so these are not treated as identical versions. The route below is supported by the bundled documentation and public source; the active DSH configuration and runtime interfaces still need inspection before activation. The public snapshot already deprecates `agent/request`; a new plugin should not depend on it.

The supplied “Soul Console / persoul-worker” screenshot shows another developer's service. It illustrates their idea, but does not establish that this memory system is built into official DSH or connected to this repository.

## Available extension points

| Need | Verified host capability | Project status |
| --- | --- | --- |
| Incrementally process completed dialogue | `sessionProjections`, committed session events, and watermarks | Extraction and restoration not implemented |
| Keep originals, event cards, indexes, and reflections separate | Host-side `storageDomain` and durable backends; retain original session logs | Concrete records and migrations not implemented |
| Preserve character, relationship habits, and current state | Agent-scoped `systemPrompt.section()`; `agent.inject()` when replayable context is needed | Limits, gradual updates, and deduplication not implemented |
| Cue-based recall and optional proactive recall | Host-side permission and scope checks followed by bounded retrieval | Not implemented; permissions cannot rely on the model alone |
| Decide whether to mention, ask, or stay silent | Give limited candidates to the main response and choose expression in that same generation | Prompts provided; runtime validation pending |
| Actually reduce old history sent to the model | Official `compaction` service and replaceable backend; record summary replacements while retaining raw events | Integration and request verification pending |
| Prevent tools from bypassing cold retrieval | Agent-scoped `tools.restrict()` and `tools.guard()` | Every archive-reading route needs checking |
| Background maintenance with call limits | Host lifecycle, timers, storage; `inject()` does not wake an agent, `followup()` does | Durable accounting and cancellation not implemented |

This supports a Host / Cordis plugin. An MCP memory-search tool can supply retrieval, but usually cannot control DSH's existing history, general file tools, and background calls on its own. Connecting MCP is not sufficient evidence of context savings.

## One conversation turn

1. A user sends a message. The host resolves the user, companion, and topic, then applies deletion, suppression, and “only when asked” restrictions.
2. It selects character/current state, recent details, and relevant facts from hot memory. Ordinary hot events use semantic relevance, recency, and valid user mentions; core character and current continuity are protected separately.
3. An explicit recall cue allows cold retrieval. With proactive mode separately enabled, current context may also open an allowed topic. Place, date, people, objects, and topic narrow the search before a few candidates are returned. Cold retrieval does not penalize an event merely for being old.
4. The main response chooses natural use, a tentative question, or silence about the past. “You were tired from overtime last time” may support “Is it work again?”, not the claim “You worked overtime today.”
5. Before dispatch, the host deduplicates, reduces retained history, and checks the complete request budget. After completion, it archives the original dialogue and advances memory by committed source IDs and watermarks. The assistant repeating a detail does not increase its heat.
6. Only newly completed interactions queue maintenance. No new content means no maintenance call. A separate reflection area may hold sourced, expiring hypotheses; it cannot turn them into things the user experienced.

Use the existing [base design](design.en.md), [continuity](persona-continuity.en.md), [vector ranking](vector-ranking.en.md), and [proactive recall](proactive-memory.en.md). DSH does not receive a second, larger budget.

## Two easy mistakes

**Adding memory does not reduce history.** DSH builds requests from its session log, and messages are frozen at model dispatch. `agent/pre-step` handles the incoming batch; it is not an arbitrary full-history deletion interface. Neither `agent/request` nor `llm/stream` should secretly rewrite it. Older history needs official compaction, preserving tool-call pairing, locking, and replay records.

Material added by `agent.inject()` also becomes retained history. Injecting at most 1,600 tokens per turn can still accumulate across ten turns. An implementation must retire superseded recalls and snapshots, bound recent history, and inspect the **complete request actually sent to the model**. System prompts, tool definitions, current input, and active tool protocol are accounted for separately; changing prompts repeatedly may also affect caching.

**Timers do not make reflection free.** Do not periodically scan the entire cold archive or wake the main companion to “think of something.” Background work processes only newly completed, authorized events and coalesces them while idle. Example limits are two attempts per day, at least six hours apart, and 2,000 input / 300 output tokens per attempt. Failures, retries, embeddings, and reranking share the configured allowance.

Storage provides atomic durability for individual KV calls, not automatic transactions across several records. Persist a reservation before dispatch; serialize work or use verified version checks, and retain idempotency keys, watermarks, and the last attempt time. Restart and midnight must not clear the limits. DSH's token meter uses approximate text estimates; it is not a hard billing limit for Chinese. Without reliable counting or an output cap, disable background model calls and retain rule-based archiving.

## First implementation scope

Start with one companion and one user scope: a Host plugin, independent storage, and bounded retrieval, without changing other agents. Cue-based recall remains the default; proactive mode is separately enabled. Memory remains 160 pinned + 960 hot + 480 recalled tokens. Proactive candidates, events, and any visible reflection share that final 480 tokens.

Protect core character, current relationship habits, and unfinished matters. Update recent state gradually from sources; compress fading events into summaries before moving them out of active memory. Retain originals and provenance except for explicit deletion. Reflections are separate, inspectable, and expiring; corrections or deletion of sources invalidate their derivatives.

Begin with an existing durable backend in the active configuration. Consider PostgreSQL / pgvector only if measured scale or query performance requires it. The screenshot's use of PG does not make a new database a prerequisite for this plugin.

General file, search, shell, subagent, or other tools may bypass retrieval budgets if they can read the full cold archive. Check and restrict each relevant route within the companion's scope. A rule on one search tool is not filesystem isolation for the whole computer.

Internal recurrent state is a separate [experiment](recurrent-state-experiment.en.md). This investigation found no DSH DeepSeek API interface for exporting and restoring Gated DeltaNet hidden state; it is not included as a plugin capability.

## When can it be called a working plugin?

- [ ] Inspect the active configuration and runtime interfaces; name supported DSH versions and disable unsupported functions with a clear reason.
- [ ] Complete the bundle, configuration, storage migrations, and rollback; confirm the requested behavior after installation.
- [ ] Preserve character baseline, recent details, watermarks, and background allowance across sessions, restart, and midnight.
- [ ] Inspect final requests: old originals and expired recalls stop repeating, tool protocol stays valid, and memory and recent history each stay bounded.
- [ ] Recover original records from old place/date/object cues; do not treat weak emotion as causal evidence; honor proactive rejection and “only when asked.”
- [ ] Propagate corrections, deletion, and revoked access to reflections, summaries, vectors, and caches; do not silently restore visible memories from old logs or backups.
- [ ] Make zero background calls without new events; keep concurrent work, failures, timeouts, and restarts within persisted reservations.
- [ ] Include extraction, retrieval, embeddings, background work, retries, and cache costs in comparisons, then publish measured results.

All of these are pending acceptance checks. There is no measured savings percentage or “never forgets” result yet.

## Interface sources

Links are pinned to the inspected public snapshot. The installed runtime still requires live inspection.

- [Official plugin practices](https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/preset/agent-preset/skills/cordis-plugin-development/references/practices.md): log-based truth, scope, injection, tool restrictions, and incremental projections.
- [Agent](https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/core/agent/README.md) and [LLM](https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/llm/llm/README.md): incoming messages, lifecycle, and frozen requests.
- [Compaction](https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/compaction/compaction/README.md): summary replacement and retention of raw logs.
- [Storage](https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/storage/storage/README.md), [Tools](https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/core/tools/README.md), and [Session projection](https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/session/session-projection/README.md): durable data, tool policy, and incremental session state.
