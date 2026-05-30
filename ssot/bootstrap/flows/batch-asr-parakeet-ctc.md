---
id: batch-asr-parakeet-ctc
name: Batch ASR — Parakeet CTC
trigger_type: user-action
trigger: Caller invokes the CTC manager's `transcribe(...)` entry (e.g. `CtcZhCnManager.transcribe(audio:audioLength:)`) with a complete audio buffer.
end_state: A transcript `String` (plus optional timings) is returned from a greedy CTC decode; per-call manager state is empty for the next invocation.
involves_features: [sliding-window-asr-manager, ctc-decoder, parakeet-tokenizer]
linked_flows: [model-download-and-load]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Batch ASR — Parakeet CTC

## TL;DR
Same window-based front-end as TDT but the decoder is a stateless CTC greedy collapse: argmax-per-frame, drop blanks and consecutive duplicates, detokenize. Used by the Mandarin zh-CN 600M pipeline (`CtcZhCnManager`) and as the underlying acoustic check inside the vocabulary rescorer.

## Trigger & Preconditions
`CtcZhCnManager.transcribe(audio:audioLength:)` is the public Mandarin entry. ASR-managed CTC fallback also invokes `ctcGreedyDecode(...)` directly on the encoder log-probs. Models loaded via `model-download-and-load.md`. Audio is 16 kHz Float32 mono.

## Stages
1. **Audio normalize** — `AudioConverter.resample*` produces 16 kHz mono Float32: [`Sources/FluidAudio/Shared/AudioConverter.swift:14`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioConverter.swift#L14).
2. **Preprocessor + encoder** — `CtcZhCnManager.transcribe` runs the preprocessor and encoder CoreML models inside the actor: [`Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CTC/CtcZhCnManager.swift:11`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CTC/CtcZhCnManager.swift#L11).
3. **CTC decoder model** — output log-probs `[1, T, V]` are passed to `ctcGreedyDecode(logProbs:MLMultiArray, ...)`: [`Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CTC/CtcDecoder.swift:46`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CTC/CtcDecoder.swift#L46).
4. **Greedy collapse** — per-frame argmax, drop blanks (Mandarin: `blankId = 7000`; 110m: `blankId = 1024`), drop consecutive duplicates: [`Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CTC/CtcDecoder.swift:17`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CTC/CtcDecoder.swift#L17).
5. **Detokenize** — `decodeCtcTokenIds(...)` maps the surviving IDs through the SentencePiece vocab via the same logic as the streaming `Tokenizer`: [`Sources/FluidAudio/ASR/Parakeet/Streaming/Tokenizer.swift:19`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/Tokenizer.swift#L19).
6. **Return result** — assembled `String` plus optional `ASRResult` envelope.

## Outputs
- Transcribed `String`, optionally wrapped in `ASRResult` with token IDs and timings.

## Error Modes
- Long audio blocks the actor — `CtcZhCnManager.transcribe` is synchronous-within-actor; callers must chunk for very long clips.
- Greedy decode is broken for `ctc06b` per `CtcModels.swift` (a CoreML conversion artefact) — recommended use is via the vocabulary rescorer rather than direct greedy.
- Empty frames silently skipped (no argmax).

## Prompts and Models Used

## Usage Metrics

## Changelogs
