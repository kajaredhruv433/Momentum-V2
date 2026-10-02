# Agentic AI Architecture: Autonomous Multi-Agent Cognitive Engine

## Executive Overview: Why Momentum is Purely Agentic

Conventional interview platforms rely on static question scripts, hardcoded bots, or naive single-turn prompts. In contrast, **Momentum V2** is architected as an **Autonomous Multi-Agent System (MAS)** where interview panels, evaluation strategies, and conversation steering are computed dynamically at runtime.

The system does not follow a predefined question list. After every candidate response, the reasoning engine analyzes the complete transcript history, makes an autonomous decision on whether to continue the interview, determines what competency to probe next, formulates the exact spoken question, and selects the optimal interviewer persona to deliver it.

![Multi-Agent LLM Routing Hierarchy](../architecture/llm-routing-hierarchy.png)

---

## 1. On-Demand Dynamic Panel Generation (1 to 5 Members)

Interview panels and their respective roles are **not predefined or hardcoded**. When an interview session is initiated, our orchestrator prompts the reasoning engine (`panel.py`) with the job criteria and candidate profile to synthesize a bespoke committee of 1 to 5 interviewers.

```mermaid
flowchart TD
    JOB_REQ["Company & Job Criteria (Role, Skills, Seniority, Qualities)"] --> ORCHESTRATOR["Panel Formulator (panel.py)"]
    CAND_RESUME["Parsed Candidate Resume (Experience, Projects, Education)"] --> ORCHESTRATOR
    
    ORCHESTRATOR --> PROMPT["Compile Dynamic Panel Synthesis Prompt (PANEL_PROMPT)"]
    PROMPT --> LLM_GEN["otheragent.py (Groq LPU Reasoning)"]
    
    LLM_GEN --> JSON_OUT["Raw Structured JSON Schema Output"]
    JSON_OUT --> VALIDATION{"Validate & Normalise Panel Shape"}
    
    VALIDATION -->|Pass| INSTANTIATE["Instantiate 1 to 5 Specialized Panel Personas"]
    VALIDATION -->|Fail| RETRY["Compact Retry & Fallback Repair"]
    RETRY --> INSTANTIATE

    subgraph GeneratedPanel ["Dynamically Instantiated Panel (Stored in Session State)"]
        MEMBER_1["Panel Member 1: Name, Role, Seniority, Evaluation Focus, Assigned Voice ID"]
        MEMBER_2["Panel Member 2: Name, Role, Seniority, Evaluation Focus, Assigned Voice ID"]
        MEMBER_N["Panel Member N (Up to 5): Name, Role, Seniority, Evaluation Focus, Assigned Voice ID"]
    end

    INSTANTIATE --> GeneratedPanel
```

### Runtime Generation Parameters
For every generated panel member, the system dynamically assigns:
- **Name & Persona**: Human identity, communication style, and personality.
- **Role & Seniority**: Contextual job title tailored specifically to the target role (e.g., Senior Infrastructure Architect, Lead Frontend Engineer, Group Product Manager, Culture Fit Evaluator).
- **Interview Stage & Responsibility**: Specific phase ownership (e.g., Technical Deep Dive, System Design, Behavioral Fit).
- **Evaluation Focus**: Distinct non-overlapping list of competencies and technical domains to assess.
- **Behavioral Guidelines**: Specific rules on questioning style, depth, and tone.
- **Assigned Neural Voice**: Dedicated ElevenLabs voice profile from the catalog (`voice.py`).

---

## 2. Real-Time Turn-by-Turn Cognitive Decision Cycle

Every conversational turn in Momentum follows a deterministic yet highly adaptive 5-step multi-agent decision cycle:

```mermaid
flowchart TD
    RESPONSE["Candidate Answer Received (Audio / Text)"] --> HISTORY["Fetch Full Transcript History & Elapsed Monotonic Time"]
    
    subgraph Step1 ["Step 1: Continuation Decision (normal_stage.py)"]
        HISTORY --> DECISION{"otheragent Decision: CONTINUE, CONCLUDE, or DRAW?"}
        DECISION -->|CONCLUDE| CONCLUDE_FLOW["Trigger Immediate Conclusion & Closing Remarks Workflow"]
        DECISION -->|DRAW| DRAW_FLOW["Formulate System Design Canvas Prompt via otheragent"]
        DECISION -->|CONTINUE| PROCEED["Proceed to Competency Direction Selection"]
    end

    subgraph Step2 ["Step 2: Direction & Competency Selection"]
        PROCEED --> EVAL_PREV["Evaluate Previous Responses & Detect Vagueness / Contradictions"]
        EVAL_PREV --> CHECK_MATRIX["Compare Covered vs Uncovered Competencies (areas_to_cover)"]
        CHECK_MATRIX --> SELECT_DIR["Select Next Context Path (ROLE/AREA, PROJECT, SKILL)"]
    end

    subgraph Step3 ["Step 3: Direct Spoken Question Generation"]
        SELECT_DIR --> GEN_Q["Formulate Direct 2nd-Person Spoken Question via otheragent"]
        DRAW_FLOW --> GEN_DRAW_Q["Generate Spoken Architectural Design Instructions"]
    end

    subgraph Step4 ["Step 4: Dynamic Panel Member Selection"]
        GEN_Q --> SELECT_MEMBER["Evaluate Instantiated Panel Members vs Question & Topic"]
        GEN_DRAW_Q --> SELECT_MEMBER
        SELECT_MEMBER --> ASSIGN["Select Optimal Interviewer Persona (Who Should Speak)"]
    end

    subgraph Step5 ["Step 5: Voice Delivery & State Update"]
        ASSIGN --> TTS["Stream Audio via ElevenLabs Flash v2.5 with Persona Voice ID"]
        TTS --> DELIVER["Deliver High-Fidelity Audio to Candidate Browser"]
        DELIVER --> UPDATE_STATE["Append Question & Answer to Working State & Persist to MongoDB"]
    end
```

---

## 3. Deep Dive into the 5-Step Execution Cycle

### Step 1: Autonomous Continuation Decision (Continue vs Conclude vs Draw)
Before formulating any new question, the reasoning engine (`normal_stage.py`, `extended_stage.py`, `final_stage.py`) executes a continuation evaluation:
- **CONCLUDE Decision**: If the candidate's answers demonstrate that the candidate is completely unqualified, or if all evaluation milestones are satisfied, the engine triggers an early conclusion rather than wasting interview time.
- **DRAW Decision**: If an architectural design assessment is required, the engine diverts to the System Design Canvas workflow.
- **CONTINUE Decision**: If further assessment is warranted, the engine proceeds to generate the next question.

### Step 2: Evaluation of Previous Answers & Direction Selection
The engine evaluates the candidate's latest response in the context of the entire previous history:
- **Vagueness Detection**: If the candidate provided a generic answer without implementation mechanics, the engine steers the direction to probe that exact claim.
- **Contradiction Identification**: If the candidate makes claims conflicting with earlier statements, the engine flags the discrepancy.
- **Competency Gap Tracking**: Compares covered areas against required competencies to select the next topic area, resume project, or skill.

### Step 3: Direct Spoken Question Generation
`otheragent.py` directly formulates the spoken question in second-person conversational phrasing tailored to the active lifecycle stage (`Normal`, `Extended`, `Final`), ready for immediate speech synthesis.

### Step 4: Dynamic Speaker Selection (Who Will Ask)
Rather than cycling through interviewers in a fixed round-robin order, the system evaluates all dynamically instantiated panel members against the newly generated question:
- The panel member whose `evaluation_focus`, `role`, and `persona` best match the subject matter is selected to speak.
- If the question challenges business trade-offs, the Product Manager is selected. If it probes database indexing or concurrency, the Lead Technical Interviewer is selected.

### Step 5: Voice Synthesis & Delivery
The selected panel member's assigned ElevenLabs neural voice synthesizes the question and streams it to the candidate browser with sub-second latency.

---

## 4. Multi-Persona Cross-Examination Example

```text
[Turn 1: Lead Technical Architect asks]
"How would you design a caching layer to handle a 10x traffic spike on our payment service?"

[Candidate Response]
"I would introduce Redis clusters with a write-through caching strategy and set up LRU eviction with a 5-minute TTL."

[Step 1: Continuation Decision]
otheragent Decision: CONTINUE

[Step 2 & 3: Evaluation and Question Generation]
Observation: Technical caching strategy is sound, but a 5-minute TTL could cause users to see outdated payment balances.
Spoken Question: "That caching setup handles the load well technically, Alex. But looking at this from our customers' perspective: if a user makes a payment and sees an outdated balance because of that 5-minute delay, how would you design this so customers never experience inconsistent transaction states?"

[Step 4: Dynamic Speaker Selection]
Selected Speaker: Product Manager (Laura) based on customer impact and business priority focus.

[Step 5: Delivery]
Synthesized via ElevenLabs Laura voice profile (FGY2WhTYpPnrIDTdsKH5) and streamed to candidate.
```

---

## 5. Tool Integration & Multimodal Vision

```mermaid
sequenceDiagram
    participant LeadTech as Lead Technical Interviewer
    participant CanvasUI as Candidate Drawing Canvas
    participant VisionTool as Multimodal Vision Engine
    participant Memory as Session Context & State Store

    LeadTech->>CanvasUI: Emits DRAW Action & Spoken Architectural Challenge
    Note over CanvasUI: Candidate sketches microservices, Redis, queues, and PostgreSQL
    CanvasUI->>VisionTool: Submits Base64 PNG Topology Diagram
    VisionTool->>VisionTool: Identifies Nodes, Ingestion Streams, Caching & Single Points of Failure
    VisionTool->>Memory: Appends Structured Vision Findings & Design Suggestions
    Memory->>LeadTech: Updates Context with Diagram Insights
    LeadTech->>CanvasUI: Formulates Deep-Dive Question Probing Identified Bottleneck
```

---

## 6. Reflection and Evidence-Based Scorecard Synthesis

Upon reaching interview conclusion, the system enters the reflection stage:
- Dispatches the complete transcript, timing data, and canvas analyses to Gemini via `otheragent.py`.
- Generates a multi-dimensional scorecard (0-100 across technical, problem-solving, and communication categories).
- Grounds every listed strength, concern, and hire/reject reason in direct timestamped verbatim quotes from the interview transcript.
- Updates the candidate's application in MongoDB and ranks them on the recruiter leaderboard.

---

## 7. Multi-Persona Panel in Action

### Human Resources & Behavioral Turn
![HR Manager Interview Turn](screenshots/interview-room/03_hr_manager_turn.png)
*HR Specialist evaluating cross-functional teamwork, conflict resolution, and communication clarity.*

### Lead Technical Interviewer Turn
![Lead Technical Turn](screenshots/interview-room/04_lead_tech_interviewer_turn.png)
*Lead Technical Architect probing Redis caching strategies and database concurrency.*

### Product & Engineering Manager Cross-Examination
![Product Manager Turn](screenshots/interview-room/05_product_manager_challenge.png)
*Product Manager cross-examining the candidate on customer friction and data consistency trade-offs.*

### Live System Design Canvas Interaction
![System Design Drawing](screenshots/interview-room/07_system_design_canvas_drawing.png)
*Candidate sketching architectural topology evaluated in real-time by the Multimodal Vision Engine.*
