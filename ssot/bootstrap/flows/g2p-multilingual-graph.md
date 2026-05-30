---
id: g2p-multilingual-graph
name: G2P — CharsiuG2P ByT5
trigger_type: user-action
trigger: A TTS preprocessor encounters a word not in its lexicon and calls `MultilingualG2PModel.shared.phonemize(word:language:)`.
end_state: An `[String]` of IPA phonemes is returned for the word; the encoder/decoder CoreML models stay loaded for the next call.
involves_features: [multilingual-g2p]
linked_flows: [model-download-and-load, kokoro-synthesize]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# G2P — CharsiuG2P ByT5

## TL;DR
Actor-isolated CharsiuG2P ByT5 wrapper. Byte-level tokenization (no vocab file required), greedy autoregressive decode up to 128 steps, returns IPA phonemes for one word in one of nine supported languages.

## Trigger & Preconditions
`MultilingualG2PModel.shared.phonemize(word:language:)`. Models lazy-loaded on first call from `~/.cache/fluidaudio/Models/kokoro/` with `computeUnits = .cpuOnly`; if loading fails and `CI` env is set, `phonemize` returns `nil` instead of throwing.

## Stages
1. **Lazy load** — `loadIfNeeded()` resolves encoder/decoder `.mlmodelc` paths and loads each with `.cpuOnly`: [`Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift:147`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift#L147).
2. **Build input** — `"<{language.charsiuCode}>: {word}"` (e.g. `"<eng-us>: hello"`) UTF-8 encoded; each byte `b` becomes token `b + 3` (ByT5 byte-level offset): [`Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift:51`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift#L51).
3. **Encoder forward** — single CoreML call.
4. **Greedy decode loop** — up to `maxDecodeSteps = 128`. Each step: rebuild `decoder_input_ids = [padTokenId = 0] + outputTokens`, run decoder, argmax the last-position logits to pick next token, stop on EOS (`1`): [`Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift:83`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift#L83).
5. **Detokenize** — output tokens inverse-mapped (`tokenId - 3`) back to bytes, joined as UTF-8, then split per Swift `Character` (grapheme cluster) into the phoneme array: [`Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift:126`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift#L126).

## Outputs
- `[String]?` of IPA phoneme grapheme clusters; `nil` if empty / UTF-8 decode fails / `CI` fallback hit.

## Error Modes
- `MultilingualG2PError.modelLoadFailed` if either `.mlmodelc` is missing on disk.
- `MultilingualG2PError.encoderPredictionFailed` / `.decoderPredictionFailed` on CoreML nil.
- `CI` env var causes silent `nil` return on load failure.
- 128-step cap silently truncates very long IPA outputs.
- Per-`Character` split assumes single-character-per-phoneme — combining diacritics may be split incorrectly [REVIEW].
- Single shared actor — calls serialize through the singleton; hot loops bottleneck.
- O(n²) per-word decode (no KV cache; full `decoder_input_ids` rebuilt each step).

## Prompts and Models Used

## Usage Metrics

## Changelogs
