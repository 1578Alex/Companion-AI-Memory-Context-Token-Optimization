# Change notes

**中文：[更新说明](CHANGELOG.md)**

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
