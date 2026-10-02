# Momentum V2 API Specification

## Overview

The Momentum V2 backend exposes high-performance asynchronous REST endpoints and real-time bidirectional WebSocket protocols powered by FastAPI. This document outlines the API contracts, request payloads, response schemas, and WebSocket events.

---

## Base URL Configuration

| Environment | Base REST URL | WebSocket URL |
|---|---|---|
| **Local Development** | `http://127.0.0.1:8000` | `ws://127.0.0.1:8000` |
| **Production** | `https://api.momentum.domain` | `wss://api.momentum.domain` |

---

## Authentication & Headers

Requests interacting with protected candidate or recruiter resources include standard Bearer authorization headers:

```http
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json
```

All AI provider API keys (Groq, Google AI Studio, ElevenLabs) are maintained strictly server-side and are never required from client headers.

---

## REST Endpoints

### 1. Create and Initialize Interview Session

Initializes a new interview session by dynamically instantiating the multi-member interviewer panel, ingesting candidate resumes, and formulating the opening question.

- **Method**: `POST`
- **Endpoint**: `/api/interviews`
- **Request Body**:

```json
{
  "company_details": {
    "company_name": "Nexus Dynamics Inc.",
    "role": "Senior Full-Stack Product Engineer",
    "required_skills": ["Python", "FastAPI", "Next.js", "Redis", "Distributed Systems"],
    "required_experience": "4+ years of professional full-stack development experience",
    "required_qualification": "Bachelor's or Master's degree in Computer Science or related field",
    "salary": "$140,000 - $175,000",
    "role_description": "Architect and scale real-time collaborative enterprise platforms with FastAPI microservices and Next.js.",
    "employee_qualities": ["System design depth", "Customer empathy", "Clear technical communication"]
  },
  "resume_text": "Alex Chen\nSenior Software Engineer\n5 years experience building distributed Python/FastAPI microservices...",
  "base_time": 1800,
  "extended_time": 2700,
  "candidate_id": "cand_984729104",
  "candidate_name": "Alex Chen",
  "candidate_email": "alex.chen@example.com",
  "job_id": "job_nexus_fspe_01",
  "application_id": "app_alexchen_001"
}
```

- **Response (`201 Created`)**:

```json
{
  "interview_id": "intv_2026_09a8f4c21",
  "stage": "START_STAGE",
  "status": "initialized",
  "current_stage": "START_STAGE",
  "interview_phase": "Opening",
  "panel": {
    "member_hr_01": {
      "name": "Bella",
      "role": "Behavioral and Culture Manager",
      "voice_id": "hpp4J3VqNfWAUOO0d1Us"
    },
    "member_tech_01": {
      "name": "Charlie",
      "role": "Principal Systems Architect",
      "voice_id": "IKne3meq5aSn9XLyUdCD"
    },
    "member_product_01": {
      "name": "Laura",
      "role": "Group Product Manager",
      "voice_id": "FGY2WhTYpPnrIDTdsKH5"
    }
  },
  "selected_panel_member": "Bella",
  "current_question": "Welcome Alex! We are excited to speak with you today. To kick things off, could you share a brief overview of your background?",
  "timer_started": false,
  "elapsed_time": 0,
  "base_time": 1800,
  "extended_time": 2700
}
```

---

### 2. Fetch Interview Public State

Retrieves the real-time snapshot of the interview session including active stage, selected speaker, current question, elapsed timer, and transcript history.

- **Method**: `GET`
- **Endpoint**: `/api/interviews/{interview_id}`
- **Response (`200 OK`)**:

```json
{
  "interview_id": "intv_2026_09a8f4c21",
  "stage": "NORMAL_STAGE",
  "status": "in_progress",
  "current_stage": "NORMAL_STAGE",
  "interview_phase": "Technical Deep Dive",
  "selected_panel_member": "Charlie",
  "panel_member_details": {
    "name": "Charlie",
    "role": "Principal Systems Architect",
    "voice_id": "IKne3meq5aSn9XLyUdCD"
  },
  "current_question": "How do you handle cache invalidation and stampede prevention when high-traffic keys expire?",
  "timer_started": true,
  "elapsed_time": 420,
  "base_time": 1800,
  "extended_time": 2700,
  "report_ready": false
}
```

---

### 3. Submit Candidate Answer

Submits the candidate's answer (transcribed via ElevenLabs STT or typed), triggers multi-agent cognitive evaluation, decides turn-taking, and returns the next question.

- **Method**: `POST`
- **Endpoint**: `/api/interviews/{interview_id}/answer`
- **Request Body**:

```json
{
  "answer": "To prevent cache stampedes, we implemented probabilistic early expiration with distributed mutex locks on key refresh.",
  "input_method": "voice",
  "question_id": "q_turn_02"
}
```

- **Response (`200 OK`)**:

```json
{
  "status": "processed",
  "decision": "CONTINUE",
  "stage": "NORMAL_STAGE",
  "next_speaker": "Laura",
  "next_speaker_role": "Group Product Manager",
  "next_question": "That technical approach is sound, Alex. However, in our payment checkout flow, how would you adjust your caching strategy to safeguard customer balance accuracy?",
  "question_pattern": "Challenging customer payment accuracy vs stale cache trade-offs",
  "action_type": "question"
}
```

---

### 4. Submit System Design Canvas Diagram

Submits base64 PNG drawings from the interactive architecture canvas for multimodal LLM vision evaluation.

- **Method**: `POST`
- **Endpoint**: `/api/interviews/{interview_id}/canvas`
- **Request Body**:

```json
{
  "question_id": "q_turn_05_canvas",
  "image_data": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA..."
}
```

- **Response (`200 OK`)**:

```json
{
  "status": "analyzed",
  "components": [
    "Client WebSocket Pool",
    "Reverse Proxy Load Balancer",
    "FastAPI Gateway Workers",
    "Redis Pub/Sub Layer",
    "PostgreSQL Cluster"
  ],
  "relationships": [
    "Bidirectional WebSocket transport from client pool to FastAPI",
    "Asynchronous event publishing to Redis Stream"
  ],
  "design_notes": "Clean separation of stateful connection management and stateless processing workers.",
  "suggestions": [
    "Introduce a Dead Letter Queue (DLQ) for malformed incoming payloads"
  ]
}
```

---

### 5. Stream ElevenLabs Neural TTS Audio

Fetches synthesized neural audio for the active question or closing statement.

- **Method**: `GET`
- **Endpoint**: `/api/interviews/{interview_id}/tts`
- **Query Parameters**:
  - `voice_id` (optional): Override specific voice profile.
  - `text` (optional): Specific text snippet to synthesize.
- **Response**: `audio/mpeg` (Binary stream with HTTP chunked transfer encoding)

---

### 6. Retrieve Final Evaluation Report

Retrieves the structured scorecard, numeric evaluations, strengths, concerns, and hiring recommendation.

- **Method**: `GET`
- **Endpoint**: `/api/interviews/{interview_id}/report`
- **Response (`200 OK`)**:

```json
{
  "overall_match_percentage": 94,
  "job_role_match": 95,
  "skills_match": 96,
  "experience_match": 92,
  "qualification_match": 95,
  "technical_score": 96,
  "communication_score": 92,
  "problem_solving_score": 94,
  "strengths": [
    "Deep practical command of distributed systems, FastAPI concurrency, and cache synchronization",
    "Demonstrated balance between technical scalability and user-facing transactional correctness"
  ],
  "concerns": [
    "Minor omission of Dead Letter Queue (DLQ) topology in the initial canvas draft"
  ],
  "hire_reasons": [
    "Exceptional technical alignment with core stack requirements",
    "Strong architectural reasoning under follow-up questioning"
  ],
  "reject_reasons": [],
  "final_recommendation": "Strong Hire"
}
```

---

## WebSocket Protocols

### 1. Real-Time Interview Events & Synchronization

- **WebSocket Route**: `ws://127.0.0.1:8000/api/interviews/{interview_id}/events`

#### Outbound Client-to-Server Events

##### `question_started`
Notifies server that the browser has commenced playing the question audio, initiating the authoritative interview timer.

```json
{
  "type": "question_started",
  "turn_id": 1,
  "timestamp": 1759281200
}
```

##### `speech_finished`
Notifies server that closing remarks playback has concluded.

```json
{
  "type": "speech_finished",
  "timestamp": 1759283140
}
```

#### Inbound Server-to-Client Events

##### `timer_tick`
Server-authoritative timer countdown broadcasted at 1-second intervals.

```json
{
  "type": "timer_tick",
  "elapsed_seconds": 421,
  "remaining_base_seconds": 1379,
  "current_stage": "NORMAL_STAGE"
}
```

##### `stage_transition`
Broadcasted when the session moves between lifecycle stages.

```json
{
  "type": "stage_transition",
  "from_stage": "NORMAL_STAGE",
  "to_stage": "EXTENDED_STAGE",
  "elapsed_seconds": 1800
}
```

---

### 2. Live Audio Streaming & Speech-to-Text (STT) Relay

- **WebSocket Route**: `ws://127.0.0.1:8000/api/interviews/{interview_id}/audio-stream`

#### Client to Server
Streams 16kHz PCM mono audio chunks captured via Web Audio API.

```json
{
  "audio_chunk": "<BASE64_PCM_DATA>"
}
```

#### Server to Client

##### `partial_transcript`
Real-time interim transcription displayed on the candidate's screen.

```json
{
  "type": "partial_transcript",
  "text": "We implemented probabilistic early expiration"
}
```

##### `committed_transcript`
Finalized utterance transcript triggering silence countdown and answer submission.

```json
{
  "type": "committed_transcript",
  "text": "We implemented probabilistic early expiration with distributed mutex locks on key refresh."
}
```

---

## HTTP Status Codes Reference

| Code | Status | Description |
|---|---|---|
| `200` | OK | Request succeeded, response returned as specified. |
| `201` | Created | Interview session initialized and created. |
| `400` | Bad Request | Validation failure or malformed payload. |
| `404` | Not Found | Specified `interview_id` does not exist in state store. |
| `429` | Too Many Requests | Rate limit protection triggered (managed internally via key rotation). |
| `500` | Internal Server Error | Unhandled backend exception or upstream service failure. |
