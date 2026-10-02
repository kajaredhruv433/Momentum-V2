# Momentum Architectural Specifications: Version 2.1 vs Version 2.2

## Executive Architectural Summary

Momentum is an autonomous Agentic AI recruitment and live voice interview platform. The platform has evolved across two major release architectures:

- **Version 2.1 (Foundation Architecture)**: Initial release introducing real-time multi-role AI interview panels, dual-model LLM routing (Groq and Google Gemini), Agora Conversational AI audio transport, and an end-to-end recruitment portal with MongoDB persistence.
- **Version 2.2 (Enterprise No-Agora Architecture)**: Engineered with internal low-latency WebAudio pipelines, direct ElevenLabs Realtime Scribe STT WebSocket streaming, ElevenLabs Flash v2.5 Neural TTS, an externalized multi-tier reasoning brain (`otheragent.py`), multi-key token-bucket rate limit rotation (1 to 20 keys per provider), Qdrant Cloud semantic vector indexing (RAG), an interactive System Design Canvas with Multimodal Vision analysis, and an authoritative server-side monotonic timer.

---

## Strategic Evolution: Why Version 2.2 Replaced Agora with Direct ElevenLabs

### 1. Cost Optimization & Telephony Elimination
In Version 2.1, routing voice streams through Agora Conversational AI introduced substantial vendor telecommunication overhead, per-minute usage billing, and third-party bridging hops. In Version 2.2, direct implementation of ElevenLabs Realtime Scribe STT and Flash v2.5 Neural TTS over native HTML5 Web Audio WebSocket streams eliminated Agora's infrastructure costs completely, making the platform dramatically more cost-efficient and scalable for high-volume enterprise interviewing.

### 2. Superior Audio Fidelity & Conversational Responsiveness
By bypassing intermediary audio gateways, Version 2.2 directly connects the candidate's browser (via a custom 16kHz mono `AudioWorklet`) to ElevenLabs. This architectural shift resulted in:
- Significantly lower speech-to-text turnaround latency.
- Direct streaming MP3 neural synthesis with nuanced conversational inflection across 21+ voice personas.
- Instantaneous client-side Web Audio buffer flushing on speech onset (sub-150ms barge-in interruption).

### 3. Advanced LLM Cognitive Decision-Making
Version 2.2 redesigned the reasoning core into a unified, token-routed brain (`otheragent.py`):
- Dynamic continuation decisions (`CONTINUE`, `CONCLUDE`, `DRAW`) before each question.
- Uncovered competency tracking across job requirements and resume projects.
- Context-aware dynamic speaker selection among dynamically generated panel members.
- Automated rate-limit protection via multi-key token bucket rotation across up to 20 keys per provider.

---

## 1. Momentum Version 2.1 Architecture (Foundation Architecture)

![Momentum Version 2.1 Architecture Diagram](../architecture/v2-1-system-architecture.png)

### Version 2.1 System Architecture Flowchart (Mermaid)

```mermaid
flowchart TD
    subgraph CandidateClient ["Candidate Interface"]
        CAND["Candidate User"]
        MIC_CAM["Microphone & Webcam Inputs"]
        SPK_V21["Audio Output / Speaker"]
        CAND <--> MIC_CAM
        CAND <--> SPK_V21
    end

    subgraph AgoraVoice ["Agora Conversational AI Layer"]
        AGORA_RTC["Agora RTC Audio Transport"]
        AGORA_STT["STT (via Agora Engine)"]
        AGORA_LLM["Agora LLM (OpenAI Relay)"]
        AGORA_TTS["Neural TTS (ElevenLabs / MiniMax via Agora)"]
        
        AGORA_RTC --> AGORA_STT
        AGORA_STT --> AGORA_LLM
        AGORA_LLM --> AGORA_TTS
        AGORA_TTS --> AGORA_RTC
    end

    subgraph FastAPIServer ["FastAPI Backend Core"]
        AUTH_STATE["Authoritative State Machine"]
        SRV_TIMER["Server-Side Monotonic Timer"]
        WS_BUS["WebSocket Event Bus"]
        JD_RESUME["JD & Resume Parsing Engine"]
        
        AUTH_STATE <--> SRV_TIMER
        AUTH_STATE <--> WS_BUS
        AUTH_STATE <--> JD_RESUME
    end

    subgraph DecisionBrain ["Interview Logic & Decision Brain"]
        GATEWAY["otheragent.py Gateway"]
        GROQ_V21["Groq LPU (Fast Prompting & Question Refinement)"]
        GEMINI_V21["Google AI Studio Gemini (Large Context & Review Generation)"]
        CANVAS_AI["Canvas & Drawing AI (Image Processing for System Design)"]
        
        GATEWAY <--> GROQ_V21
        GATEWAY <--> GEMINI_V21
        GATEWAY <--> CANVAS_AI
    end

    subgraph DatastoresV21 ["Datastores & Persistence"]
        MONGO_V21["MongoDB Atlas (Interviews, Jobs, Candidate Applications)"]
        QDRANT_V21["Qdrant Cloud (Semantic Resume Vector Chunks)"]
    end

    subgraph PostSynthesis ["Post-Interview Synthesis"]
        REPORT_GEN["Detailed Review & Match Score Compilation"]
        DECISION_OPTS["Hiring Decision (Strong Hire, Hire, Consider, Do Not Hire)"]
        REPORT_GEN --> DECISION_OPTS
    end

    %% Inter-subsystem connections
    MIC_CAM -->|WebRTC Audio Stream| AGORA_RTC
    AGORA_RTC -->|Audio Playback Stream| SPK_V21
    AGORA_RTC <-->|Control Signals| AUTH_STATE
    AGORA_LLM <-->|Turn Instructions| GATEWAY

    JD_RESUME -->|Extracts Info to Define Panels & Areas| GATEWAY
    AUTH_STATE <-->|Sync State & History| GATEWAY
    WS_BUS -->|Canvas Data| CANVAS_AI

    AUTH_STATE <-->|Persist State & History| MONGO_V21
    JD_RESUME -->|Index Resume Chunks| QDRANT_V21
    GATEWAY -->|Retrieve Chunks for RAG Context| QDRANT_V21

    AUTH_STATE -->|Final Transcript & Canvas Output| REPORT_GEN
    GEMINI_V21 -->|Rubric Evaluation & Quotes| REPORT_GEN
    REPORT_GEN -->|Persist Completed Scorecard| MONGO_V21
```

---

## 2. Momentum Version 2.2 Enterprise Architecture (No-Agora Internal Pipeline)

![Momentum Version 2.2 Enterprise Architecture Diagram](../architecture/v2-2-system-architecture.png)

### Version 2.2 System Architecture Flowchart (Mermaid)

```mermaid
flowchart TD
    subgraph ClientLayer ["Candidate Interface (Browser)"]
        CAND_22["Candidate User"]
        MIC_22["16kHz PCM Mono Mic Capture (AudioWorklet)"]
        SPK_22["Web Audio MP3 Streaming Player"]
        CANVAS_22["Interactive System Design Drawing Canvas"]
        TRANS_22["Live Scrolling Transcript & Monotonic Clock"]
        BARGE_22["Client-Side Barge-In & Interruption Flush"]

        CAND_22 <--> MIC_22
        CAND_22 <--> SPK_22
        CAND_22 <--> CANVAS_22
        CAND_22 <--> TRANS_22
        MIC_22 -.->|Speech Detected| BARGE_22
        BARGE_22 -.->|Instant Playback Halt| SPK_22
    end

    subgraph VoiceEngine ["Internal Voice & Audio Processing Engine"]
        RTC_CTRL["Real-time WebAudio / WebSocket Controller"]
        EL_STT_22["ElevenLabs Realtime STT API (scribe_v2_realtime)"]
        EL_TTS_22["Direct ElevenLabs Neural TTS API (eleven_flash_v2_5)"]
        VOICE_REG["Persona Voice Registry (21+ Dynamic Profiles)"]

        RTC_CTRL -->|Direct Stream| EL_STT_22
        EL_STT_22 -->|Committed Transcript| RTC_CTRL
        VOICE_REG -->|Voice ID Parameter| EL_TTS_22
        EL_TTS_22 -->|Chunked MP3 Stream| RTC_CTRL
    end

    subgraph Orchestrator ["AI Interview Orchestrator & Agent Gateway"]
        STATE_MACH["Authoritative State Machine"]
        SRV_CLOCK["Authoritative Monotonic Server Timer"]
        WS_HUB["WebSocket Event Bus (/events & /audio-stream)"]
        PANEL_GEN["Dynamic Panel Formulator (panel.py)"]

        STATE_MACH <--> SRV_CLOCK
        STATE_MACH <--> WS_HUB
        STATE_MACH <--> PANEL_GEN
    end

    subgraph LLMUnits ["LLM & Logic Processing Units"]
        AGENT_GW["otheragent.py Gateway & Primary AI Agent"]
        GROQ_LPU_22["Groq LPU Engine (Sub-Second Conversational Turn Decision)"]
        GEMINI_22["Google AI Studio Gemini (Large Context Ingestion & Report Synthesis)"]
        CANVAS_VISION["Canvas & Drawing AI (Multimodal Vision Engine)"]
        TOKEN_ROUTER_22["Token Counter (Threshold: 7,500 Tokens)"]
        KEY_POOL_22["Multi-Key Token Bucket Rotation (1-20 Keys per Provider)"]

        AGENT_GW --> TOKEN_ROUTER_22
        TOKEN_ROUTER_22 -->|"<= 7.5k Tokens"| GROQ_LPU_22
        TOKEN_ROUTER_22 -->|"> 7.5k Tokens"| GEMINI_22
        GROQ_LPU_22 <--> KEY_POOL_22
        GEMINI_22 <--> KEY_POOL_22
        AGENT_GW <--> CANVAS_VISION
    end

    subgraph DataStorage ["Datastores & Vector Search"]
        MONGO_22["MongoDB Atlas (Persistent Interviews, Jobs, Applications, Scorecards)"]
        QDRANT_22["Qdrant Cloud (Semantic Resume Vector Chunks & RAG)"]
    end

    subgraph SynthesisStage ["Post-Interview Synthesis & Recruiter Ranking"]
        SCORECARD["Multi-Dimensional Match Scorecard (0-100)"]
        RECRUITER_LB["Recruiter Leaderboard (Ranked Strictly Descending by Match %)"]
        DECISION_EVIDENCE["Verbatim Transcript Quotes for Strengths & Concerns"]

        SCORECARD --> DECISION_EVIDENCE
        SCORECARD --> RECRUITER_LB
    end

    %% Cross-layer Signal Flows
    MIC_22 -->|"16kHz PCM Stream"| WS_HUB
    WS_HUB -->|"Relay Stream"| RTC_CTRL
    RTC_CTRL -->|"Transcribed Utterance"| STATE_MACH
    STATE_MACH -->|"Trigger Turn Evaluation"| AGENT_GW
    
    AGENT_GW -->|"Direct Spoken Question & Voice ID"| EL_TTS_22
    RTC_CTRL -->|"MP3 Audio Stream"| SPK_22
    
    CANVAS_22 -->|"Base64 PNG Image"| WS_HUB
    WS_HUB -->|"Image Data"| CANVAS_VISION
    CANVAS_VISION -->|"Extracted Topology, Flows & Bottlenecks"| AGENT_GW

    STATE_MACH <-->|"Persist Working State"| MONGO_22
    STATE_MACH <-->|"Retrieve Grounding Context"| QDRANT_22
    
    SRV_CLOCK -->|"1Hz Monotonic Broadcast"| WS_HUB
    WS_HUB -->|"Clock Synchronization"| TRANS_22

    STATE_MACH -->|"Final Session Context"| SCORECARD
    GEMINI_22 -->|"Evaluation Rubric Compilation"| SCORECARD
    SCORECARD -->|"Save Completed Record"| MONGO_22
```

---

## 3. Multi-Agent LLM Routing Hierarchy (`otheragent.py`)

![Multi-Agent LLM Routing Hierarchy](../architecture/llm-routing-hierarchy.png)

### LLM Router Flowchart (Mermaid)

```mermaid
flowchart TD
    REQ_MODULE["Requesting Module (normal_stage, extended_stage, final_stage, report)"] -->|"Prompt + Total Tokens"| ROUTER["otheragent.py (LLM Router Logic)"]
    
    ROUTER --> TOKEN_CHECK{"TOKEN CHECK: Prompt Tokens > 7.5k?"}

    %% Gemini Path (Long Context)
    TOKEN_CHECK -->|"Yes (> 7.5k Tokens)"| GEMINI_RTR["geminirouter.py (Google AI Studio)"]
    subgraph GeminiPool ["Gemini Multi-Key Pool (1 to 20 Keys)"]
        GEMINI_KEYS["20 API KEYS (.env)"]
        GEMINI_LIMITS["Limit Checks: Day (20 Max Req), Per-Min (120k+ TPM), Concurrency"]
        GEMINI_KEYS --> GEMINI_LIMITS
    end
    GEMINI_RTR <--> GeminiPool
    GEMINI_RTR -->|"Selected Key + Prompt"| GEMINI_RESP["geminiresponse.py (Gemini 2.5 Flash / 1.5 Flash)"]
    GEMINI_RESP -->|"Model Output"| ROUTER

    %% Groq Path (Sub-Second Fast Turn)
    TOKEN_CHECK -->|"No (<= 7.5k Tokens)"| GROQ_RTR["groqrouter.py (Groq Cloud LPU)"]
    subgraph GroqPool ["Groq Multi-Key Pool (1 to 20 Keys)"]
        GROQ_KEYS["20 API KEYS (.env)"]
        GROQ_LIMITS["Limit Checks: Per-Min (8k TPM), Active Concurrency"]
        GROQ_KEYS --> GROQ_LIMITS
    end
    GROQ_RTR <--> GroqPool
    GROQ_RTR -->|"Selected Key + Prompt"| GROQ_RESP["groqresponse.py (Llama-3.3-70b-versatile / Gpt-oss-120b)"]
    GROQ_RESP -->|"Model Output"| ROUTER

    ROUTER -->|"Final Formatted Response"| REQ_MODULE
```

---

## 4. Architectural Comparison: Version 2.1 vs Version 2.2

| Architectural Dimension | Version 2.1 (Agora Foundation) | Version 2.2 (Internal No-Agora) |
|---|---|---|
| **Audio Transport Layer** | Agora RTC Engine & WebRTC Relay (Cost-Intensive) | Native HTML5 Web Audio API (16kHz PCM mono) + FastAPI WebSocket Relay (Zero-Telephony Cost) |
| **Speech-to-Text (STT)** | Agora STT Gateway | Direct ElevenLabs Scribe Realtime STT (`scribe_v2_realtime`) with automatic silence commit |
| **Text-to-Speech (TTS)** | Neural TTS relayed via Agora | Direct ElevenLabs Flash v2.5 (`eleven_flash_v2_5`) with 21+ assigned voice profiles |
| **Conversational Latency** | ~2.5 - 3.5s due to Agora bridging hops | Sub-second turn decisions (~0.7 - 1.2s end-to-end) |
| **LLM Router Hierarchy** | Ad-hoc routing calls | Formalized `otheragent.py` token threshold router (`<= 7.5k` Groq LPU vs `> 7.5k` Gemini) |
| **Rate Limit Protection** | Single key per provider | Dynamic Multi-Key Token Bucket Pool rotating 1 to 20 API keys per provider |
| **Visual Architecture Tool** | Conceptual text prompts | Interactive System Design Canvas with Multimodal Vision Engine analyzing base64 PNGs |
| **Timer Integrity** | Client-initiated timer | Server-authoritative monotonic timer starting strictly on `question_started` playback |
| **Barge-In Handling** | Agora speech detector | Sub-150ms client-side Web Audio buffer flush on candidate voice onset |
| **Recruiter Scoring** | Text evaluation summaries | Validated multi-dimensional scorecards with verbatim transcript quote grounding |

---

## 5. Platform Interface Showcase

### Candidate Portal and Job Application
![Candidate Job Directory](screenshots/candidate-portal/01_candidate_jobs_list.png)
*Candidate browsing verified company job requisitions with dynamic interview parameters.*

![Candidate Profile and Resume Parsing](screenshots/candidate-portal/04_candidate_profile_resume.png)
*Structured candidate profile extracting experience, skills matrix, and project history.*

### Pre-Flight Device Lobby & Live Interview Room
![Pre-Flight Device Lobby](screenshots/interview-room/01_preflight_device_lobby.png)
*Hardware verification lobby testing camera feed, Web Audio microphone levels, and disclosing AI evaluation.*

![Live Lead Technical Interviewer Turn](screenshots/interview-room/04_lead_tech_interviewer_turn.png)
*Lead Technical Interviewer probing distributed systems and caching invalidation with real-time audio playback.*

![Interactive System Design Canvas](screenshots/interview-room/07_system_design_canvas_drawing.png)
*Interactive architectural canvas for sketching microservice topologies evaluated by the Multimodal Vision Engine.*

### Recruiter Leaderboard and Scorecard Breakdown
![Recruiter Ranked Leaderboard](screenshots/recruiter-dashboard/04_recruiter_leaderboard_ranked.png)
*Recruiter dashboard arranging all interviewed candidates in strictly descending order of overall match score.*

![Detailed AI Evaluation Scorecard](screenshots/recruiter-dashboard/05_detailed_ai_evaluation_scorecard.png)
*Multi-dimensional candidate scorecard detailing strengths, concerns, and hire recommendations with transcript quotes.*
