# Architecture Diagram — Lumos Smart Tutor Lamp

System-level architecture across the three repos in this workspace:

| Tier | Repo / path | Stack |
| --- | --- | --- |
| Device firmware | `Smart-Tutor-Lamp/tutor_lamp/` | ESP32-S3, Arduino, FreeRTOS |
| Cloud backend | `Smart-Tutor-Lamp-backend/app/` | Python, FastAPI, asyncio |
| Web app | `Smart-Tutor-frontend/` | Next.js, Clerk |

---

## 1. System overview

```mermaid
flowchart TB

    subgraph LAMP["💡 Smart Tutor Lamp — ESP32-S3 firmware"]
        BTN["Buttons<br/>(cancel / scroll / zoom)"]
        CAM["OV5640 camera<br/>(JPEG snapshot of the page)"]
        MICD["INMP441 mic + on-device DSP<br/>(wake word, NS, ALE, AGC, VAD)"]
        FWO["Firmware orchestrator<br/>(state machine + net_ws,<br/>ADPCM codec)"]
        TFTN["240×320 TFT<br/>(text / LaTeX / graphs / images)"]
        SPKN["MAX98357A speaker<br/>(24 kHz, half-duplex I2S)"]

        BTN --> FWO
        CAM --> FWO
        MICD --> FWO
        FWO --> TFTN
        FWO --> SPKN
    end

    subgraph FE["🌐 Human-facing web — Next.js"]
        SIM["Web simulator<br/>(browser stand-in for the lamp)"]
        DASH["Dashboard — home<br/>(profile glance, analytics cards)"]
        DEEP["Analytics / Devices / Admin<br/>(charts, manage lamps, turn traces)"]
        QRP["QR pairing<br/>(in-browser scanner)"]
        ONB["Onboarding<br/>(learner profile)"]
    end

    CLERK(["🔐 Clerk<br/>human identity / sessions"])

    subgraph BE["🧠 FastAPI Backend — the brain"]
        WSE["Binary WebSocket endpoint<br/>(/lamp/ws, one per lamp)"]
        AUTHP["Auth + pairing<br/>(device_jwt / device_secret)"]
        FMAPI["Frontend Manager API<br/>(profile, analytics, admin, insights)"]
        PRE["Pre-router<br/>(image check, deterministic escalation)"]
        ORCH["Turn orchestrator<br/>(guard stack, math verifier,<br/>problem memo)"]
        MEMN["Memory<br/>(L1 Redis / L2 compaction / L3 profile)"]
        TTSN["TTS engine<br/>(Piper / Gemini, 24 kHz)"]
        RENDN["Display renderers<br/>(LaTeX → RGB565, graphs,<br/>molecules, scroll-docs)"]
        TIER["Tiered model selection<br/>(Flash → Flash-thinking → Pro)"]
        OUTS["Answer streamer — session.py<br/>(voice + visuals in parallel,<br/>paced ≤4 KB chunks)"]

        WSE --> PRE
        PRE --> ORCH
        ORCH <--> MEMN
        ORCH --> TTSN
        ORCH --> RENDN
        ORCH --> TIER
    end

    SUPA[("🗄️ Supabase Postgres<br/>devices · sessions · turns · turn_traces<br/>user_memory · user_profiles")]
    REDIS[("⚡ Redis<br/>hot memory + revocation pub/sub")]

    subgraph LLMS["🤖 LLM providers — swappable"]
        GEMP["Gemini 2.5<br/>Flash / Pro"]
        OAIP["OpenAI"]
        CLAP["Claude"]
        DSKP["DeepSeek"]
    end

    %% invisible links — keep the vertical order: web → lamp → backend
    QRP ~~~ CAM
    ONB ~~~ MICD

    %% lamp ↔ backend
    FWO -->|"WSS uplink — question audio<br/>(PCM/ADPCM) + JPEG"| WSE
    OUTS -->|"WSS downlink — paced 24 kHz<br/>audio + TFT frames"| FWO
    FWO -->|"HTTPS — register → QR → device_jwt<br/>+ OTA poll / self-flash"| AUTHP

    %% web ↔ backend
    QRP -->|"scan QR →<br/>complete pairing"| AUTHP
    DASH <-->|"Clerk JWT on every call —<br/>profile + analytics JSON back"| FMAPI
    DEEP <--> FMAPI
    ONB -->|"profile save"| FMAPI
    SIM <-->|"/solve — question up,<br/>answer back (text + KaTeX)"| FMAPI
    FMAPI -->|"web turns — same pipeline"| ORCH

    %% identity
    DASH -->|"sign-in / sessions"| CLERK
    CLERK -.->|"user.deleted webhook (Svix)"| AUTHP
    FMAPI -.->|"verify Clerk JWT"| CLERK

    %% answer out — dispatch to the asking surface
    TTSN -->|"voice"| OUTS
    RENDN -->|"visuals"| OUTS

    %% data + models
    AUTHP <-->|"devices +<br/>pairing codes"| SUPA
    AUTHP -.->|"revocation publish<br/>→ WS close 4402"| REDIS
    FMAPI <-->|"profile / devices / turn stats<br/>(user-scoped reads + profile writes)"| SUPA
    MEMN <-->|"read context / write turns<br/>(writes fire-and-forget)"| SUPA
    MEMN <-->|"hot read +<br/>write-through"| REDIS
    SIM -.->|"turns tagged<br/>source=simulation"| SUPA
    TIER -->|"multimodal LLM calls"| LLMS
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
