# Architecture Diagram — Lumos Smart Tutor Lamp

System-level architecture across the three repos in this workspace:

| Tier | Repo / path | Stack |
|---|---|---|
| Device firmware | `Smart-Tutor-Lamp/tutor_lamp/` | ESP32-S3, Arduino, FreeRTOS |
| Cloud backend | `Smart-Tutor-Lamp-backend/app/` | Python, FastAPI, asyncio |
| Web app | `Smart-Tutor-frontend/` | Next.js, Clerk |

---

## 1. System overview

```mermaid
flowchart LR

    subgraph LAMP["ESP32-S3 Lamp — tutor_lamp firmware"]
        MIC["INMP441 mic<br/>I2S RX 16 kHz"]
        CAM["OV5640 camera<br/>SVGA JPEG 25–40 KB"]
        BTN["Buttons + RGB LED<br/>cancel turn, scroll pages,<br/>pan / zoom graphs"]
        DSP["DSP chain<br/>NS → ALE → AGC → VAD"]
        WAKE["Edge Impulse wake word<br/>hey Lumos"]
        NETWS["net_ws — persistent WSS client<br/>type-byte binary protocol, ≤4 KB msgs<br/>up: AUDIO_CHUNK / IMAGE_PART / CANCEL<br/>down: STATE / AUDIO_OUT / TFT frames"]
        PROV["provisioning<br/>device_jwt in NVS"]
        SPK["MAX98357A speaker<br/>24 kHz PCM, half-duplex I2S"]
        TFTUI["tft_ui — 240×320 TFT<br/>pages, LaTeX frames, QR"]

        MIC --> DSP
        DSP --> WAKE
        WAKE -.->|"trigger record"| DSP
        DSP -->|"question audio"| NETWS
        CAM -->|"photo chunks"| NETWS
        BTN -->|"CANCEL"| NETWS
        BTN -.->|"scroll"| TFTUI
        NETWS -->|"TTS audio"| SPK
        NETWS -->|"frames + STATE"| TFTUI
        TFTUI -.->|"GRAPH_VIEW"| NETWS
        PROV -->|"pairing QR"| TFTUI
    end

    subgraph BE["FastAPI Backend — app/"]
        WSL["routes/ws_lamp.py<br/>WS endpoint /lamp/ws"]
        SESS["session.py<br/>audio + image accumulators,<br/>paced chunk senders"]

        PRER["1 — escalation_router<br/>image-quality retake check,<br/>deterministic starting tier"]
        GUARD["2 — input_guard<br/>distress / harm /<br/>prompt-injection screen"]
        ORCH["3 — orchestrator — TurnOrchestrator<br/>+ problem_memo (solve once, tutor many)<br/>+ session memory into prompt<br/>+ 2-tier escalation"]
        LLMP["4 — providers/llm_* (Gemini default)<br/>audio + image + prompt → JSON"]
        MATHV["5 — math_verifier<br/>recompute answer (numpy),<br/>balance chemistry"]
        OUTV["6 — output_validator + render_check<br/>PASS / REPAIR / REPLACE gate"]
        DISP["7 — dispatch<br/>audio leg + display legs<br/>run concurrently (asyncio.gather)"]

        PRER --> GUARD
        GUARD --> ORCH
        ORCH --> LLMP
        LLMP --> MATHV
        MATHV --> OUTV
        OUTV --> DISP

        MEMS["services/session_memory.py<br/>L1 recent / L2 summary / L3 profile"]
        TTSP["tts_piper<br/>speech → 24 kHz PCM"]
        LTX["latex_renderer<br/>LaTeX → RGB565 frames"]
        GRPH["graph_renderer<br/>function plots → JPEG"]
        CHEM["rdkit_renderer<br/>molecule steps → JPEG"]
        DOCP["document composer<br/>scroll-doc manifest + blocks"]
        TRACE["turn_trace — passive observer<br/>→ turn_traces (admin pipeline audit)"]
        WORKERS["offline workers (never on live path)<br/>topic_tagger, mistake_tagger,<br/>transcribe, embed, rollup"]
        HTTPR["HTTP routes<br/>device_*, pairing_info, ota,<br/>frontend_manager, admin, pilot, insights"]
        AUTHD["deps/clerk + device_jwt<br/>token verification"]

        WSL --> SESS
        SESS -->|"complete turn:<br/>WAV + JPEG"| PRER
        ORCH <-->|"load_context / record_turn"| MEMS
        DISP --> TTSP
        DISP --> LTX
        DISP --> GRPH
        DISP --> CHEM
        DISP --> DOCP
        DISP -->|"stream frames"| SESS
        SESS -->|"GRAPH_VIEW → re-render<br/>cached graph spec, no LLM"| GRPH
        ORCH -.->|"every pipeline phase timed"| TRACE
        HTTPR --> AUTHD
        AUTHD -.->|"gates /lamp/ws"| WSL
    end

    subgraph FE["Next.js Frontend"]
        PAGES["dashboard / devices /<br/>analytics / admin"]
        CONNECT["connect page — QR scanner<br/>→ pair confirm"]
        SIM["simulator — web bench<br/>mic + webcam, same brain"]
        LIBAPI["lib/api.ts<br/>Clerk-Bearer fetch"]

        PAGES --> LIBAPI
        CONNECT --> LIBAPI
        SIM --> LIBAPI
    end

    %% External services — free-standing so each sits near its callers
    GEM(["Google Gemini API<br/>multimodal LLM"])
    CLERK(["Clerk<br/>human identity"])
    REDIS[("Redis<br/>hot memory +<br/>revocation pub/sub")]
    SUPA[("Supabase Postgres<br/>devices, sessions, turns,<br/>user_memory, user_profiles")]

    NETWS -->|"UPLINK — wss /lamp/ws<br/>auth: device_jwt"| WSL
    SESS -->|"DOWNLINK — answer frames<br/>on the same WebSocket"| NETWS
    PROV -->|"HTTPS — register / pairing /<br/>OTA self-update"| HTTPR
    LIBAPI -->|"HTTPS /api/*<br/>Clerk Bearer token"| HTTPR
    PAGES -->|"sign-in / sessions"| CLERK
    CLERK -->|"user.deleted webhook"| HTTPR
    LLMP --> GEM
    MEMS --> REDIS
    MEMS --> SUPA
    TRACE --> SUPA
    WORKERS --> SUPA
    WORKERS -.->|"cheap tagging calls"| GEM
    HTTPR --> SUPA
    AUTHD -->|"verify Clerk JWT"| CLERK
```

**Key rule shown above:** the lamp never talks to Clerk — it only sees the backend's
HTTPS + WSS endpoints and authenticates with the backend-minted `device_jwt`. All
human identity lives in Clerk and is verified server-side.

---

## 2. Tutoring turn — end-to-end data flow

```mermaid
sequenceDiagram
    autonumber
    participant C as Child
    participant L as ESP32-S3 Lamp
    participant W as Backend /lamp/ws
    participant O as TurnOrchestrator
    participant G as Gemini
    participant P as Piper TTS

    C->>L: "Hey Lumos" + question
    L->>L: wake word fires → record until VAD end-of-speech
    L->>W: AUDIO_CHUNK ×N — PCM or ADPCM, then AUDIO_END<br/>(+ up to 5 images, each IMAGE_PART ×N + IMAGE_JPEG)
    W-->>L: STATE thinking → spinner page + LED
    W->>O: run_turn(audio WAV, images)
    O->>O: pre-route — image-quality check<br/>(blurry / dark → ask to retake, zero LLM spend),<br/>pick starting tier
    O->>O: input_guard — distress / harm /<br/>prompt-injection screen
    O->>O: load memory context (Redis L1, Supabase fallback)<br/>+ problem memo (answer key from earlier turns)
    O->>G: system prompt + history + WAV + JPEG (JSON mode)
    G-->>O: JSON {speech, display, confidence, analytics}
    Note over O: low confidence / hard query →<br/>tier-2 re-call (escalation)
    O->>O: math_verifier — recompute answer<br/>(numpy / atom counts)
    O->>O: output_validator + render_check —<br/>repair or replace off-contract reply
    W-->>L: STATE speaking + TFT_LATEX_LOADING<br/>skeleton (when equations are coming)
    par audio leg
        O->>P: speech text
        P-->>W: 24 kHz mono PCM
        W-->>L: AUDIO_OUT ≤4 KB every 85 ms<br/>(Opus / ADPCM / PCM per config)
        L->>L: speaker takes I2S from mic (half-duplex)
        W-->>L: AUDIO_OUT_END — lamp hands I2S<br/>back to the mic, wake word re-arms
    and display legs
        O->>W: LaTeX → RGB565 frames,<br/>graphs / molecules / documents → JPEG
        W-->>L: TFT_PART ×N + graph / chem / doc pages
    end
    L-->>C: spoken answer + equation on TFT
    opt student pans or zooms a graph
        L->>W: GRAPH_VIEW — graph id + new viewport
        W-->>L: re-rendered TFT_GRAPH_JPEG<br/>(cached spec, no LLM call)
    end
    O->>O: record_turn → Redis + Supabase (background)<br/>+ turn_trace saved for admin audit
```

---

## 3. Pairing flow — QR onboarding

```mermaid
sequenceDiagram
    autonumber
    participant L as Lamp
    participant B as Backend
    participant A as Next.js app
    participant U as Parent

    L->>B: POST /api/device/register (device_id + secret)
    B-->>L: pairing code
    L->>L: render QR on TFT
    U->>A: sign in via Clerk, open /connect, scan QR
    A->>B: POST /api/device/complete-pairing (Clerk Bearer)
    B->>B: devices.user_id ← parent account
    L->>B: poll /api/device/poll-pairing
    B-->>L: device_jwt → stored in NVS
    L->>B: open wss /lamp/ws with device_jwt
    Note over L,B: unpair / Clerk user.deleted →<br/>revocation via Redis pub/sub,<br/>WS closed with code 4402
```

---

## 4. Contracts worth remembering

- **Wire protocol:** type-byte-framed binary WebSocket protocol; every message
  ≤ ~4 KB in both directions — large payloads (JPEG up, TFT frames down) are
  app-layer chunked. **Uplink frames:** audio chunks (PCM or ADPCM 4:1),
  AUDIO_END, image chunks, CANCEL, AUDIO_RESET, GRAPH_VIEW (pan/zoom), PING.
  **Downlink frames:** STATE byte (idle / listening / thinking / speaking /
  error / unpaired → drives LED + pages), TTS audio + AUDIO_OUT_END, LaTeX /
  text / graph / chem / scroll-doc frames, TFT_LATEX_LOADING skeleton, PONG.
- **Audio out:** 24 kHz mono, ≤4 KB chunks paced at 85 ms — wire codec is Opus
  by default (ADPCM / raw PCM fallbacks); `AUDIO_OUT_END` is mandatory, it
  releases the half-duplex I2S bus back to the mic so the wake word re-arms.
- **Memory:** L1 verbatim recent turns (Redis→Supabase), L2 session-summary
  compaction, L3 cross-session user profile — shared by the lamp and the web simulator.
- **Ingest gates:** up to 5 images per turn, each capped at 256 KB (an over-cap
  or excess image is dropped but the turn still runs); audio 0.5 s min / 30 s
  max (truncated over cap); a new AUDIO_END cancels and awaits any in-flight turn.
- **Interactive graphs:** graph specs are cached per session — a lamp GRAPH_VIEW
  pan/zoom re-renders server-side with zero LLM spend (re-render calls coalesced).
- **Auth split:** Clerk owns humans (frontend), backend owns devices (`device_jwt`);
  the ESP32 never sees Clerk.
- **Safety + quality gates are code, not prompt rules:** `input_guard` screens
  incoming text (distress / harm / injection), `math_verifier` recomputes the
  model's arithmetic, `output_validator` + `render_check` repair or replace any
  off-contract reply before dispatch.
- **Observability:** `turn_trace` passively records every pipeline phase to
  `turn_traces` (with media in the `/blobs` static mount) for the admin
  visual-pipeline audit; it can never fail or slow a live turn.
- **Offline enrichment:** topic tagging, mistake-type tagging, transcription,
  embedding, and analytics rollups run as background workers — never on the
  live turn path.
- **OTA:** the lamp polls `GET /lamp/ota/version` at boot and self-flashes when
  the backend advertises a newer firmware.
