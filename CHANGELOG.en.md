# Change notes

**中文：[更新说明](CHANGELOG.md)**

## 2026-10-30 · v0.4 Remembering on her own, and knowing when to stay quiet

This update aims to go beyond waiting for an explicit memory question and respond to “I'm exhausted today.” Past overtime does not establish today's cause. After remembering, she still needs to choose whether to ask gently or leave it unsaid.

- Add proactive companion mode with user opt-in, keeping cue-based recall as the default. Recall-only, deletion, and scope restrictions always take priority.
- Separate real experiences from the character's reflections and hypotheses. References need sources; guesses expire and cannot become facts or rewrite personality through repetition.
- Keep separate space for core personality, interaction habits, recent state, and unfinished details. Proactive events and reflections share the existing recall allowance; the memory block remains 1,600 tokens.
- Process new interactions only in the background, skipping runs with nothing new. The example allows at most two maintenance calls a day, counting failures, retries, and helpers. Usage and watermarks must survive restarts.
- Add bilingual designs, prompts, a profile, and 22 proactive-recall acceptance specifications.
- Inspect DSH's plugin and compaction interfaces and add bilingual integration guides. Plugin code, installation, and runtime tests remain unfinished.
- Keep internal recurrent model state as a separate experiment, without promising permanent personality or detail retention after context reset. Add related-work references.
- Add an optional Star invitation and update plans. Reading the project does not automatically act on a reader's account.

This is a proposal package with no measured savings percentage. The existing 55% remains a hypothetical calculation, not a test result.

## 2026-10-27 · v0.3 vector relevance and affective continuity

- Add host-side semantic-vector references to event units. Compute relevance to the current question, combine hot relevance with recency decay and qualifying mention frequency, then select bounded Top N.
- Apply no age or last-mention decay to cold recall. Keep exact time, place, object, and other index routes so a matching old record is not excluded merely for being old.
- Reserve separate space for personality and current interaction state outside ordinary-event eviction. Add recent affect causes, unresolved matters, and review rules. Mood can settle naturally without resetting relationship baselines overnight or declaring unfinished matters resolved.
- Synchronize Chinese and English designs, prompts, profiles, and acceptance cases, including vector versions, fallback, deduplication, deletion, and total cost accounting. The 1,600-token memory total and hot's 320/320/320 split remain unchanged.

This release updates the specification, not a running integration. Retrieval quality, convincing characterization, and token benefits still need real-conversation testing.

## 2026-10-27 · v0.2 design update

This update focuses on preserving the familiar companion from one day to the next while reducing repeated reading of old dialogue.

- Corrected the name to Da Fei Yu and synchronized the Chinese and English introductions.
- Explained the idea in plain terms: draw on human memory and ordinary storage optimization. Recent frequent matters, core personality, interaction habits, and recent state remain in the current context. Less-mentioned events gradually cool and return when the user supplies a clue.
- Added cross-day continuity: preserve the base persona, separate slowly evolving interaction habits from transient state, and save and restore the committed current version across restarts and session changes. Midnight does not clear the state.
- Retain a few recent phrases, distinctive details, and unfinished commitments. Older events progress through “details → short summary → cold archive.” All dialogue from both sides is archived from the start; cooling changes visibility.
- The memory block remains capped at 1,600 tokens. The optional continuity profile splits hot's existing 960 into 320 for state, 320 for recent details, and 320 for topical facts. Apply defined compression priorities under capacity pressure; see the design documents for the constraints.
- Added bilingual prompts, configuration, synthetic examples, and acceptance specifications covering midnight, restart, persona-version changes, time decay, graduated cooling, corrections, and deletion.

This release updates the design, prompts, and integration instructions; it is not running inside Coopanion. Structural and illustrative arithmetic checks do not replace cross-day model tests, and no measured savings percentage is available.
