# 🏥 Hospital Customer Support — Technical Plan
### Agentic System — Implementation Blueprint

---

## 1. Tech Stack Overview

| Layer | Technology | Purpose |
|---|---|---|
| **Orchestration Framework** | LangGraph | Agent graph, state management, routing logic |
| **LLM Provider** | Google Gemini (via Vertex AI) | Core reasoning engine for all agents |
| **RAG Framework** | LangChain | Document loading, chunking, retrieval pipeline |
| **Vector Store** | ChromaDB (or Vertex AI Vector Search for cloud) | Storing and querying document embeddings |
| **Embedding Model** | Google `text-embedding-004` | Converting documents and queries into vectors |
| **SQL Database** | PostgreSQL | Patient records, doctor schedules, appointment slots |
| **ORM** | SQLAlchemy | Database access layer for all SQL operations |
| **Communication Platform** | Twilio | Voice calls, SMS, WhatsApp messaging |
| **Speech-to-Text** | Whisper (via Groq API) | Transcribing voice calls into text |
| **Text-to-Speech** | ElevenLabs | Converting agent responses to voice |
| **Document Parsing** | PyMuPDF (PDFs) + python-docx (Word) | Extracting text from knowledge base documents |
| **Audit Logging** | PostgreSQL (separate schema) + Python `logging` | Persisting all agent actions and events |
| **API Layer** | FastAPI | Exposing the system as an API for channel integrations |
| **Task Queue** | Celery + Redis | Async tasks (ambulance dispatch, reminders, notifications) |
| **Containerization** | Docker + Docker Compose | Local development and deployment packaging |

---

## 2. LLM Configuration

| Setting | Value |
|---|---|
| **Provider** | Google Gemini via Vertex AI |
| **Primary Model** | `gemini-2.5-flash` (fast responses, cost-efficient for most agents) |
| **Complex Reasoning Model** | `gemini-2.0-pro` (Orchestrator + Emergency agent for critical decisions) |
| **Embedding Model** | `text-embedding-004` |
| **Temperature** | `0.2` — low, for consistent and factual responses |
| **Max Output Tokens** | `1024` per agent turn |
| **Streaming** | Enabled — for real-time chat and voice responsiveness |

> **Reasoning:** `gemini-2.5-flash` is used for most agents to minimize latency and cost. The Orchestrator and Loyalty & Emergency Agent use `gemini-2.0-pro` because their decisions (routing, emergency dispatch) require higher accuracy and reasoning depth.

---

## 3. Agentic Framework — LangGraph

### Why LangGraph
LangGraph models the multi-agent system as a **stateful directed graph**. Each node in the graph is an agent. Edges represent routing decisions. The graph state is automatically passed between nodes, eliminating the need for a separate session memory store.

### Graph State Object
A single shared state object travels through the entire graph per conversation turn. It holds:

| State Field | Type | Description |
|---|---|---|
| `session_id` | string | Unique ID per conversation session |
| `channel` | enum | `chat`, `voice`, `sms`, `whatsapp` |
| `patient_id` | string / null | Resolved after identification |
| `patient_record` | object / null | Full patient data from SQL |
| `is_new_patient` | boolean | Whether a new record needs to be created |
| `is_high_risk` | boolean | Diabetes, cardiac history, etc. |
| `is_loyal` | boolean | Loyalty flag from history |
| `urgency_level` | enum | `normal`, `high`, `emergency` |
| `active_agent` | string | Which agent is currently handling |
| `conversation_history` | list | Full message history for context |
| `query_intent` | string | Classified intent from the orchestrator |
| `resolution_attempts` | int | Counter for escalation fallback logic |
| `audit_events` | list | Events to be flushed to audit log |

### Graph Routing Logic
```
START
  └──► Orchestrator Agent
          ├── intent = "appointment"         ──► Scheduling Agent
          ├── intent = "service / info"      ──► Services & Knowledge Agent
          ├── intent = "new / update record" ──► Patient Records Agent
          ├── intent = "loyalty / emergency" ──► Loyalty & Emergency Agent
          ├── resolution_attempts >= 3       ──► Human Escalation Node
          └── urgency = "emergency"          ──► Loyalty & Emergency Agent (priority path)
```

---

## 4. Agents — Technical Specifications

---

### 4.1 🧭 Orchestrator Agent

**Model:** `gemini-2.0-pro`
**Role in Graph:** Entry node — every conversation starts here

**System Prompt Responsibilities:**
- Greet the patient and request identification (phone number or National ID)
- Classify query intent into one of: `appointment`, `service_info`, `records`, `loyalty`, `emergency`, `unknown`
- Assess urgency based on keywords and patient medical history
- Route to the correct specialist agent

**Tools:**

| Tool | Description |
|---|---|
| `lookup_patient_by_phone` | Queries SQL for a patient record by phone number |
| `lookup_patient_by_national_id` | Queries SQL for a patient record by National ID |
| `classify_intent` | LLM-powered intent classifier (returns intent enum + confidence score) |
| `assess_urgency` | Checks message content + patient medical flags for urgency level |
| `route_to_agent` | LangGraph edge transition — moves state to the next node |
| `trigger_human_escalation` | Sends session to human agent queue after failed resolution attempts |

**Escalation Condition:** `resolution_attempts >= 3` → triggers `trigger_human_escalation`

---

### 4.2 📅 Scheduling Agent

**Model:** `gemini-2.5-flash`
**Role in Graph:** Specialist node — handles all appointment-related queries

**System Prompt Responsibilities:**
- Retrieve available doctors by specialty or name
- Show available time slots
- Book, reschedule, or cancel appointments
- Send confirmation and reminder notifications

**Tools:**

| Tool | Description |
|---|---|
| `get_doctors_by_specialty` | Queries SQL for doctors filtered by specialty |
| `get_available_slots` | Returns open appointment slots for a given doctor and date range |
| `book_appointment` | Inserts a new appointment record into SQL |
| `reschedule_appointment` | Updates an existing appointment slot in SQL |
| `cancel_appointment` | Soft-deletes an appointment record in SQL |
| `send_appointment_confirmation` | Triggers Twilio to send SMS/WhatsApp confirmation |
| `send_appointment_reminder` | Schedules a Celery task to send a reminder before the appointment |

**SQL Tables Used:** `doctors`, `specialties`, `availability_slots`, `appointments`

---

### 4.3 📚 Services & Knowledge Agent

**Model:** `gemini-2.5-flash`
**Role in Graph:** Specialist node — answers hospital knowledge base queries

**System Prompt Responsibilities:**
- Answer questions about hospital services (MRI, X-Ray, physiotherapy, etc.)
- Retrieve pricing, availability, preparation instructions
- Answer policy and FAQ questions

**Tools:**

| Tool | Description |
|---|---|
| `rag_search` | Semantic vector search over the hospital knowledge base |
| `rerank_results` | Reranks retrieved chunks by relevance before passing to LLM |
| `format_response` | Structures the LLM response for the specific channel (chat vs. voice) |

**No SQL access** — this agent operates entirely on the RAG knowledge base.

**Retrieval Strategy:** Hybrid search — semantic (vector) + keyword (BM25) combined, then reranked. See Section 6 for full RAG pipeline.

---

### 4.4 🗂️ Patient Records Agent

**Model:** `gemini-2.5-flash`
**Role in Graph:** Specialist node — manages all patient data operations

**System Prompt Responsibilities:**
- Create new patient records for first-time callers
- Update existing records (contact info, last visit, conditions)
- Flag high-risk medical conditions in the state object
- Return structured patient history to the orchestrator on request

**Tools:**

| Tool | Description |
|---|---|
| `create_patient_record` | Inserts a new patient row into the `patients` table |
| `update_patient_record` | Updates contact info, last visit date, or notes |
| `get_full_patient_history` | Returns all visits, conditions, and interactions for a patient |
| `flag_medical_condition` | Sets `is_high_risk = true` in graph state based on condition list |
| `validate_patient_data` | Validates input fields (phone format, ID format, required fields) |

**SQL Tables Used:** `patients`, `medical_conditions`, `visit_history`, `patient_contacts`

---

### 4.5 🎁🚨 Loyalty & Emergency Agent

**Model:** `gemini-2.0-pro`
**Role in Graph:** Specialist node — handles two critical flows

**System Prompt Responsibilities (Loyalty):**
- Calculate a loyalty score based on visit frequency and history length
- Select the appropriate reward tier and present it to the patient
- Log the reward offer

**System Prompt Responsibilities (Emergency):**
- Confirm the patient's current location
- Ask for explicit confirmation before dispatching ambulance
- Trigger dispatch and immediately escalate to human agent

**Tools:**

| Tool | Description |
|---|---|
| `calculate_loyalty_score` | Queries visit history and computes a loyalty tier (`bronze`, `silver`, `gold`) |
| `get_reward_for_tier` | Returns the appropriate reward offer based on loyalty tier |
| `send_reward_offer` | Sends the reward message via Twilio (SMS/WhatsApp) or chat |
| `confirm_emergency_with_patient` | Sends a structured confirmation message and waits for patient reply |
| `dispatch_ambulance` | Calls the ambulance dispatch API with patient location (async via Celery) |
| `trigger_human_escalation` | Immediately routes the session to a human agent queue |
| `log_emergency_event` | Writes a high-priority entry to the audit log |

**SQL Tables Used:** `patients`, `visit_history`, `loyalty_rewards`, `emergency_events`

---

## 5. SQL Database Design

### Design Approach
PostgreSQL, designed from scratch. Schema organized into logical groups: patient data, clinical data, scheduling, loyalty, and audit.

### Core Tables

**`patients`**
```
patient_id        UUID PRIMARY KEY
phone_number      VARCHAR UNIQUE
national_id       VARCHAR UNIQUE
full_name         VARCHAR
date_of_birth     DATE
gender            VARCHAR
address           TEXT
email             VARCHAR
created_at        TIMESTAMP
updated_at        TIMESTAMP
is_high_risk      BOOLEAN DEFAULT FALSE
loyalty_tier      ENUM('none', 'bronze', 'silver', 'gold')
```

**`medical_conditions`**
```
condition_id      UUID PRIMARY KEY
patient_id        UUID FK → patients
condition_name    VARCHAR
diagnosed_at      DATE
severity          ENUM('low', 'medium', 'high', 'critical')
notes             TEXT
```

**`visit_history`**
```
visit_id          UUID PRIMARY KEY
patient_id        UUID FK → patients
visit_date        TIMESTAMP
department        VARCHAR
doctor_id         UUID FK → doctors
notes             TEXT
outcome           TEXT
```

**`doctors`**
```
doctor_id         UUID PRIMARY KEY
full_name         VARCHAR
specialty_id      UUID FK → specialties
phone_number      VARCHAR
email             VARCHAR
is_active         BOOLEAN
```

**`specialties`**
```
specialty_id      UUID PRIMARY KEY
name              VARCHAR
description       TEXT
```

**`availability_slots`**
```
slot_id           UUID PRIMARY KEY
doctor_id         UUID FK → doctors
start_time        TIMESTAMP
end_time          TIMESTAMP
is_booked         BOOLEAN DEFAULT FALSE
```

**`appointments`**
```
appointment_id    UUID PRIMARY KEY
patient_id        UUID FK → patients
slot_id           UUID FK → availability_slots
booked_at         TIMESTAMP
status            ENUM('confirmed', 'cancelled', 'rescheduled', 'completed')
channel           VARCHAR
```

**`loyalty_rewards`**
```
reward_id         UUID PRIMARY KEY
patient_id        UUID FK → patients
tier              ENUM('bronze', 'silver', 'gold')
reward_type       VARCHAR
offered_at        TIMESTAMP
accepted          BOOLEAN
```

**`emergency_events`**
```
event_id          UUID PRIMARY KEY
patient_id        UUID FK → patients
triggered_at      TIMESTAMP
confirmed_by_patient  BOOLEAN
ambulance_dispatched  BOOLEAN
location          TEXT
resolved          BOOLEAN
human_agent_id    VARCHAR
```

**`audit_log`** *(separate schema)*
```
log_id            UUID PRIMARY KEY
session_id        VARCHAR
agent_name        VARCHAR
action            VARCHAR
entity_type       VARCHAR
entity_id         UUID
metadata          JSONB
timestamp         TIMESTAMP
channel           VARCHAR
```

---

## 6. RAG Pipeline — Knowledge Base

### Document Types Supported
- PDF files (hospital brochures, service guides, policy documents)
- Word (.docx) files (FAQs, internal procedures)

### Ingestion Pipeline

```
Raw Documents (PDF / DOCX)
        │
        ▼
Document Parser
  └── PyMuPDF for PDFs
  └── python-docx for Word files
        │
        ▼
Text Cleaner
  └── Remove headers/footers, page numbers, artifacts
        │
        ▼
Chunker
  └── Strategy: Recursive Character Text Splitter
  └── Chunk size: 512 tokens
  └── Overlap: 64 tokens
  └── Respect paragraph / section boundaries
        │
        ▼
Embedder
  └── Google text-embedding-004
  └── Batch embedding for efficiency
        │
        ▼
Vector Store (ChromaDB)
  └── Persisted to disk / cloud storage
  └── Metadata stored per chunk: source file, page number, section title
```

### Retrieval Strategy — Hybrid Search

At query time, two retrieval methods run in parallel and results are merged:

| Method | Description | Weight |
|---|---|---|
| **Semantic Search** | Vector similarity using cosine distance on embeddings | 70% |
| **Keyword Search (BM25)** | Exact and partial keyword matching | 30% |

Results are then passed through a **reranker** (`cross-encoder/ms-marco-MiniLM-L-6-v2`) to re-score and select the top-K most relevant chunks before being sent to the LLM.

| Parameter | Value |
|---|---|
| Top-K retrieval (before rerank) | 10 chunks |
| Top-K after rerank | 3 chunks |
| Similarity threshold | 0.75 (below this, agent says "I don't have that information") |

---

## 7. Communication Layer — Twilio Integration

### Channels & Integration Points

| Channel | Twilio Product | Flow |
|---|---|---|
| **SMS** | Twilio Programmable SMS | Inbound webhook → FastAPI → LangGraph → Twilio reply |
| **WhatsApp** | Twilio WhatsApp Business API | Same as SMS flow via WhatsApp sandbox / approved number |
| **Voice** | Twilio Programmable Voice | Call → STT (Whisper / Groq) → LangGraph → TTS (ElevenLabs) → Twilio audio stream |

### Voice Call Flow (Detailed)

```
Patient calls hospital number (Twilio)
        │
        ▼
Twilio streams audio → Whisper (Groq API)
        │
        ▼
Transcribed text → FastAPI webhook
        │
        ▼
LangGraph processes text through agent graph
        │
        ▼
Agent text response → ElevenLabs Text-to-Speech
        │
        ▼
Audio streamed back to patient via Twilio
```

### Notification Events Sent via Twilio

| Event | Channel | Trigger |
|---|---|---|
| Appointment confirmation | SMS + WhatsApp | After `book_appointment` |
| Appointment reminder | SMS + WhatsApp | 24h before appointment (Celery scheduled task) |
| Loyalty reward offer | SMS + WhatsApp | After `send_reward_offer` |
| Emergency confirmation request | SMS + WhatsApp + Voice | During emergency flow |

---

## 8. Async Task Queue — Celery + Redis

Some operations must run asynchronously (non-blocking) to keep response times minimal.

| Task | Trigger | Delay |
|---|---|---|
| `send_appointment_reminder` | After booking | Scheduled for 24h before appointment |
| `dispatch_ambulance` | After patient confirmation | Immediate async |
| `send_reward_notification` | After reward offer generated | Immediate async |
| `flush_audit_log_batch` | Every N events or every 60s | Periodic |

**Redis** acts as the message broker between FastAPI and Celery workers.

---

## 9. API Layer — FastAPI

All external systems (Twilio, future web chat frontend, hospital internal tools) interact with the agentic system through a FastAPI REST API.

### Key Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/chat/message` | Receives a chat message, runs through LangGraph, returns response |
| `POST` | `/voice/webhook` | Twilio voice webhook — receives transcribed text, returns TTS response |
| `POST` | `/sms/webhook` | Twilio SMS/WhatsApp inbound webhook |
| `GET` | `/patient/{patient_id}` | Retrieve patient record (internal use) |
| `GET` | `/audit/session/{session_id}` | Retrieve audit log for a session (internal use) |
| `GET` | `/health` | System health check |

---

## 10. Audit Logging Strategy

Every agent action is logged. The audit log is **append-only** and stored in a dedicated PostgreSQL schema.

### What Gets Logged

| Event Type | Example |
|---|---|
| Patient identified | `patient_id=XXX identified via phone` |
| Intent classified | `intent=appointment, confidence=0.94` |
| Agent routed | `routed to SchedulingAgent` |
| Record created | `new patient record created: patient_id=XXX` |
| Record updated | `updated last_visit for patient_id=XXX` |
| Appointment booked | `slot_id=YYY booked for patient_id=XXX` |
| Reward offered | `gold tier reward offered to patient_id=XXX` |
| Emergency triggered | `emergency confirmed, ambulance dispatched` |
| Human escalated | `escalated to human after 3 failed attempts` |
| RAG query | `query="MRI cost", top_chunk_score=0.88` |

### Audit Log Properties
- **Append-only** — records are never updated or deleted
- **JSONB metadata field** — flexible storage for agent-specific details
- **Indexed** on `session_id`, `patient_id`, `timestamp` for fast querying
- **Retained** for a minimum of 7 years (hospital compliance requirement)

---

## 11. Security Considerations

| Concern | Approach |
|---|---|
| **Patient data encryption** | All PII fields encrypted at rest (PostgreSQL `pgcrypto`) |
| **API authentication** | FastAPI endpoints secured with API keys + JWT tokens |
| **Twilio webhook verification** | Validate Twilio signature on all inbound webhooks |
| **LLM prompt injection** | Input sanitization before passing to LLM; system prompt hardening |
| **Ambulance dispatch authorization** | Double-confirmation pattern — patient confirms + action logged before dispatch |
| **Audit log integrity** | Audit schema is write-only for application; read access restricted to compliance role |
| **Environment secrets** | All API keys (Gemini, Groq, ElevenLabs, Twilio, DB) stored in environment variables / secrets manager |

---

## 12. Key Technical Decisions — Summary

| Decision | Choice | Rationale |
|---|---|---|
| Orchestration | LangGraph | Native stateful graph; eliminates need for external session memory |
| LLM | Google Gemini via Vertex AI | Consistent ecosystem with Google embeddings; Whisper (Groq) for STT and ElevenLabs for TTS |
| Two Gemini models | Flash (speed) + Pro (reasoning) | Optimizes cost vs. accuracy per agent criticality |
| Vector Store | ChromaDB | Lightweight, easy to self-host; can migrate to Vertex AI Vector Search for cloud |
| Retrieval | Hybrid (semantic + BM25 + rerank) | Best accuracy for structured hospital documents |
| Database | PostgreSQL | Reliable, ACID-compliant, supports JSONB for flexible audit metadata |
| Async | Celery + Redis | Decouples time-sensitive tasks (reminders, dispatch) from response path |
| Communication | Twilio | Single platform for voice, SMS, and WhatsApp |
| API | FastAPI | High performance, async-native, clean OpenAPI docs |

---

*This document covers the full technical plan for implementation. Next step: folder structure, module breakdown, and implementation sprints.*
