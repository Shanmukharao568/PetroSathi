# Gnani integration notes

What PetroSathi relies on, where it came from, and what still needs your account to confirm.
Source: docs.gnani.ai, read **2026-11-08** (pages listed at the end). Nothing here has been run
against the live Gnani service yet. All automated tests use the mock provider or a **local stub**
that implements the documented protocol.

## Prisma speech-to-text (implemented, Phase 4)

| Item | Documented behaviour | Where in code |
|---|---|---|
| Auth | Single header `X-API-Key-ID` (lower-case `x-api-key-id` for WebSocket). No secret header is documented. | `providers/gnani/gnaniPrismaService.ts` |
| REST endpoint | `POST https://api.vachana.ai/stt/v3`, multipart | same |
| REST fields used | `audio_file` (file), `language_code` (required), `model` (optional) | same |
| Model | `prisma-v3.0` must be sent explicitly; omitting `model` selects Prisma v2.5 | `GNANI_PRISMA_MODEL` |
| REST limits | 30 s audio per request; formats WAV, MP3, OGG, FLAC, AAC, M4A; format detected from file extension | `voice.service.ts`, `utils/audioFormat.ts` |
| REST success | `{ success, request_id, transcript, model, detected_language_code?, truncated? }` | zod schema in adapter |
| REST errors | 400 `{ success:false, error:{ type, message } }`; 401/402/403/429 `{ detail:{ error_code, … } }`; 504 `UPSTREAM_TIMEOUT` | `classifyHttpError` |
| Retry policy | Retry 429/500/503/504 with backoff; never other 4xx | adapter loop (`GNANI_MAX_RETRIES`) |
| Streaming endpoint | `wss://api.vachana.ai/stt/v3/stream` | adapter `openStream` |
| Streaming config | **All config as upgrade headers**: `x-api-key-id`, `language_code`, `model`, `x-sample-rate` | same |
| Audio frames | Raw PCM s16le mono, 16 kHz, **exactly 1,024 bytes per binary frame**, real-time cadence | adapter re-frames, pads the last frame |
| Client control | `{"type":"commit"}` closes the open utterance. No end-of-stream message is documented; closing the socket ends the session. | `finish()` |
| Server messages | `connected`, `speech_start`, `processing`, `transcript` (final per segment, `is_final:true`), `error`, `settings_updated`. **No interim/partial transcripts.** | message switch |
| Session limits | 15 min per session, 60 s default idle timeout, per-org concurrency cap | app caps at `VOICE_MAX_STREAM_SECONDS` (default 120) |
| Languages | bn-IN, en-IN, gu-IN, hi-IN, kn-IN, ml-IN, mr-IN, pa-IN, ta-IN, te-IN | `config/languages.ts` |
| Auto-detect | `indic-auto`, Prisma v3.0 only | `autoDetectionAvailable()` |
| Code-mixed | `format=codemix` (v3.0). en-IN accepts English–Hindi mixed audio. *Not enabled yet:* PetroSathi sends the default `verbatim` format so technical terms are not rewritten. | — |

### PetroSathi ↔ Gnani streaming translation

```
Browser (AudioWorklet, PCM16LE 16 kHz)            PetroSathi API                      Gnani Prisma
 ─ WS /api/v1/voice/stream ─────────────▶  auth via subprotocol "bearer.<JWT>"
   {"type":"start","language":"hi-IN"}  ─▶  validate language ─────────────────▶  WSS + headers
                                         ◀─ session_ready   ◀── "connected" ──────
   binary PCM (any size, even length)   ─▶  re-frame to 1,024 B ───────────────▶  binary frames
                                         ◀─ transcript       ◀── "transcript" ─────
   {"type":"stop"}                      ─▶  pad last frame, {"type":"commit"} ─▶
                                            wait for trailing segments, close
                                         ◀─ completed (conversationId, joined transcript)
```

## Evon (LLM): not available, labelled mock only (Phase 5)

Checked again on 2026-11-08. docs.gnani.ai has **no Evon page**: nothing in `llms.txt`, and nothing on
the TTS pages. The only LLM-related pages belong to the separate Agent Builder platform. Following
the project rules, no endpoint or payload was invented.

- `providers/gnani/gnaniEvonService.ts` contains the **prompt** (safety rules, operator text marked
  as untrusted data, strict JSON contract) and **output validation** (`parseEvonResponseText` plus the Zod
  `ComplaintExtraction` schema, which ignores any equipment ID the model supplies). Both are tested. Only
  the HTTP transport is missing.
- Startup refuses `EVON_PROVIDER_MODE=real`. With `AI_PROVIDER_MODE=real` you must set
  `EVON_PROVIDER_MODE=mock` **explicitly**, so a mock never runs silently in place of Evon.
- The mock (`mockEvonService.ts`) uses keyword rules for en/hi/te/ta. It reports a severity only from
  explicit evidence: safety words give "critical", "minor" gives "low", anything else stays "unknown". The UI labels it
  "Mock extractor: keyword rules, not Gnani Evon".
- **To enable real Evon you need:** the endpoint URL, the auth header, the request and response format,
  the model name, and supported languages. Then implement the transport in `createGnaniEvonService` and
  remove the guard in `config/env.ts`.

## Timbre TTS (implemented, optional, Phase 5)

| Item | Documented behaviour | Where in code |
|---|---|---|
| Endpoint | `POST https://api.vachana.ai/api/v1/tts/inference`, header `X-API-Key-ID` | `providers/gnani/gnaniTimbreService.ts` |
| Body used | `text`, `model`, `voice`, `language`, `speed: 1.0`, `audio_config { sample_rate 24000, num_channels 1, sample_width 2, container "mp3", bitrate "64k" }` | same |
| Response | Raw audio bytes (no JSON); `audio/mpeg` for mp3 | adapter rejects non-audio 200s |
| Errors | 400/500 `{ success:false, message }`; 401/402/403/429 `{ detail:{ error_code } }` | same |
| Concurrency | "Only 1 active request is allowed at a time" | requests serialised, one retry on 429 |
| Models and voices | `timbre-v3.0`: 13 voices, all 10 languages. `timbre-v2.5`: 42 voices, one preferred language each | `config/timbreVoices.ts` (validated at startup) |

- TTS runs only in real mode. In mock mode it is **unavailable** (no fake audio) and the UI uses text.
- The server builds the spoken text from fixed templates (`complaints/questions.ts`), so the endpoint is not
  a general text-to-speech proxy. The en/hi/te/ta templates should be reviewed by native speakers.
- The text-normalisation page lists short language codes (`hi`, `en`, …), while the inference page uses
  `hi-IN` etc. PetroSathi sends the inference page's `-IN` codes. Check this with a live key.

## Documentation inconsistencies noticed

- Quick-start says REST sample rate "8 kHz to 44.1 kHz"; the REST/limits pages say audio is converted to 16 kHz mono.
- `indic-auto` is described as v3.0-only on the languages page, but the realtime page suggests v2.5 also accepts it. PetroSathi only offers auto-detect when `GNANI_PRISMA_MODEL=prisma-v3.0`.
- Quick-start realtime example shows `segment_id: "seg_abc123"`; the reference shows `<request_id>-seg-<N>`. PetroSathi does not depend on this field.

## Pages read

- https://docs.gnani.ai/api/introduction/quick-start
- https://docs.gnani.ai/api/introduction/introduction
- https://docs.gnani.ai/api/STT/speech-to-text.md
- https://docs.gnani.ai/api/STT/stt-websocket.md
- https://docs.gnani.ai/api/STT/languages.md
- https://docs.gnani.ai/api/STT/limits.md
- https://docs.gnani.ai/api/introduction/errors.md
- https://docs.gnani.ai/llms.txt
