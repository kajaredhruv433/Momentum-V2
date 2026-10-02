# Momentum V2 System Design and Engineering Principles

## System Design Objectives

1. **Ultra-Low Decision Latency**: Sub-second conversational turnaround between candidate speech completion and panel response generation.
2. **High Availability and Fault Tolerance**: Automated mitigation of third-party API rate limits and connection drops.
3. **Evaluation Integrity**: Deterministic, server-authoritative stage transitions and tamper-resistant countdown timers.
4. **Data Privacy**: Complete isolation of candidate voice, transcripts, and credentials.

---

## 1. Latency Breakdown and Optimization Pipeline

```mermaid
flowchart TD
    A["1. Candidate Stops Speaking (Silence Threshold)"] -->|0 - 150ms| B["2. ElevenLabs Scribe STT Commits Final Utterance"]
    B -->|50ms| C["3. WebSocket Packet Ingestion by FastAPI Backend"]
    C -->|400 - 800ms| D["4. Groq LPU Token-Routed Reasoning & Speaker Selection"]
    D -->|200 - 400ms| E["5. ElevenLabs Flash v2.5 Neural TTS Audio Chunk Streaming"]
    E -->|50ms| F["6. Browser AudioWorklet Buffer Ingestion & Playback"]

    subgraph TurnAround ["Total End-to-End Decision Turnaround"]
        A --- F
    end
```

### End-to-End Latency Profile
- **Standard Cloud Environment**: ~1.2 to 1.8 seconds total turnaround (accounting for multi-key quota verifications and network hops).
- **Dedicated Enterprise Environment**: ~0.7 to 1.0 second total turnaround with direct persistent connections.

---

## 2. Resilient Multi-Key Token Bucket Architecture

To prevent API rate limit failures (HTTP 429) during peak concurrency without requiring prohibitive single-key enterprise tiers, Momentum implements a custom token-bucket key manager (`groqrouter.py` and `geminirouter.py`):

```mermaid
flowchart TD
    REQ["Incoming Reasoning Request (Prompt Text)"] --> TOKEN_EST["Estimate Tokens Locally: len(text) / 4"]
    TOKEN_EST --> KEY_MANAGER["Multi-Key Token Bucket Manager"]
    
    subgraph PoolState ["Dynamic Key Pool (Keys 1..20 in .env)"]
        KEY_1["Key 1: Window Tokens, Requests, Active Count"]
        KEY_2["Key 2: Window Tokens, Requests, Active Count"]
        KEY_N["Key N: Window Tokens, Requests, Active Count"]
    end
    
    KEY_MANAGER <--> PoolState
    KEY_MANAGER --> SELECT{"Find First Key with Available Quota in Current 60s Window"}
    
    SELECT -->|Key Available| DISPATCH["Dispatch Async API Request via Selected Key"]
    SELECT -->|All In-Window Keys Busy| WAIT_KEY["Rotate to Key with Earliest Quota Reset"]
    WAIT_KEY --> DISPATCH
    
    DISPATCH --> HTTP_STATUS{"Evaluate Upstream HTTP Status"}
    HTTP_STATUS -->|200 OK| DEDUCT["Deduct Consumed Tokens & Update Usage Counters"]
    HTTP_STATUS -->|429 Rate Limit| COOLDOWN["Mark Key in Cooldown & Fallback Immediately to Next Key"]
    COOLDOWN --> SELECT
```

---

## 3. Server-Authoritative Timer & Monotonic Clock Synchronization

```mermaid
sequenceDiagram
    participant Browser as Candidate Browser Client
    participant API as FastAPI Gateway
    participant State as Authoritative State Machine
    participant WS as WebSocket Event Hub (/events)

    Browser->>API: POST /api/interviews (Initialize Session)
    API->>State: Creates Session State (Timer Status = IDLE)
    State-->>Browser: Returns Initialized State
    Note over Browser: Candidate verifies camera and microphone in Lobby

    Browser->>API: Launches Interview Room
    API->>Browser: Delivers Opening Greeting Audio Stream
    Note over Browser: Audio starts playing in candidate speaker
    Browser->>WS: Emits 'question_started' Event

    WS->>State: Sets Monotonic Start Timestamp (T0 = Now())
    State->>State: Marks timer_started = True

    loop Every 1 Second
        State->>WS: Broadcasts 'timer_tick' (elapsed_seconds, remaining_seconds, active_stage)
        WS->>Browser: Updates Client-Side Render Target
    end

    Note over Browser: If candidate tampers with local clock or pauses JavaScript:
    Browser->>API: Submits Late Answer
    API->>State: Validates Against Server Monotonic Clock
    State-->>Browser: Enforces True Server State (Rejects Tampering)
```

### Tamper Resistance Guarantees
- The client-side UI acts purely as a passive render target for timer events.
- If a candidate manipulates system clock settings or pauses JavaScript execution, the backend rejects late responses and enforces stage boundaries based strictly on the server's monotonic clock.
- Page reloads reconnect to `/api/interviews/{interview_id}/events` and resume displaying the exact elapsed time recorded on the server.

---

## 4. State Persistence and Fault Tolerance

The interview state machine (`state.py`) stores full runtime context in memory with write-through persistence to MongoDB:

```mermaid
flowchart LR
    subgraph MemoryState ["In-Memory State (state.py)"]
        ACTIVE_SESSION["Interview Session Context"]
        ACTIVE_PANEL["Instantiated Panel Details"]
        ACTIVE_TURNS["Question-Answer History"]
        ACTIVE_TIMER["Authoritative Monotonic Clock"]
    end

    subgraph MongoPersistence ["Persistent Datastore (MongoDB Atlas)"]
        DOC_JOB["Job Requisition Document"]
        DOC_APP["Candidate Application Record"]
        DOC_TRANSCRIPT["Full Transcript & Timeline"]
        DOC_SCORECARD["Validated Match Scorecard"]
    end

    ACTIVE_SESSION <-->|Write-Through Sync & Recovery| DOC_APP
    ACTIVE_TURNS -->|Continuous Append| DOC_TRANSCRIPT
    ACTIVE_TIMER -->|Checkpoint Elapsed Time| DOC_APP
```

- **Fault Tolerance**: If the backend service restarts during an active interview, the session state is reconstituted from MongoDB without loss of historical turns or elapsed monotonic time.
