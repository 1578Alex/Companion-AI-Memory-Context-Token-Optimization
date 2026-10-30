# Related Research and Implementation Boundaries

Sources checked: 2026-10-25.

Version 0.1 of this project is an independent design and prompt package for Coopanion, not an installable extension. Bounded working memory, persistent external storage, on-demand recall, frequency-based reinforcement, and time decay have all been studied before. This project does not claim to have invented those ideas. Results in the papers below do not imply that this design has been implemented or has achieved the same results.

## 1. MemGPT: Moving Data Between Bounded Context and External Memory

- Paper: [MemGPT: Towards LLMs as Operating Systems, arXiv:2310.08560v2](https://arxiv.org/abs/2310.08560v2), revised 2024-02-12; [full text, §2](https://arxiv.org/html/2310.08560v2).
- Authors' project page: [research.memgpt.ai](https://research.memgpt.ai/).
- Original official repository entry: [cpacker/MemGPT](https://github.com/cpacker/MemGPT), which redirected to [letta-ai/letta](https://github.com/letta-ai/letta) when checked. This link documents the project's lineage. Our design draws on the research site and paper; it does not depend on current Letta product interfaces or treat them as a feature-by-feature reproduction of the paper.

**What we draw on:** Treat the active context as bounded main memory and keep history and archives in external storage. Move data in and out explicitly, with pressure reminders and paging limits so recalls cannot fill the entire context. In the paper, messages removed from the active context can remain in recall storage. Thus, leaving working memory without deleting history already has a clear precedent.

**Our additional constraints:** A cold recall must be authorized by a cue from the current, authenticated user message. The model's associations, background timers, and past recall results cannot authorize further cold searches. Frequency counts only attributable, deduplicated user mentions. This is a narrower product policy than the paper's autonomous memory retrieval; it is not, by itself, a novelty claim. Runtime code enforces budgets, while prompts express behavioral guidance.

## 2. MemoryBank: Forgetting Curves, Reinforcement, and Long-Term Companionship

- Paper: [MemoryBank: Enhancing Large Language Models with Long-Term Memory, arXiv:2305.10250v3](https://arxiv.org/abs/2305.10250v3); [full text, §§2.2–2.3](https://arxiv.org/html/2305.10250v3). Published at [AAAI 2024](https://doi.org/10.1609/AAAI.V38I17.29946).
- Official repository: [zhongwanjun/MemoryBank-SiliconFriend](https://github.com/zhongwanjun/MemoryBank-SiliconFriend). Checked revision: [cf61c4196e4cfdb0f2b7a0316249fa40312dc3a9](https://github.com/zhongwanjun/MemoryBank-SiliconFriend/tree/cf61c4196e4cfdb0f2b7a0316249fa40312dc3a9).
- Relevant implementation: [forget_memory.py](https://github.com/zhongwanjun/MemoryBank-SiliconFriend/blob/cf61c4196e4cfdb0f2b7a0316249fa40312dc3a9/memory_bank/memory_retrieval/forget_memory.py).

**What we draw on:** The current conversation can serve as a semantic retrieval cue. Memories decay over time; recalls reinforce them and reset decay. SiliconFriend applies this to a long-term AI companion, beyond document question answering.

**Paper versus implementation:** The paper proposes a simplified probabilistic forgetting and reinforcement mechanism. In the official revision linked above, `update_memory_when_searched` strengthens an item after recall, while `initial_load_forget_and_save` deletes forgotten history items and writes the file back. This code path must not be described as an archive where every cold memory remains recoverable.

**Our additional constraints:** Cooling moves a record into a nondestructive cold archive by default; it changes prompt visibility and recall eligibility without destroying the source. Only fresh user mentions increase frequency. Model recall, repetition, and maintenance summaries do not. Cold recall also requires a user cue and a budget check. These differences from the inspected implementation do not establish novelty against all prior art. The paper does not provide a token savings percentage transferable to this project.

## 3. MemoryOS: Memory Tiers, Topic Heat, and Selective Retrieval

- Paper: [Memory OS of AI Agent, EMNLP 2025](https://aclanthology.org/2025.emnlp-main.1318/); [final PDF](https://aclanthology.org/2025.emnlp-main.1318.pdf). We also checked [arXiv:2506.06326v1](https://arxiv.org/html/2506.06326v1), §§3.3–3.4 and Table 3.
- Official repository: [BAI-LAB/MemoryOS](https://github.com/BAI-LAB/MemoryOS). Checked revision: [587ed7755c7aed179965792830ff1b5ad9a6fa92](https://github.com/BAI-LAB/MemoryOS/tree/587ed7755c7aed179965792830ff1b5ad9a6fa92).
- Inspected implementation paths: [memoryos-pypi/mid_term.py](https://github.com/BAI-LAB/MemoryOS/blob/587ed7755c7aed179965792830ff1b5ad9a6fa92/memoryos-pypi/mid_term.py) and [retriever.py](https://github.com/BAI-LAB/MemoryOS/blob/587ed7755c7aed179965792830ff1b5ad9a6fa92/memoryos-pypi/retriever.py). The repository has other variants; our observations do not cover every one.

**What we draw on:** Short-, medium-, and long-term memory tiers; a bounded queue for recent dialogue; topic heat based on visit count, interaction length, and recency; selecting a topic before retrieving its relevant pages; and promoting salient topics into long-term information.

**Paper versus implementation:** The paper describes evicting medium-term topics by heat. The inspected path computes heat, but its capacity eviction function is `evict_lfu`, which removes the least frequently visited session before saving. This path does not guarantee a recoverable cold archive, and the paper's description should not be treated as the behavior of every code variant. System retrieval can also raise heat there, unlike our user-mention-only rule.

**Limits of the efficiency evidence:** Table 3 compares retrieved tokens, model calls, and answer quality on LoCoMo. Different baselines involve tradeoffs across those measures. The table is not an end-to-end Coopanion API billing experiment, nor does it isolate the gains from user-cue gating or our frequency rule. This project has no verified savings percentage.

## 4. Zep: Source Episodes, Derived Facts, and Selected Context

- Official documentation: [Agent memory](https://help.getzep.com/v4/agent-memory-solution), [Episodes](https://help.getzep.com/episodes), [How Graph Creation Works](https://help.getzep.com/how-graph-creation-works), and [Context types](https://help.getzep.com/context-types). We checked the accessible v4 documentation on 2026-10-25; these observations do not cover every earlier version or deployment.

**Direct overlap:** Each message an application sends to Zep becomes a source `episode`. Zep retains ingested episodes and derives entities, time-scoped facts, and summaries from them. Relevant source text and derived context can be searched or selected into a Context Block. This already covers much of the basic idea of keeping originals outside the prompt, finding sources through indexed facts, and supplying only relevant context. Zep can retain only what an application actually ingests; its documentation does not prove that every integration archives every conversation turn.

**Our additional rules:** The host must archive every user and assistant dialogue turn. Only a discriminating cue from the current authenticated user, or a valid continuation of that topic, can open old-memory retrieval. Only deduplicated substantive user mentions raise heat, and fixed limits apply to the actual outbound request. Zep's documented standard flow retrieves a relevant Context Block before the next model request, with default relevance based on recent messages. That is not the same as a mandatory user-cue gate, although an application could add one. This proposal has not shown lower cost or better recall than an integration built on Zep or another existing system.

## 5. What This Project Combines

| Policy | Our rule | Relationship to prior work |
| --- | --- | --- |
| Nondestructive cold storage | Demote and archive by default; low frequency alone does not delete source records | MemGPT's recall storage and Zep's source episodes are precedents; this differs from inspected implementations that delete records |
| User-cue gate | Only the current authenticated user can authorize cold recall; background tasks and model initiative cannot expand the scope | A narrower product policy than general autonomous or every-query retrieval |
| Faceted, progressive lookup | Start with any supported type, event time, place, person, object, or topic; show short candidate cards before reading necessary bodies | Uses established metadata filtering and faceted search; logical partitioning is not claimed as a new algorithm |
| User-mention frequency | Count only deduplicated user mentions; model recall, summaries, and repetition do not count | Avoids self-reinforcing heat; the effect still requires evaluation |
| Enforced budgets | Bound working memory, recalls, maintenance, and tool results with defined overflow behavior | Memory tiers and capacity limits have precedents; this project specifies observable runtime constraints |
| Coopanion integration | Adapt the existing session handoff, tool loop, persistent workspace, and cache behavior | Still to be implemented and validated; not an official existing feature or an accepted maintainer proposal |

This combination is a proposed engineering design. The identified precedents are enough to rule out a claim that nobody has tried related ideas. This document is not an exhaustive patent search or academic novelty review.

## 6. Coopanion-Specific Boundaries

The source review targeted these fixed revisions, not a completed runtime integration:

- [Coopanion `4afc70898ac1065dd4559d891a2ce0bebd1261f8`](https://github.com/Pal-AI-Lab/Coopanion/tree/4afc70898ac1065dd4559d891a2ce0bebd1261f8).
- Its `vendor/cortico` pointer to [Cortico `cb9ddbd685bcccba1f29905098648f46a79b3862`](https://github.com/Pal-AI-Lab/Cortico/tree/cb9ddbd685bcccba1f29905098648f46a79b3862).

An implementation must inspect the commits, configuration, and provider endpoints actually used at integration time. It should not claim compatibility with an unspecified “latest version.”

- [Coopanion issue #342 and maintainer reply](https://github.com/Pal-AI-Lab/Coopanion/issues/342#issuecomment-6379588695): The maintainer chose to keep a fixed tool list to reuse prefix caching instead of adopting the issue reporter's dynamic tool retrieval suggestion. This existing tradeoff belongs in our comparison; it is not universal proof that dynamic retrieval is always more expensive.
- [Coopanion issue #333 and maintainer reply](https://github.com/Pal-AI-Lab/Coopanion/issues/333#issuecomment-6379589299): The persistent single session and threshold-based handoff have existing implementation and product rationale. This proposal does not introduce summarization into a system that lacked any handoff mechanism.
- [Coopanion PR #375](https://github.com/Pal-AI-Lab/Coopanion/pull/375): Conventions already exist for organizing workspace content by topic and date under `archive/`. This proposal must demonstrate its incremental contribution.

Cold content excluded from model input consumes no primary-request input tokens for that content. Indexing, retrieval, summarization, maintenance, tool round trips, and cache misses can still add cost. An evaluation should record actual input tokens, cached input, output, extra calls, maintenance cost, latency, and memory accuracy. A prompt by itself cannot guarantee external storage, an enforced budget, or access control.

## v0.4: proactive recall, reflection, and recurrent state

- [Generative Agents (2023)](https://arxiv.org/abs/2304.03442): experience records, retrieval, and higher-level reflection have precedents. Proactivity alone does not establish novelty.
- [Letta memory and dreaming](https://docs.letta.com/configuration/memory): background consolidation and reflection already exist. This proposal emphasizes provenance separation, expression decisions, and bounded background cost, without claiming these broad ideas are new.
- The [official Qwen3.8-27B card](https://huggingface.co/Qwen/Qwen3.8-27B/blob/main/README.md) and [Gated DeltaNet research](https://research.nvidia.com/publication/2025-04_gated-delta-networks-improving-mamba2-delta-rule) support architectural and state-mechanism statements, not permanent personality after context clearing.

Checked on 2026-10-30. The user-supplied third-party dashboard informed requirements; it is not public source, a performance benchmark, or compatibility evidence. Neither the screenshot nor its private dialogue was uploaded. See the separate DSH notes for version-pinned evidence.
