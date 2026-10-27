# 陪伴 AI 记忆上下文 token优化

**English: [Companion AI Memory Context and Token Optimization](README.en.md)**

最近看到大肥鱼陪伴 AI“爸爸想给你完整的生命”事件
（[Coopanion](https://github.com/Pal-AI-Lab/Coopanion)）。随着聊天内容增多，上下文超额，会丢失记忆与消耗巨量 token。为此我设计了一套适用于陪伴 AI 保留完整记忆并节省 Token的方案。

## 解决思路

原理是参考人脑，也参考常见的存储优化逻辑，把记忆分成热区和冷区。

最近频繁提到的事情、性格底色、相处习惯和近期状态，留在热记忆这一侧，放进当前上下文。性格底色与已经形成的相处基线持续保留，具体事情只选这次用得上的少量内容。

逐渐不提的往事，先从具体细节收成短摘要，再慢慢退出上下文，放到硬盘里的冷记忆库。所有原话从一开始就完整留底；优化时也要保住近期的细节和没聊完的事，尽量让变化自然，不因为过了一晚或换个会话就突然像变了个人。

等用户再提到相关的人、时间、地点、物品或话题，就按线索去冷库找回来。省下的是每次都重复读取旧话的上下文和 Token。

## 三条核心规则

1. **全部对话留底，低频渐进冷存。** 每轮原文都归档；少被用户提及的事件按“细节 → 摘要 → 冷库”逐步退出活跃区。只把目录折叠起来不够，还要从实际模型请求中移除旧正文与旧召回片段。
2. **用户线索唤回。** “去年杭州那家咖啡店”可以启动检索；普通问候、定时唤醒、工具输出或模型自己想起某件事不能启动冷检索。线索模糊时，请用户补充一句。
3. **高频调用有限。** 只统计用户的有效提及，同一轮重试、助手复述、自动摘要不增加热度；即便高频，也只注入与当前对话有关的少量条目。

## 第二天还是昨天那个它

大肥鱼让人有感觉的地方，还有它真实、熟悉的性格。如果第二天记得事情，却突然换了语气、忘了昨天的状态，陪伴还是会断掉。

因此增加一层“连续性状态”：基础性格一直保留，最近形成的相处方式慢慢变化；近期语气、没说完的话和两三条具体细节跨天接着用，再逐渐淡下来。久远的事件和历史状态版本进冷库，当前的性格与相处基线持续保留，不在零点或重启时清空。

这部分仍是待接入的可选设计，在原有 960 tokens 热记忆中预留：320 给连续性状态、320 给近期细节、320 给本轮相关事实，总额度不增加。需要程序真正保存、恢复和限制变化幅度，单靠一句“保持性格一致”不够。详见[性格与近期状态连续性](docs/persona-continuity.md)、[补充提示词](prompts/persona-continuity.zh.md)和[配置](config/continuity.example.json)。

实现时，基础人格保留在固定角色前缀，近期变化放在有上限的热记忆快照；两者都会进入当前上下文，也都计入总输入。

## 自然过渡，不突然断片

近期事件先保留有辨识度的细节，例如谁说的、当时的原句、时间和下一步；不再常提时，先留一段仍能接上话的短摘要，再逐步转入只按线索读取的冷库。细节和摘要都是热记忆的不同精简程度。当前正在聊的话题和未完成的约定优先保留，不能只因为到点了就整段撤下。

性格底色和当前相处基线不跟着旧事件一起冷藏。近期状态缓慢变化，旧状态版本进入历史档案。所有阶段都受容量限制，保留哪些细节、怎样缩短摘要，需要在真正发给模型之前执行；这仍是待接入验证的规则。详见[渐进冷却设计](docs/graduated-cooling.md)。

## 远古记忆按线索分区检索

冷档案保存全部对话原文，包括不会单独做成“记忆卡”的日常聊天。再按**类型、时间、地点、人物、物品、话题**建立逻辑索引，同一段原文只存一份，各个分类都是检索入口。

比如用户突然提到很久之前的“你还记得当年在西安吃的那碗擀面皮儿吗？”就去根据【西安】【擀面皮】检索相关记忆。这段检索等待期也可以当做人类正常回忆久远时期的感觉。如果找到几件往事，返回少量候选，问“大雁塔那次，还是老城区那次？”用户补充“大雁塔那次”，再组合地点、物品和事件类型找到正文。

完整目录与全部西安往事都不进入上下文。候选卡和最终摘要共用召回额度，最多 3 条 / 480 tokens。详见[逻辑分区设计](docs/cold-storage-index.md)和[结构化查询示例](examples/recall-plan.json)。

## 用向量选出这次真正用得上的记忆

每个“事件元”可以保存一个语义向量，像给这件事标出一个“意思坐标”。用户换个说法，也能找到同一件事；关联程度按当前问题重新计算，不是给事件永久打一个分。

热记忆先筛相关内容，再把**向量关联度、时间新近度、有效提及频率**合起来排序，选靠前的 Top N 条。N 是上限，不需要凑满；条数、主题和 Token 额度仍然有效。向量在存储程序里计算，主对话只接收选中的少量文字。

冷库用另一套规则：**旧事不会因为年头久就被压下去。** 问“当年西安那次”，就按西安和后续线索找；如果用户说了年份，匹配的是事件发生时间。冷召回不叠加“距今多久”的衰减，也不先取最近的几条来冒充全部候选。语义搜索之外保留地点、时间、物品等索引入口，避免换了向量模型就找不到原本明确的记录。

性格底色、已经形成的相处方式和当前连续性状态不参加普通事件的 Top N 淘汰。昨天为什么低落、哪句话还没聊完，可以留下带来源的短锚点；情绪可以慢慢缓和，相处的熟悉感和没解决的事不能到零点就清零。它也会回应今天的新情况，保持连续不等于每天说同样的话。

具体公式、降级规则和边界见[向量关联与 Top N 设计](docs/vector-ranking.md)、[配置](config/vector-ranking.example.json)及[验收案例](examples/vector-ranking-scenarios.json)。这是待实现的 v0.3 设计，权重尚需用真实对话校准。

## 初始预算建议

以下是待实验的初始值，不是 Coopanion 现有配置，也不是最佳参数。我暂时没空来跑测试，欢迎反馈我会优化方案。如果有用欢迎关注我的 B 站或任何平台：原无奇变

| 项目 | 上限 |
| --- | ---: |
| 每个请求中的记忆总块 | 1,600 tokens |
| 其中：用户固定的事实 | 160 tokens / 4 条 |
| 其中：相关活跃记忆 | 960 tokens / 8 条，每主题最多 2 条 |
| 其中：当前线索召回 | 480 tokens / 3 条 |
| 近期历史窗口（另计） | 2,000 tokens / 最多 6 个完整交互单元 |

各子预算不可互借，没用完就留空。固定事实、当前任务也不能无限增长；详见[预算与例外处理](docs/design.md)。系统提示词、工具定义、当前用户消息、工具执行、模型输出仍要另外计入完整请求与账单。中文字符数不能直接当作 token 数。

## 大概能省多少？

还没有把这套方案接进 Coopanion 跑真实对话，所以现在没有“实测节省 X%”的数据。

可以拿一个长对话粗算：假设原来一次要带 **30,000 输入 tokens**（固定内容 8,000、旧聊天 20,000、当前消息 2,000）。整理后若只带固定内容 8,000、近期聊天 2,000、选中的记忆 1,600、当前消息 2,000，就是 **13,600 tokens**。这样算，单次输入少了约 **55%**。这是按假设数字算的例子，不是 Coopanion 已经省下的量。

实际能省多少，还要看缓存、工具调用和找记忆本身的开销；变快多少也得跑了才知道。[评估计划](docs/evaluation.md)写了怎么测。

## 使用方式

1. 阅读[设计说明](docs/design.md)，确认记忆状态、热度、检索门控和预算规则。
2. 按[Coopanion 接入说明](docs/coopanion-integration.md)实现宿主侧存储、选取、冷检索权限与实际请求投影。需要改变 Cortico 的请求构造行为时，在其支持的扩展点或上游实现。
3. 将[对话记忆策略](prompts/companion-policy.zh.md)作为现有角色提示词的补充；上游接入可参考[英文版](prompts/companion-policy.en.md)。不要用它替换完整人格或工具协议。
4. 用[记忆提取提示词](prompts/memory-extractor.zh.md)生成候选变更，由程序核验来源后写入；[线索判定提示词](prompts/cue-router.zh.md)仅在规则无法判断时选用。
5. 以[示例配置](config/policy.example.json)、[三段对话归档样本](examples/dialogue-archive.json)、[合成案例](examples/scenarios.json)和[评估计划](docs/evaluation.md)验证效果。

陪伴场景建议同时采用[连续性配置](config/continuity.example.json)，并按[跨天状态提示词](prompts/persona-continuity.zh.md)分别接入主对话补充与独立维护；近期细节的过渡再用[冷却案例](examples/cooling-scenarios.json)回放。同时按[向量排序配置](config/vector-ranking.example.json)实现事件向量、热区综合排序与不惩罚年代的冷召回；用[向量案例](examples/vector-ranking-scenarios.json)检查久远记忆和状态保护。配置是设计规格，需要开发者实现对应行为。

**单独粘贴提示词无法保证节省 token。** 当完整历史已经发给模型后，再要求它“只记重要的”并不会减少输入。真正的限制必须在请求发出前执行。

## 对话效果示例

以下为预期行为示例，不是模型实测结果。假设杭州咖啡店记录已经冷存。

> 用户：今天有点累。  
> 陪伴 AI：今天主要是哪件事让你累？  
> 行为：不主动搜索或提起过去的旅行。

> 用户：去年在杭州，下雨那天我们聊到的咖啡店，你还记得吗？  
> 行为：用户线索允许检索，只返回与杭州、下雨、咖啡店匹配的少量记录。  
> 陪伴 AI：你说的是河岸咖啡。那天你提过想找个能安静看书的位置。

> 用户：不是那家，是靠近车站的。  
> 行为：使用新线索修正检索；找不到就说明未定位到，不编造店名。单次召回不会自动让整段旅行成为常驻记忆。

## 仓库内容

| 文件 | 用途 |
| --- | --- |
| [docs/vector-ranking.md](docs/vector-ranking.md) | 事件向量、热区综合排序、冷库线索匹配与有限 Top N |
| [config/vector-ranking.example.json](config/vector-ranking.example.json) | 可选向量排序配置，不增加现有预算 |
| [examples/vector-ranking-scenarios.json](examples/vector-ranking-scenarios.json) | 久远事件、性格保护、去重、降级与删除的待验证场景 |
| [docs/design.md](docs/design.md) | 状态迁移、热度规则、预算、召回与更正 |
| [docs/persona-continuity.md](docs/persona-continuity.md) | 跨天性格、近期状态、具体细节与渐变规则 |
| [docs/graduated-cooling.md](docs/graduated-cooling.md) | 近期细节到短摘要再到冷库的过渡与容量处理 |
| [config/continuity.example.json](config/continuity.example.json) | 可选连续性配置，细分原有 hot 预算 |
| [examples/continuity-state.json](examples/continuity-state.json) | 虚构跨日状态与近期细节快照 |
| [examples/continuity-scenarios.json](examples/continuity-scenarios.json) | 跨午夜、重启、渐变与删除的待验证场景 |
| [docs/cold-storage-index.md](docs/cold-storage-index.md) | 类型、时间、地点等多入口索引与逐步检索 |
| [docs/coopanion-integration.md](docs/coopanion-integration.md) | 源码依据、接入位置、兼容性与实施顺序 |
| [docs/evaluation.md](docs/evaluation.md) | 验收条件与 token、费用、记忆质量评估 |
| [prompts/README.md](prompts/README.md) | 提示词的使用边界和输入输出约定 |
| [config/policy.example.json](config/policy.example.json) | 供未来实现读取的策略示例 |
| [examples/memories.json](examples/memories.json) | 虚构记忆记录与字段示例 |
| [examples/dialogue-archive.json](examples/dialogue-archive.json) | 三段虚构对话归档样本，包括没有生成记忆卡的闲聊；并非全部记忆卡的来源全集 |
| [examples/scenarios.json](examples/scenarios.json) | 可转为自动化测试的验收场景 |
| [examples/recall-plan.json](examples/recall-plan.json) | “西安那次”到“买相机那次”的筛选过程 |
| [REFERENCES.md](REFERENCES.md) | 参考项目、论文与差异说明 |

主要设计、接入、评估、参考与提示词均提供英文同名 `.en.md` 版本；示例 JSON 也提供 `.en.json` 对照版本。结构化配置的字段名为英文，中英两版共用。更新内容见[更新说明](CHANGELOG.md)。

## 开发顺序

- [x] 设计规则、提示词、示例数据与验收清单。
- [ ] 实现完整对话归档、来源索引、用户有效提及计数与状态更新。
- [ ] 接入 Coopanion / Cortico 的请求构造，阻止旧历史与通用文件工具绕过冷存策略。
- [ ] 接入预算器，保留正在执行的工具调用所需协议结构。
- [ ] 按统一日志回放，与现有 Coopanion 和仅截短历史的方案比较。
- [ ] 发布真实费用、缓存、延迟与召回质量结果后，再评估是否作为扩展发布。

## 参考与项目关系

[MemGPT](https://arxiv.org/abs/2310.08560)提供主上下文与外部存储的分层思路；[MemoryBank](https://github.com/zhongwanjun/MemoryBank-SiliconFriend)研究基于时间与强化的遗忘及陪伴记忆；[MemoryOS](https://github.com/BAI-LAB/MemoryOS)提供操作系统式记忆层级与热度管理。具体借鉴和实现差别见[参考说明](REFERENCES.md)。

本项目是面向 Coopanion 的独立方案，不代表 Pal-AI-Lab 官方计划或背书。源码分析锁定于 2026-10-25 核验的版本，后续版本需要重新确认接入点。仓库仅含自编说明、提示词与虚构数据，未复制上游源码；上游项目继续适用其各自许可。发布者可在确定分发许可后补充 LICENSE；本包没有替发布者选择开源授权条款。
