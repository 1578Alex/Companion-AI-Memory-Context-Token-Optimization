# Package Validation Record

Checked on 2026-10-25. This record covers consistency of the design, prompts, and synthetic examples. Coopanion was not run.

Completed checks:

- The package has 35 files. All nine JSON data and configuration files parse, as do four JSON examples embedded in prompt templates.
- The seven fictional memory records have unique IDs. Their source fields, version references, dates, and classification fields passed structural checks.
- The 31 acceptance scenarios have unique IDs. They specify behavior for future tests; they are not 31 passing runtime tests.
- All 92 internal relative file links resolve, and Markdown code fences are paired.
- Each language's examples contain seven fictional records, 31 scenarios, a two-turn Xi'an cue search plan, and three archived conversations with six messages in total (two conversations have no memory card), with matching structure.
- The example budget is internally consistent: 160 pinned + 960 hot + 480 recalled = 1,600 memory tokens; the recent-history window is 2,000; at most 12 global candidates enter selection; cards and bodies share a limit of three items.
- The “that time in Xi'an → the camera purchase” query plan refers to existing sample records and keeps event dates distinct from record creation dates.
- Independent document review led to fixes for the precedence of “only when I ask” over pinned and ongoing status, explicit handling of an ended matter, idempotency when several management actions occur in one turn, and the creation-date field on a recalled body.

Not completed or demonstrated by these checks:

- Integration into Coopanion or Cortico, or compatibility with running tools and extensions.
- Counting a serialized request with the actual model tokenizer and enforcing its token limit at runtime.
- Testing a real model's classification, cue interpretation, recall accuracy, or adherence to these prompts.
- Measuring provider usage, cache hits, latency, charges, or a percentage saving.

Static document and example checks cannot establish actual token savings. Logical partitions, cue permission, and outbound history projection remain implementation requirements. Future runtime validation should follow [the evaluation plan](evaluation.en.md).

## 2026-10-26: Optional continuity extension checks

- Added bilingual continuity design, prompts, a profile, synthetic state examples, and 14 proposed cross-day acceptance cases. All 14 repository JSON files parse; the new bilingual examples align in keys, structure, IDs, dates, and numeric values.
- Checked hot subbudgets of 320+320+320=960, item limits of 1+3+4=8, and an unchanged memory total of 160+960+480=1600. Checked the illustrative 8/24-hour decay and rolling-24-hour step calculations. No state controller was implemented or run.
- Relative file targets exist and Markdown fences are paired. The existing Chinese introduction only changes the incorrect name to Da Fei Yu, with the English name synchronized. Continuity sections and file links were added; other existing README wording matches the pre-change snapshot.
- Added requirements for restart restoration, source watermarks, state versions, recent wording, explicit corrections, recall-only limits, and deletion. P01–P14 are specifications, not passed runtime tests.

Not completed: host persistence and injection, real tokenizer measurements, cross-day model replay, blind evaluation, or cost measurement. These files do not establish improved character continuity for Da Fei Yu.

## 2026-10-27: Publication and graduated-cooling checks

- Synchronized bilingual explanations and change notes around recent events and current personality state entering context while old events cool gradually. Preserved the author's introduction and Xi'an example.
- Added graduated-cooling design, configuration, and C01–C06 specifications. All 50 package files were reviewed structurally; all 16 JSON files parse, local link targets exist, Markdown fences are paired, and README section names are unique.
- The 31 base, 14 continuity, and 6 cooling acceptance specifications have corresponding bilingual structures. They still await execution. The 72-hour detail period, at least 24 hours of summary eligibility, and seven-day normal-cooling grace are ordered consistently. Memory1,600 and hot960 budgets are unchanged.
- This review covers documents, parameters, and synthetic examples, not runtime integration, cross-day blind model evaluation, or cost experiments.

## 2026-10-27: v0.3 vector-ranking and affective-continuity checks

- All 55 package files, 19 JSON files, and four JSON examples inside prompts were checked. The JSON parses, 216 relative file links resolve, Markdown fences close, and README headings are unique.
- Chinese and English structures, numbers, and dates match across 31 base, 17 continuity, six cooling, and 14 vector cases. These are 68 requirements awaiting execution, not 68 passed runtime tests.
- Checked illustrative hot/cold score arithmetic, weights summing to one, the unchanged 1,600 total and hot's 320/320/320 split, and no age, last-mention, or heat input to cold ranking. Distinguished up to 24 internal index references from at most 12 fused body-selection candidates.
- Independent review led to separate hot/cold fallback and a strong-exact-evidence priority group so ordinary semantic candidates cannot evict an accurate old event solely for low vector similarity. Specifications also cover state restoration, recent affect sources, unresolved matters, and responses to new circumstances.
- Compared with the pre-update snapshot, the author's introduction, Xi'an example, budget explanation, hypothetical 55% calculation, conversation examples, and references remain unchanged. Added explanations, profiles, and prompts are synchronized across languages.

This checks documentation, parameters, synthetic arithmetic, and structure only. It does not implement a vector index, call an embedding model, or run Coopanion. Retrieval accuracy, convincing cross-day characterization, token accounting, and cost benefits remain unmeasured.
