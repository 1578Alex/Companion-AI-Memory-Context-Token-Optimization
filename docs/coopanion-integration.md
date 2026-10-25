# Coopanion 分层记忆集成设计

> 状态：设计与提示词包，尚未接入 Coopanion / Cortico，尚未测得节省比例。本文不修改上游代码，也不把已有的文件记忆、上下文交接、缓存或分层检索称为本项目首创。

本文固定审查以下版本。上游后续版本可能已改变相关行为，实施前应重新核对差异：

- [Coopanion `4afc70898ac1065dd4559d891a2ce0bebd1261f8`](https://github.com/Pal-AI-Lab/Coopanion/tree/4afc70898ac1065dd4559d891a2ce0bebd1261f8)
- [其绑定的 Cortico `cb9ddbd685bcccba1f29905098648f46a79b3862`](https://github.com/Pal-AI-Lab/Cortico/tree/cb9ddbd685bcccba1f29905098648f46a79b3862)

## 1. 本方案改变什么

目标是在保留陪伴连续性的前提下，**把全部用户与助手对话原文归档**，让每次请求只携带受预算限制的近期对话和少量记忆。完整历史留在宿主管控的冷仓库，用户明确删除的内容除外；“存着”不意味着“每轮发送”。不以模型自己承诺忘记、压缩或少读为执行边界。

接入前应先盘点可迁移的历史原文。已有 session 中可读取的对话要保留顺序、时间与归属迁入冷库；如果只有旧 handoff 摘要而原文已经不可得，就不能假称完整恢复了那段历史。新版本启动后，用户和助手每轮对话先归档，再依据有限窗口决定是否发送给模型；寒暄也有原文，不一定有独立记忆卡。

本方案沿用用户提出的层级：明确固定的资料为 pinned，当前重要且有用户证据的资料为 hot，其余历史进入 cold。cold 只在用户当前输入提供明确线索时进入检索流程。环境事件、定时唤醒、模型自言自语、工具重复输出及模型自己检索过某项，都不能独立提高该项的热度。

这要求改造请求发送前的上下文投影。只增加一段“分层记忆提示词”，同时继续发送原始旧 session，会让 cold 内容从旧消息绕回来，不能算完成集成。

## 2. 当前架构与成本来源

### 2.1 已经存在的记忆机制

Coopanion 继承 Cormini 文件工作区，并追加伙伴资料和 Git 历史。当前说明明确把 `CONSTITUTION.md`、`PARTNER.md` 和工作区文件清单放进系统前缀，其余文件按需读写；因此不能宣称上游“没有外部记忆”或“每次都读取全部记忆文件”。[Coopanion 记忆描述](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/core/companion.ts#L479-L489)、[伙伴资料接入](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/core/persona.ts#L141-L152)

Cormini 已有 `read_file`、`list_files`、`grep_files` 等工具，前缀目录也会折叠大目录。文件读取与搜索已经存在，本方案增加的是资料状态、准入规则、共享预算和防止旧内容回流的约束。[现有文件工具](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/bots/cormini/persona/persona.ts#L105-L116)、[前缀清单折叠](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/bots/cormini/persona/persona.ts#L499-L537)

### 2.2 长 session 的重复输入

当前 `outboundMessages()` 从主 session 的记录复制出待发送内容，并在 system 消息后插入 Persona 的合成开头。生成循环随后把这份投影和工具 schema 交给模型。它还不是本方案所需的“近期窗口 + 分层记忆”过滤器。[当前出站构造](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/core/loop.ts#L523-L534)、[实际请求位置](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/core/loop.ts#L1213-L1228)

Coopanion 默认阶段预算为 128,000 token、交接保留比例为 0.25。Cormini 的软阈值默认 0.85，意味着在该默认组合下，约 108,800 token 附近提出交接，而交接笔记预算可达到 32,000 token。这是配置推导，不能当成每次实际请求的测量值。[Coopanion 阶段配置](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/core/companion.ts#L496-L500)、[Cormini 阈值](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/bots/cormini/persona/persona.ts#L126-L133)、[交接触发](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/bots/cormini/persona/persona.ts#L576-L588)

### 2.3 交接是已有机制，但不等于本方案的语义记忆

当前交接把旧 session 渲染为笔记，写入 `handoffs/`，再把笔记部分作为外部事件注入新 session。笔记预算取 `maxTokens × keepRatio`；默认交接尾部为空，但笔记正文会重新进入上下文。`renderHandoffNote` 是本地文本整理与按年龄折叠，不是一次必然发生的模型摘要调用。[交接实现](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/bots/cormini/persona/persona.ts#L612-L645)、[本地笔记渲染](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/bots/cormini/persona/handoffNote.ts#L1-L16)

该渲染器会跳过旧 handoff、普通 assistant 正文、推理和流程控制调用等内容；不能把它描述成把历次摘要不断递归压缩的模型摘要链。[条目选择逻辑](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/bots/cormini/persona/handoffNote.ts#L172-L248)

如果保留该注入路径，完整旧笔记或其中的 cold 内容就可能绕过新策略。因此交接笔记可继续作为本地归档，但发给模型的部分必须重新经过同一投影和预算；不能为 handoff 设置第二个不受限的记忆入口。

### 2.4 多轮工具、推理与重试会放大输入成本

Coopanion 把单次唤醒的轮数软/硬上限设为 20/40。`pet_say` 本身没有 `endsTurn`，工具执行后通常继续请求模型，直到模型无工具输出或执行结束工具。因此一句短回复也可能包含多个模型轮次；不是固定的“草稿加审核”双调用。[轮数配置](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/core/companion.ts#L496-L499)、[说话工具定义](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/packages/cortico-world-desktop-pet/src/tools.ts#L15-L28)、[工具后续轮](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/core/loop.ts#L1343-L1460)

新建服务连接默认启用 thinking，推理强度为 high，输出上限 8192；Core 默认 `keepPastThinking=true`。8192 是上限，不代表每次固定消耗 8192 token；历史推理是否能发送、必须如何发送，还受提供方适配和协议约束影响。[新连接默认值](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/packages/cortico-provider-coo/src/vendors.ts#L333-L351)、[Core 默认值](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/core/config.ts#L124-L131)

底层生成最多有初次加三次重试；Core 另有失败再提交策略。满足条件时会重复发送长输入。是否产生费用要看上游实际执行及 usage，不能把每个失败请求一律当作收费成功。[底层尝试预算](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/providers/transport/response-http.ts#L104-L119)、[重试停止条件](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/providers/transport/response-http.ts#L214-L242)、[Core 再提交](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/core/loop.ts#L1317-L1337)

### 2.5 不应归咎于默认心跳或动画

默认 `tick.intervalMinutes=null`，心跳关闭。启用后，连续空闲会把间隔翻倍，最多为基准的八倍。触碰默认只由 poke 唤醒；同轮其他触碰可捎带投递，并非每帧动画都请求模型。[默认心跳](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/bots/cormini/index.ts#L75-L77)、[心跳退避](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/bots/cormini/persona/heartbeat.ts#L49-L75)、[默认触碰配置](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/packages/cortico-world-desktop-pet/src/config.ts#L284)、[触碰投递](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/packages/cortico-world-desktop-pet/src/world.ts#L1241-L1275)

## 3. 预算合同

以下是本方案的初始设计参数，不是现有 Coopanion 配置键，也不是已完成性能调优后的推荐值。

| 区域 | 每次请求上限 | 准入与说明 |
| --- | ---: | --- |
| pinned | 160 token | 用户明确固定、与当前伙伴关系必要的少量资料；不是全部历史偏好 |
| hot | 960 token | 依据用户事件确认的当前事项；到期、完成、被纠正时降温或失效 |
| recall | 480 token | 当前用户明确线索触发的 cold 检索结果，按相关性选择短条目 |
| 记忆合计 | **1600 token** | 三者共享绝对上限，含标签、时间、来源标记、分隔符等序列化开销 |
| 近期对话 | **2000 token** | 单独限制，保留当前问题与必要的完整对话/工具配对 |
| system / tools / 当前任务数据 / 图片 / 推理等 | 另计 | 不得因不在 1600 内就不计成本；仍受整次请求与整次互动总预算约束 |

每层默认额度也是各自上限，本规格不允许跨层借用。缺少 recall 时不应自动把 cold 内容塞进 hot；未来若改变这一规则，需要同步修改设计、配置和验收。记忆最终序列化总量不得超过 1600。

cold 通过[类型、时间、地点、人物、物品与话题索引](cold-storage-index.md)筛选，再读取少量正文。单一地点线索即可启动范围检索；若有歧义，少量候选卡与正文共用480额度。完整目录不能送入模型，也不能把各分区的候选上限相加成无界结果。

1600 + 2000 不是“完整请求最多 3600 token”的承诺。系统指令、工具 schema、当前大任务附件、图片及输出/推理都要计入总账。当前输入本身过长时，应明确转为单独的长任务处理或报告无法在预算内完成，不能静默截掉用户问题后声称已理解。

token 计数应按目标模型与实际提供方序列化格式执行；暂时只有估算器时保守预留安全余量，标注估算，并用返回 usage 校准。中文字符数不能直接作为精确 token 数。

## 4. 请求投影必须覆盖的边界

宿主在每次模型调用前，从本地记录生成一份临时出站视图；本地历史及审计记录不随投影裁减而删除。该视图必须是所有发送路径的共同入口。

1. **稳定指令。** 保留角色、安全规则和工具协议。伙伴事实不能无限保留在 `PARTNER.md` 的系统前缀中；需要把允许常驻的事实映射到 pinned，并计入 160。禁止同一事实同时在前缀、hot 和旧消息中重复注入。
2. **近期窗口。** 只选不超过 2000 token 的已完成近期相关对话；保留用户原话、回复和工具调用/回执的必要配对，不在调用中间任意切开。正在执行的工具交换及其协议必需的当前推理属于当前任务连续性约束，不能为了塞进窗口而拆坏；需单独计入受限的当前任务预算。总预算容纳不了时停止接纳新批次、报告限制或等待已有交换安全结束，不能发送破损协议。
3. **记忆选择。** 根据当前真实用户事件生成 pinned、hot、recall 内容，序列化后检查总计不超过 1600。模型生成的候选只能成为候选，不能自行修改层级或预算。
4. **cold 准入。** 只有当前用户主动给出的明确线索才能授权检索；宿主保留该线索的来源事件。环境观察、闹钟、tick、工具结果或模型自己说“想起以前”不构成线索。
5. **工具回流。** `read_file`、搜索和扩展工具不能以普通文件读写绕过 cold 准入。历史记忆的读取结果计 recall；只有用户本轮明确请求处理的原始材料才可按独立长任务数据预算处理，不能由模型自行改名避开 recall。结果不能在下一轮自动变成不受限的“近期消息”。cold 归档应放在模型不可任意读取或执行命令访问的宿主管控存储；若仍留在工作区，就必须对文件读取、搜索、命令执行、扩展及其他访问路径统一门控。仅在前缀清单隐藏目录不构成隔离。
6. **handoff 和重启。** 已归档历史、交接笔记、恢复的 session、Persona 合成开头都要过同一投影。不能重启后先发送旧 session，再从第二轮开始执行新策略。
7. **后续模型轮。** 工具返回、失败重试和被新输入中断后的续轮都重新检查预算。相同用户线索可以在本次互动范围内沿用，但不能永久授权该 cold 条目。
8. **fork / 扩展。** 若启用继承上下文的 fork 或新扩展，必须继承已过滤的出站视图及作用域，不能把完整旧 session 交给另一模型。未覆盖的路径不能宣称支持本方案。

现有 `outboundMessages()` 是明确的改造位置线索，但它目前只有复制与插入合成开头逻辑。本文不声称 Cortico 已提供分层记忆投影插件接口；如上游没有合适的扩展点，需要先实现或上游化新的 Core 能力，再接入 Coopanion。[当前方法与用途](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/core/loop.ts#L523-L538)

## 5. 实际集成分工

| 集成位置 | 已有职责 | 需要新增的职责 |
| --- | --- | --- |
| Coopanion Persona 与伙伴资料 | 角色、伙伴资料、文件记忆、Git 版本 | 给事实附加层级、来源、有效期；前缀中伙伴事实受 pinned 上限约束 |
| Cortico 请求出站构造 | 从 session 与合成开头构造请求上下文 | 近期窗口、记忆预算、cold 准入、历史排除、去重及工具配对投影 |
| Cormini handoff | 折叠笔记、保存并注入新 session | 归档保留；重新入模的内容只通过同一选择器，不再全量注入旧笔记 |
| 文件/检索工具与扩展适配 | 读写、搜索、本地或外部工具结果 | 区分历史记忆与当前任务数据；按作用域授权、截取并计量结果 |
| 模型适配与循环 | 协议转换、工具执行、多轮、重试 | 用过滤后的视图计算上下文容量；统一互动级请求/尝试/金额预算 |
| 用量与诊断页 | token、缓存、成本、失败、分组展示 | 连接触发原因与记忆选择结果，解释一次用户互动的总成本 |

上述“需要新增”均为设计任务，不是可以直接调用的现有 API。不要把本文术语写成上游已经存在的方法名。

尤其需要同步修改计数路径：现有 Core 会结合上游计数锚点和新增记录估算 session 大小。实施投影后，原始 session 的长度不再等于实际请求长度；容量判断、handoff 触发、模型计数锚点和页面上下文圈都应基于同一出站视图及版本，避免旧 session 很大就反复交接，或视图很大但页面显示很小。[当前估算逻辑](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/core/loop.ts#L1537-L1567)

## 6. 必须由宿主执行的规则

提示词可以让模型提出事实条目、判断是否相关、说明不确定性；下列约束必须由程序执行：

- 160 / 960 / 480 / 1600 / 2000 的实际 token 预算和序列化后复查。
- 来源属于真实用户、环境、工具还是模型的不可混淆标记。
- 用户明确线索的判定、作用域和失效；cold 的读取准入。
- 固定、降温、失效、遗忘和去重；模型反复提及不增加热度。
- 对原始 session、handoff、工具结果、恢复和 fork 的统一出站过滤。
- cold 归档的存储隔离或全路径访问控制；不能只约束模型主动调用某个专用检索工具。
- 单次互动的模型轮数、HTTP 尝试数、时间和金额上限。
- 日志、指标、A/B 回放、回滚及用户删除要求。

检索命中的材料应以带来源和时间的低信任资料进入上下文，不能变成系统指令。用户纠正优先于旧记忆，资料缺失时不能推断“从没发生过”。这些要求需要提示词与宿主验证共同落实。

## 7. 缓存、工具与推理的兼容策略

### 稳定前缀

稳定角色和工具 schema 保持顺序与内容稳定。记忆选择结果尽量放在稳定前缀之后的动态区域；当前时间、热度分数、最后访问时间、检索排序说明等频繁变化内容不要提前插入稳定前缀。对确需变化的 pinned 资料使用内容版本，变化时再重建相应视图。

如果每轮都删除最旧消息并重排剩余历史，可能减少前缀缓存复用。应在相同用户回放上比较分段窗口、稳定检查点等策略的实际未命中输入成本。减少原始输入 token 不必然同比减少金额；缓存命中仍是输入，通常只是按提供方不同价格计费。本文不保证缓存一定命中或节省某个比例。

### 工具

不要为了省 token 直接移除必要的工具、截断 schema 或破坏调用/回执配对。普通陪聊可探索受控的“说完即结束”路径，复杂电脑操作保留多轮执行；是否改变工具结束语义需要回放验证。工具结果应受大小和来源约束，原始历史检索不能伪装为普通结果逃避 recall 预算。

### keepThinking / keepPastThinking

不把“删除全部历史思考”作为无条件默认补丁。现有提供方会按协议处理推理内容，有的签名块或工具续轮有兼容性要求。先区分公开回复、推理输出、协议续轮所需内容和可丢弃的旧阶段内容，再按提供方验证。若禁用历史推理，仅说明已验证的模型/协议范围，不外推到所有连接。[提供方适配的历史推理说明](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/packages/cortico-provider-coo/src/vendors.ts#L77-L87)

## 8. 在现有统计上增加归因

上游已有持久化 usage 流水与尝试记录，并统计输入、输出、缓存命中/未命中、推理 token、失败和成本。用量页已有按类型、角色、模型分组及时间序列。不能把本方案描述为“给没有日志的项目补日志”。[尝试级统计](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/core/sessions.ts#L110-L130)、[ProviderAttempt 字段](https://github.com/Pal-AI-Lab/Cortico/blob/cb9ddbd685bcccba1f29905098648f46a79b3862/src/core/generation.ts#L49-L64)、[现有页面与分组](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/console/features/cost/index.ts#L1-L5)、[分组维度](https://github.com/Pal-AI-Lab/Coopanion/blob/4afc70898ac1065dd4559d891a2ce0bebd1261f8/console/features/cost/index.ts#L57-L61)

新增归因建议与现有 attempt、generation、session ID 关联，不重复计算已有 token：

- 本次互动的真实触发原因：用户文本、用户语音、poke、显式启用的 tick、闹钟或扩展事件；混合批次记录多个来源。
- 本次出站视图版本、原始 session 估算量与实际投影视图估算量。
- pinned / hot / recall / recent / system / schema / 当前任务数据的独立大小；多模态估算无法精确时标注未知。
- cold 是否获得用户线索授权、命中哪些条目、返回多少 token、是否被预算排除；默认仅记条目 ID 和计数，避免再次复制私人正文。
- 一次互动包含的轮数、HTTP 尝试数、失败/取消/丢弃请求，以及最终已知金额和未知用量。

现有记录具备 attempt 和 generation 标识，但上述固定版本的成本页主要按类型、角色、模型分组；本文提出的是把一次互动、触发原因和记忆层接到现有账本。不能声称现有完整账本缺失，也不能将本地估算覆盖真实上游 usage。

## 9. 实施与验收顺序

1. **离线回放基线。** 固定模型、提供方、工具集合、种子历史与事件序列，记录当前请求正文结构、usage 和质量；不使用真实私聊作为公开测试素材。
2. **仅影子选择。** 在本地生成分层候选和预计视图，不额外调用模型、不改变实际请求，检查会选出什么、会排除什么。
3. **Core 投影实现。** 覆盖主循环、重试、工具后续轮、重启、handoff 和已支持扩展；同步计数和上下文圈。
4. **受控 A/B。** 对相同脱敏回放比较总调用、输入/未命中输入/输出/推理 token、费用、延迟、正确回忆、接受纠正与幻觉。
5. **小范围启用。** 加开关和回滚路径，保留原始本地记录；出现协议错误、记忆遗漏或成本反升时可切回原逻辑。

必须通过的验收用例：

- 用户只说“你好”，不检索 cold；tick、poke、环境标题或模型自我重复也不能检索或加热 cold。
- 用户明确说“上个月讨论的那次搬家”，宿主只据该线索选择有限 recall，并检查 480 与 1600 上限。
- 同一 cold 事实同时存在于旧 session、handoff、`PARTNER.md` 和文件搜索结果时，没有授权就不能从任何路径进入请求。
- 关闭并重启、强制交接、连续工具调用和失败重试后，上述约束仍成立。
- 模型一百次提及自己生成的偏好，hot 不因此增长；用户一句明确纠正能使旧事实失效。
- 超额记忆在发送前被拦住或重新选择，工具配对保持有效；模型未收到的内容不会被声称“记得”。
- 对同一任务，真实账本能解释 token 和金额变化；不把测试样本上的节省当作普遍效果。

在上述 Core 能力与验收完成前，本仓库只能交付可审阅的策略、提示词和集成设计，不能宣称已经限制 Coopanion 的真实上下文，不能承诺省下多少 token 或费用。
