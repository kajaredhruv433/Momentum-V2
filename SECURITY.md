# Security Policy

## Overview

Momentum V2 is engineered with security, candidate privacy, and data isolation at its core. This document outlines the security architecture, credential management principles, and the process for reporting potential vulnerabilities.

---

## Security Architecture Principles

### 1. Zero Client-Side Credential Exposure
- All third-party AI keys (Groq LPU keys, Google AI Studio Gemini keys, ElevenLabs API keys, and Qdrant Cloud credentials) are retained exclusively within backend environments.
- Candidate browser sessions interact strictly with the backend via authenticated REST endpoints and WebSocket relays.
- No third-party API tokens are ever delivered to or stored within the client-side JavaScript bundle.

### 2. Audio & Data Privacy
- Candidate audio data captured during interviews is processed strictly in-memory or streamed directly through secure WebSockets to the speech transcription pipeline.
- Audio chunks are not permanently logged or exposed via unauthenticated endpoints.
- Transcripts and evaluation reports are scoped strictly to the candidate's authenticated interview session ID.

### 3. Server-Side Timer & State Integrity
- Interview clocks, duration limits, and stage transitions are controlled exclusively by the backend state machine.
- Client-side event emissions (such as audio playback commencement) undergo server-side validation to prevent tampering or race conditions.

### 4. Resilient Multi-Key Token Bucket Rotation
- API requests dispatched to external LLM providers utilize dynamic key rotation and rate-limit tracking across multiple credential sets.
- This design isolates individual key exhaustion without risking service disruption or exposing credential metadata in network payloads.

---

## Intellectual Property & Codebase Access Policy

To protect project innovations, proprietary reasoning algorithms, and multi-agent coordination architectures from unauthorized copying or scraping, the production implementation repositories are hosted privately.

- **Public Repository Scope**: Contains architectural specifications, system design whitepapers, API contracts, workflow diagrams, sample schemas, and video demonstrations.
- **Recruiter & Evaluator Access**: Engineering hiring teams, recruiters, and academic evaluators can request full access to the underlying backend (FastAPI) and frontend (Next.js) repositories by contacting the author directly.

---

## Reporting Security Issues

If you identify a security vulnerability or privacy concern within this project or its specifications, please report it responsibly:

1. **Email**: Contact the repository maintainer directly at [kajaredhruv3@gmail.com](mailto:kajaredhruv3@gmail.com) or via GitHub ([@kajaredhruv433](https://github.com/kajaredhruv433)).
2. **Details to Include**:
   - Detailed description of the vulnerability.
   - Steps to reproduce or proof-of-concept payload.
   - Potential impact on system confidentiality, integrity, or availability.
3. **Response Commitment**: All security disclosures will be acknowledged within 48 hours, followed by a remediation plan and validation.

Please do not open public GitHub issues for security vulnerabilities.
