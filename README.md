# Voice Bot Optimized (design doc)

This folder describes a faster, continuous, speech-to-speech pipeline without button clicks. No files were modified in `voice-bot/`; this is a clean blueprint you can implement here.

## What the current `voice-bot` does (slow path)
- Frontend uses browser `SpeechRecognition` (Web Speech API), started by a button.
- Gets a one-shot transcript (`en-US`), then sends it via `POST /chat`.
- Backend Flask calls OpenAI chat (`gpt-4o-mini`), returns text.
- Frontend calls `POST /speech` to TTS (`gpt-4o-mini-tts` voice `onyx`), gets an MP3, and plays it.
- Latency sources: browser STT start/stop, two HTTP round trips, separate TTS call, and audio autoplay restrictions.

## Optimized continuous approach (high level)
- **Capture**: Use `MediaRecorder` with `audio/webm;codecs=opus` and `getUserMedia` once; keep a single stream alive.
- **Transport**: Stream audio chunks over a persistent WebSocket to the backend; avoid per-request setup.
- **VAD & silence**: Client-side silence timer (e.g., 1–1.5s) or server-side VAD (`webrtcvad`) to decide when a user utterance ends.
- **STT**: Server transcribes chunks incrementally (e.g., `faster-whisper` locally or OpenAI Realtime STT if available). Emit interim and final transcripts back on the same WebSocket.
- **LLM**: On final transcript, call chat model and stream tokens back over the WebSocket.
- **TTS**: Stream TTS audio (if provider supports) or chunked MP3 over WebSocket; play via a single `Audio` element, pausing mic capture during playback if needed.
- **Loop**: After playback, immediately resume capture. No buttons needed—auto-connect on page load.

## Suggested backend shape (FastAPI example)
- `GET /health` for readiness.
- `WS /ws/voice`:
  - Receives binary Opus chunks.
  - Optional JSON control messages: `{type:"pause"|"resume"|"stop"}`.
  - Emits JSON events: `{type:"transcript", text, is_final}`, `{type:"reply_chunk", text}`, `{type:"audio_chunk", data: <base64>}`, `{type:"error", message}`.
- Services:
  - STT worker (faster-whisper or provider API) consuming an audio queue.
  - Chat worker (OpenAI `gpt-4o-mini` or `gpt-4o` depending on quality/speed needs).
  - TTS worker (OpenAI `gpt-4o-mini-tts`) with optional chunking.

## Suggested frontend shape
- On load: open WebSocket, request mic once, start `MediaRecorder` with 250–500ms slices.
- Send each `dataavailable` blob directly to WebSocket.
- Track `isListening`, `isSpeaking`, and `connection` state; disable local mic sending during TTS playback to avoid feedback.
- Silence handling:
  - Client: reset a timer on every interim transcript; when timer fires, send `{type:"end_utterance"}`.
  - Server: optional VAD double-check before finalizing.
- Display interim/final transcripts live; render LLM reply stream as it arrives; play audio as chunks arrive or after full MP3 is ready.

## Why `recog.continuous = true` alone is not enough
- Browser `SpeechRecognition` is still single-session, non-streaming, with poor control and frequent auto-stops.
- You would still need to manage restarts (`onend`), echo cancellation during TTS, and network round trips.
- Switching to `MediaRecorder` + WebSocket streaming avoids these limitations and gives consistent control.

## Latency levers
- Keep WebSocket hot; avoid re-auth on each request.
- Use shorter VAD silence (1–1.2s) and smaller audio chunks (250ms) to reduce end-of-utterance lag.
- Stream LLM output (`stream=True`) and start TTS as soon as text is ready (or use partials if your TTS allows).
- Cache system prompts and reuse a conversation id to keep context small.

## Minimal file layout to implement here
```
voice_bot_optimized/
  backend/
    app.py           # FastAPI + WebSocket entrypoint (to build)
    requirements.txt # fastapi, uvicorn, webrtcvad, faster-whisper, openai, pydantic
  frontend/
    index.html
    script.js        # MediaRecorder + WebSocket client
    style.css
```

## Next steps to make it real
1) Scaffold backend: FastAPI WebSocket endpoint, stub STT/LLM/TTS calls, health route.  
2) Add client: MediaRecorder → WebSocket, silence timer, interim/final rendering, audio playback.  
3) Choose STT: local `faster-whisper` for speed (GPU if available) or provider STT with low latency.  
4) Add VAD and auto-reconnect logic.  
5) Measure E2E latency; tighten chunk size/silence thresholds accordingly.

