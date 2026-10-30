# 参考研究与实现边界

检索日期：2026-10-25。

本项目 0.1 是一份面向 Coopanion 的独立设计与提示词包，不是可安装扩展。有限工作记忆、外部持久存储、按需召回、频率强化和时间衰减都有已有研究；本文不主张这些原理为本项目首创。下述论文结果也不代表本项目已经实现或测得相同收益。

## 1. MemGPT：有限上下文与外部记忆之间的数据移动

- 论文：[MemGPT: Towards LLMs as Operating Systems，arXiv:2310.08560v2](https://arxiv.org/abs/2310.08560v2)，2024-02-12 修订版；[全文 §2](https://arxiv.org/html/2310.08560v2)。
- 作者项目页：[research.memgpt.ai](https://research.memgpt.ai/)。
- 官方代码原入口：[cpacker/MemGPT](https://github.com/cpacker/MemGPT)，检索时重定向至 [letta-ai/letta](https://github.com/letta-ai/letta)。该入口只用于追溯项目延续关系；本方案引用的是研究站与论文设计，不依赖当前 Letta 产品的接口，也不将其视为论文实现的逐项复现。

**借鉴点：** 将当前上下文视为有限主存，将历史与归档放在外部存储；通过工具显式调入、调出；使用压力提醒与分页限制避免召回结果挤满上下文。论文中的消息移出当前窗口后仍保存在 recall storage，因此“退出工作记忆不等于删除历史”已有明确先例。

**本方案的约束：** 冷记忆只能由当前真实用户消息提供的线索授权召回；模型自己的联想、后台定时任务与旧召回结果不能自行授权冷检索。频率只统计可归因、去重的用户提及。此约束比论文描述的自主记忆检索更窄，是本项目的产品策略，不能据此认定具有独创性。具体预算由运行时执行，提示词只表达行为规则。

## 2. MemoryBank：遗忘曲线、重复强化与长期陪伴

- 论文：[MemoryBank: Enhancing Large Language Models with Long-Term Memory，arXiv:2305.10250v3](https://arxiv.org/abs/2305.10250v3)；[全文 §2.2–2.3](https://arxiv.org/html/2305.10250v3)。正式发表：[AAAI 2024](https://doi.org/10.1609/AAAI.V38I17.29946)。
- 官方仓库：[zhongwanjun/MemoryBank-SiliconFriend](https://github.com/zhongwanjun/MemoryBank-SiliconFriend)。核验快照：[cf61c4196e4cfdb0f2b7a0316249fa40312dc3a9](https://github.com/zhongwanjun/MemoryBank-SiliconFriend/tree/cf61c4196e4cfdb0f2b7a0316249fa40312dc3a9)。
- 相关实现：[forget_memory.py](https://github.com/zhongwanjun/MemoryBank-SiliconFriend/blob/cf61c4196e4cfdb0f2b7a0316249fa40312dc3a9/memory_bank/memory_retrieval/forget_memory.py)。

**借鉴点：** 当前对话可作为语义检索线索；记忆随时间衰减，召回会强化记忆并重置衰减时间。SiliconFriend 将这种机制用于长期 AI 陪伴，而非仅用于文档问答。

**论文与实现的区别：** 论文提出简化的概率遗忘与强化机制。上述官方快照中，`update_memory_when_searched` 在召回后增加强度；`initial_load_forget_and_save` 对被遗忘的历史项执行删除并写回文件。因此不能把它描述为“所有冷记忆始终可恢复的归档系统”。

**本方案的约束：** 默认采用非破坏性冷归档；降温只改变是否进入提示词及召回资格，不销毁原始记录。只有真实用户的新提及增加频率，模型召回、复述和维护摘要不增加。冷召回还须通过用户线索授权与预算检查。这些是对照该实现的区别，不是对全部现有技术的新颖性证明。未从该论文得到可用于本项目的 token 节省比例。

## 3. MemoryOS：分层记忆、主题热度与选择性检索

- 论文：[Memory OS of AI Agent，EMNLP 2025](https://aclanthology.org/2025.emnlp-main.1318/)；[最终论文 PDF](https://aclanthology.org/2025.emnlp-main.1318.pdf)。同时核对了 [arXiv:2506.06326v1](https://arxiv.org/html/2506.06326v1) 的 §3.3–3.4 与表 3。
- 官方仓库：[BAI-LAB/MemoryOS](https://github.com/BAI-LAB/MemoryOS)。核验快照：[587ed7755c7aed179965792830ff1b5ad9a6fa92](https://github.com/BAI-LAB/MemoryOS/tree/587ed7755c7aed179965792830ff1b5ad9a6fa92)。
- 本次核查的实现路径：[memoryos-pypi/mid_term.py](https://github.com/BAI-LAB/MemoryOS/blob/587ed7755c7aed179965792830ff1b5ad9a6fa92/memoryos-pypi/mid_term.py)、[retriever.py](https://github.com/BAI-LAB/MemoryOS/blob/587ed7755c7aed179965792830ff1b5ad9a6fa92/memoryos-pypi/retriever.py)。仓库还有其他变体；本次结论不覆盖所有变体。

**借鉴点：** 短、中、长期分层；近期对话使用有限队列；主题热度综合访问次数、交互量和近期性；先定位主题，再选择相关页面；热主题提炼长期信息。

**论文与实现的区别：** 论文描述按热度淘汰中期主题；核查路径虽计算热度，其容量淘汰函数实际为 `evict_lfu`，移除最低访问频率的 session 后保存。不能将这一路径等同于保证可恢复的冷归档，也不能将论文描述直接视为所有代码变体的执行行为。其热度可由系统检索增加，与本方案“仅用户提及计频率”不同。

**效率证据边界：** 论文表 3 比较 LoCoMo 上的召回 token、模型调用与回答质量；相对不同基线，各指标存在取舍。它不是本项目在 Coopanion 上的完整 API 账单实验，也未证明用户线索门控或频率规则单独带来的收益。本项目尚无经验证的节省比例。

## 4. Zep：原文片段、派生事实与相关上下文

- 官方文档：[Agent memory](https://help.getzep.com/v4/agent-memory-solution)、[Episodes](https://help.getzep.com/episodes)、[How Graph Creation Works](https://help.getzep.com/how-graph-creation-works)、[Context types](https://help.getzep.com/context-types)。核查的是 2026-10-25 可访问的 v4 文档，不推断所有旧版本或部署配置。

**相似点：** 应用送入的每条消息成为原文 `episode`；Zep 保留这些原文，并从中派生人物等实体、带时间的事实与摘要。相关原文和派生信息可检索、可选入 Context Block。这已经覆盖“原文冷存、按线索找来源、只给模型相关片段”的大部分基本做法。Zep 只负责保存应用实际送入的消息，不能由此断言任意接入程序都完整归档了每轮对话。

**本方案的额外规定：** 宿主必须归档每轮用户与助手对话原文；只有当前真实用户的可定位线索或有效同话题续问才能开启旧记忆检索；只有用户的去重有效提及才能增加热度，并在实际发出的请求中执行固定限额。Zep 文档给出的常规接入会在下一次模型请求前获取相关 Context Block，相关性默认依据近期消息；这不等同于上述用户线索门控，但应用可以自行增加门控。本方案尚未证明比采用 Zep 等现成系统更省钱或更准确。

## 5. 本项目组合的明确含义

| 策略 | 本项目的规定 | 与参考研究的关系 |
| --- | --- | --- |
| 非破坏性冷存 | 默认降级、归档，不因低频而删掉源记录 | MemGPT 的历史库与 Zep 的原文 episode 已有先例；针对存在删除行为的实现作不同选择 |
| 用户线索门控 | 仅当前真实用户消息能授权冷召回；不能由后台或模型自发扩大 | 比一般自主或逐请求语义检索更窄的产品策略 |
| 分类索引与逐步定位 | 类型、时间、地点、人物、物品、话题多入口筛选；先候选卡，后必要正文 | 使用已有元数据过滤与分面检索方法；不将逻辑分区称为新算法 |
| 用户提及频率 | 只计去重后的用户提及；召回、摘要、模型复述不计 | 避免“越召回越热”的自强化；效果仍须检验 |
| 严格预算 | 工作记忆、召回、维护与工具结果分别受限；超额有确定回退 | 分层与容量控制已有先例；本项目要求可执行、可观测的约束 |
| Coopanion 集成 | 在其既有会话交接、工具调用、持久工作区与缓存机制中落地 | 尚待实现和验证；不是已有官方功能或已获维护者认可的方案 |

上述组合是待验证的工程方案。检索到先例足以否定“此前无人这样做”的表述；本文件也不作完整专利或学术新颖性检索结论。

## 6. 对 Coopanion 的适用边界

本方案的源码审阅目标为以下固定快照，不表示已经完成运行时集成：

- [Coopanion `4afc70898ac1065dd4559d891a2ce0bebd1261f8`](https://github.com/Pal-AI-Lab/Coopanion/tree/4afc70898ac1065dd4559d891a2ce0bebd1261f8)。
- 该快照的 `vendor/cortico` 指向 [Cortico `cb9ddbd685bcccba1f29905098648f46a79b3862`](https://github.com/Pal-AI-Lab/Cortico/tree/cb9ddbd685bcccba1f29905098648f46a79b3862)。

实际实现须以接入时锁定的提交、配置与端点为准，不能只写“兼容最新版”。

- [Coopanion #342 及维护者回复](https://github.com/Pal-AI-Lab/Coopanion/issues/342#issuecomment-6379588695)：维护者目前选择保留固定工具表，以复用前缀缓存；没有采纳报告者的动态工具检索建议。这是本方案必须纳入比较的既有取舍，不是“动态检索必然更贵”的普遍证明。
- [Coopanion #333 及维护者回复](https://github.com/Pal-AI-Lab/Coopanion/issues/333#issuecomment-6379589299)：连续单会话与阈值交接已有实现和产品理由。本方案不是给一个完全没有压缩机制的系统首次增加摘要。
- [Coopanion #375](https://github.com/Pal-AI-Lab/Coopanion/pull/375)：已有按主题、日期整理工作区与 `archive/` 的约定。此次方案仍须证明自己的增量作用。

冷资料未进入模型输入时，不产生该部分主请求输入 token；但索引、检索、摘要、维护、工具往返和缓存失效仍可能增加成本。检验时应同时记录实际输入、缓存输入、输出、额外调用、维护费用、延迟与记忆正确率。只写一份提示词不能保证外部存储、强制预算或权限门控已经生效。

## v0.4：主动回忆、反思与循环状态

- [Generative Agents（2023）](https://arxiv.org/abs/2304.03442)：体验记录、检索与高层反思已有研究；主动回忆不能单独作为首创声明。
- [Letta 记忆与 dreaming](https://docs.letta.com/configuration/memory)：后台整理与反思已有实现思路；本方案强调来源隔离、表达判断和后台预算，不声称这些大方向首次出现。
- [Qwen3.8-27B 官方模型说明](https://huggingface.co/Qwen/Qwen3.8-27B/blob/main/README.md)与[Gated DeltaNet研究](https://research.nvidia.com/publication/2025-04_gated-delta-networks-improving-mamba2-delta-rule)：仅支持架构与状态机制依据，不证明清空上下文后永久保留人格。

以上于2026-10-30核对。用户提供的第三方控制台截图用于理解需求，未当作公开源码、性能测试或已验证兼容性的证据，也未上传截图或其中的私人对话。DSH的固定版本依据见其单独接入说明。
