# Internal recurrent state: an experiment for local companions

Test whether preserved model state can continue a familiar voice and interaction style while rereading less text. This repository does not export state, modify an inference engine, or report completed experiments. Fixed-size state is not established permanent personality memory.

## Known structure and unproven behavior

The [official Qwen3.8-27B card](https://huggingface.co/Qwen/Qwen3.8-27B/blob/main/README.md) lists 16 groups of three Gated DeltaNet layers and one Gated Attention layer, for 64 layers. Recurrent state coexists with full attention. The original [Gated DeltaNet research](https://research.nvidia.com/publication/2025-04_gated-delta-networks-improving-mamba2-delta-rule) supports efficient state updates; it does not establish stable personality after other context is cleared.

Retained hidden state carries history in another form. It is neither zero history nor a weight update that teaches the model a user or personality. Fixed capacity can still overwrite, interfere with, or lose information. Personality and events may not be separable, so do not promise to retain exactly the character while forgetting exactly the details.

## Verify restoration before partial clearing

- First export and restore every state component required by a fixed model, quantization, backend, and template. Compare uninterrupted and restored generation. Fingerprint the model, quantization, tokenizer, chat template, cache layout, and backend.
- A hybrid engine may also need convolution state, full-attention KV, positions, and sequence metadata. Retaining only recurrent layers is a separate modification, not an assumed standard API. Do not splice inconsistent branches or reset positions arbitrarily.
- Ordinary hosted chat APIs generally do not expose raw state import/export. Start with a local engine that explicitly supports it and verify the actual version. Binary snapshots still require persistence and device resources; they do not remove all external storage cost.

## Four comparison arms

Using the same model, sampling settings, and new user input, compare a fresh state without persona replay; recurrent-state-only restoration; full-state restoration; and the current short-persona plus continuity-summary approach. Separate the base model's default voice from traits acquired through interaction.

Start with small reproducible scripts, then test cross-day gaps, restart, extensive unrelated dialogue, mood changes, and explicit correction. Measure character consistency, detail accuracy, invented shared experiences, restore latency, RAM/VRAM, and total input/output. Matching seeds help control sampling, but different backends need not produce token-identical output; that alone proves neither success nor failure.

## Relationship to the usable memory design

Retain current personality, state, and details under the [continuity design](persona-continuity.en.md). Keep traceable facts and original archives, with [proactive association](proactive-memory.en.md) bounded by permission and budget. Recurrent state is a candidate enhancement, not a replacement for provenance, correction, deletion, or original records.

A snapshot can contain deleted or restricted information. Deleting a database row does not establish removal from that state. Replay permitted content from a clean checkpoint or reset the state; disable the snapshot if this cannot be done. An incompatible upgrade falls back to the short state rather than allowing old binaries to override a new character.

Until validated, describe this as a research direction, without zero-prompt permanent-memory claims, saving percentages, or claimed DSH compatibility.
