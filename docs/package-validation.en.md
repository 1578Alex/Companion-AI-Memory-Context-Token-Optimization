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
