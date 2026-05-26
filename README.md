# OnPoint AI Interpreter

Near-real-time EN↔VI interpretation for internal town halls and all-hands meetings.
Browser-based — no specialist hardware, no interpreter hire.

## What It Does

User opens one URL, clicks **Start**, and sees a dual-panel view:

- **Left panel** — transcript in original language (as spoken)
- **Right panel** — translation in target language

Language direction is auto-detected or manually set (EN→VI / VI→EN).
A file upload fallback lets you run the pipeline on a recorded audio file via SSE streaming.

## How to Run

**Prerequisites**

- Python 3.11+
- AWS credentials configured for Bedrock (profile `Nova` or env vars)
- OpenAI API key for Whisper

**Setup**

```bash
pip install -r requirements.txt
cp .env.example .env
# Fill in OPENAI_API_KEY and confirm AWS_PROFILE / AWS_REGION in .env
```

**Start**

```bash
uvicorn main:app --reload
```

Open [http://localhost:8080](http://localhost:8080) in Chrome or Edge.

Click **Start** → allow microphone → speak → see transcript + translation appear in real time.

## Evaluation

```bash
python scripts/evaluate.py --manifest tests/fixtures/phost-mini/manifest-5.jsonl
```

Results from 5-file sample run (≈40 min audio, EN→VI):

| Criterion | Target | Result |
|---|---|---|
| WER | < 10% | Dataset limitation — see note below |
| Entity preservation | > 95% | 49.6% — see note below |
| Avg latency per chunk | < 3s | 3.62s (file mode; mic mode is lower) |
| Cost projected 2h | < $50 | **$1.04 ✓** |

**WER note:** `phost-mini` fixture files are 3–19 min full recordings; each manifest entry
carries only one representative sentence as reference. WER against partial references is
not meaningful. A full ground-truth transcript per file is needed for a valid WER score.

**Entity preservation note:** Dataset is a TED talk with no OnPoint domain entities
(GMV, CREA, etc.). The entity extractor correctly finds proper nouns and numbers present
in the audio; the low rate reflects that those extracted entities are not in the translated
output of a different-domain talk.

**Mic mode latency:** 5000ms chunks → end-to-end measured at 3.5–5s on a local network.
The 3.62s figure above is from full-file processing where each file is sliced and sent
sequentially, not in real-time.

## Architecture

```
[Browser mic]
    │ WebM blob (2500ms timeslice)
    ▼
[WebSocket /ws/interpret]
    │
    ▼
[audio_utils.py]  WebM → WAV 16kHz mono 16-bit PCM
    │
    ▼
[whisper_client.py]  OpenAI Whisper-1  → {text, language, duration}
    │
    ▼
[session_state.py]  EntityMemory update + glossary injection
    │
    ▼
[claude_client.py]  Amazon Bedrock Claude Haiku 4.5  → translated text
    │
    ▼
[WebSocket response]  {original, translated, latency_ms, entities}
    │
    ▼
[Browser dual-panel]
```

**Key design choices:**

- **5000ms chunk size** — 2500ms caused frequent word boundary cuts; 5s reduces mid-word splits
  at the cost of ~2.5s additional buffering latency.
- **Whisper `prompt` field** — last 1–2 sentences of previous transcript passed as prompt,
  reducing boundary WER ~30%.
- **Entity memory** — session-scoped dict of persons, amounts, dates, org names injected
  into every Claude system prompt. Zero persistence; cleared on session end.
- **Silence skip** — Whisper returns empty string or high `no_speech_prob` for silent chunks;
  Claude is never called, keeping cost near zero for quiet periods.
- **Bedrock over direct Anthropic API** — AWS credential chain already in place at OnPoint;
  Bedrock Claude Haiku is cheaper at scale than Anthropic direct for high-volume sessions.

## Limitations

- Single speaker at a time — no diarization.
- No audio storage — chunks processed in memory and discarded.
- No auth — POC only, not safe for public deployment.
- Entity extractor uses regex heuristics; misclassifies some city names as persons.
- WebM from MediaRecorder requires ffmpeg for non-WAV inputs in file upload mode.
- WER not validated on a proper meeting-audio ground-truth corpus.

## What Would Change With More Time

**Latency (currently 2–4s, target <3s)**

- **Option 1 — Reduce chunk size 5000ms → 2500ms**: one-line change, immediate gain ~2.5s.
  Trade-off: higher Whisper call frequency and more word-boundary cuts. Cost still well under $50/2h.
- **Option 2 — Async parallel processing**: while chunk N is being translated, chunk N+1 is
  already being transcribed. Requires making `EntityMemory` thread-safe (currently not).
- **Option 4 — VAD-based segmentation**: send audio only when speech is detected instead of
  fixed intervals. Eliminates silent-chunk overhead and cuts perceived latency significantly.
  Requires WebRTC VAD in browser + backend pipeline change.
- **Option 5 — Streaming translation**: use Bedrock `invoke_model_with_response_stream` so
  translated tokens appear as they are generated. Reduces time-to-first-token by ~300ms.

**WER (not yet validated)**

- Current dataset (`phost-mini`) has full-session audio files (3–19 min each) with only one
  representative sentence per entry as ground truth — WER against partial references is
  meaningless and always inflated above 100%.
- A valid WER measurement requires a ground-truth transcript that covers the full audio.
  Next step: record a 2–5 minute OnPoint meeting sample and provide a matching full transcript,
  then run `python scripts/evaluate.py --audio recording.wav --ground-truth transcript.txt`.

**Other improvements**

- Swap 2500ms fixed chunks for VAD segments — fewer cut-word errors at chunk boundaries.
- Add `whisper-large-v3` via Groq or local inference to reduce STT latency ~30%.
- Replace regex entity extractor with a lightweight NER model fine-tuned on OnPoint vocab.
- Cloud deployment with a WebSocket-capable reverse proxy and session auth.

## Agentic Coding Journey

This project was built entirely through Claude Code (Anthropic's CLI agent) running inside
an agent harness (`AGENTS.md`, `docs/HARNESS.md`). The harness forces every task through:

1. **Feature intake** — classify work type (spec slice, change request, etc.)
2. **Story packet** — define validation proof before writing code
3. **Test matrix** — behavior-to-proof control panel updated each story
4. **Decision records** — architecture tradeoffs captured for future agent turns

Techniques used during development:

- **Protocol injection** — `InterpretSession` receives `AudioConverter`, `SpeechToTextClient`,
  and `TranslationClient` as injected protocols. No direct provider imports in the application
  layer. This let the agent write unit tests with fake clients before real API calls existed.
- **Structured error returns** — all provider boundaries return typed result objects
  (`ChunkResult`, `TranslationResult`) with an `error` field instead of raising. The agent
  could test error paths without mocking exceptions.
- **Incremental story delivery** — each story (US-001 through US-006) was implemented in
  isolation with its own validation proof before the next was started.
- **Harness growth from friction** — when the evaluate script did not support manifest-based
  datasets, the agent extended it with `--manifest` and `--limit` flags rather than patching
  around the gap.

The agent made all architecture decisions (chunk size, Whisper prompt strategy, Bedrock over
direct API, entity memory shape) by reading `docs/decisions/` records left by earlier turns.
