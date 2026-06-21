# Patient Assistant Plan

## 1. Problem Statement

Patients currently experience long wait times when contacting support, especially by phone, because they must wait until a human agent becomes available. Many patients do not find a human agent available at all, which creates missed contacts and weakens the reliability of the support experience.

The current process also creates inconsistent call handling. Some calls take longer than needed because agents must collect context manually, while other calls may be too short to capture the right patient need. Human agents may not have immediate access to patient history, or they may spend significant time fetching it from the database before they can respond effectively.

The business needs an agentic AI Patient Assistant for phone and chat that reduces wait time, improves availability, prepares human agents with relevant context, and supports tiered routing based on urgency, agent availability, intent, and loyalty/status priority.

The target outcome is a multi-agent AI orchestrated triage and handoff system where specialized AI agents collect patient context, apply routing rules, retrieve relevant CRM/EHR data, use a hybrid knowledge base that is not RAG alone, and hand off prepared cases to human agents.

## 2. User Stories

1. As a patient, I want to contact support by phone or chat and quickly have my request understood so that I do not wait unnecessarily for help.
2. As a human agent, I want to receive a summarized handoff with patient identity, intent, priority, and history so that I can resolve the case faster.
3. As an admin, I want to configure routing rules, loyalty priority, knowledge sources, and monitor wait/availability metrics so that the support operation stays efficient.

## 3. Product Requirements Document (PRD)

### Product Goal
Reduce patient wait time and improve agent availability through an omnichannel agentic AI assistant that coordinates specialized AI agents for orchestrated triage, patient-context retrieval, routing, and high-quality human handoff preparation.

### Target Users
- Patients using phone or chat support.
- Human support agents who receive patient handoffs.
- Admin and operations users who manage support rules, knowledge sources, and performance.

### Core Capabilities
- Accept patient interactions from phone (FreeSWITCH / Asterisk / WebRTC) or chat (WebSockets/Socket.io).
- Orchestrate specialized AI agents and deterministic tools through a central Orchestrator.
- Use a **Patient Identification Tool** to match incoming credentials (phone, email) to CRM identity or mark as a new customer.
- Perform unified triage using a **Triage AI Agent** to determine urgency, intent, and patient emotional state.
- Use a **Dynamic Knowledge Agent** that actively pursues the user's goal across multiple data sources (curation, API, rules) without depending solely on RAG, with dynamic source pivoting.
- Implement deterministic tools for routing decisions and agent availability checks.
- Use a **Handoff AI Agent** to synthesize and package interaction context into a structured CRM case summary.
- Manage shared state across agents using a robust state management framework (e.g., LangGraph or AutoGen).
- Provide operational monitoring for wait time and availability.

### Success Metrics
- Reduced average patient wait time.
- Reduced unanswered calls and chats.
- Improved completeness of handoff information.
- Faster agent handling time after handoff.

### Constraints
- The operational human agent pool must include a minimum of 4 agents.
- The agentic AI design must include at least 4 specialized AI agent roles: **Orchestrator**, **Triage AI Agent**, **Dynamic Knowledge Agent**, and **Handoff AI Agent**.
- Patient data must be treated as protected health information under a HIPAA-like privacy and security posture.
- The knowledge base must not rely on RAG alone.
- Version 1 channels are phone and chat.
- Version 1 automation covers orchestrated triage, dynamic knowledge retrieval, and handoff preparation (not full self-service).

## 4. Functional Requirements Document (FRD)

### System Components & Tech Stack
- **Backend Framework:** FastAPI (Python) for API endpoints and WebSocket handling.
- **AI/Agent Framework:** LangGraph for maintaining shared interaction state and orchestration.
- **Database:** Sqllite for CRM/EHR data, case saving, and user configurations.
- **Frontend Stack:** HTMX for both the Agent Dashboard and Admin Dashboard.
- **Telephony Integration:** <open source/ free tool> (Voice & SMS/WhatsApp).

### Agentic AI Requirements (Inputs, Outputs, and Tools)
1. **Agent Coordinator (Orchestrator)**
   - *Role*: Supervisor that maintains shared interaction state, directs traffic, and calls upon sub-agents based on the current context.
   - *Tools*: `Patient Identification Tool`, `Sub-agent delegation routines`.
2. **Triage AI Agent**
   - *Role*: Evaluates the patient transcript to extract intent and determine clinical or administrative urgency.
   - *Tools*: NLP sequence classification, Urgency/Priority deterministic rules engine.
3. **Dynamic Knowledge Agent**
   - *Role*: Goal-seeking retrieval agent that pursues information across curated documents and APIs, pivoting when a chosen source lacks evidence.
   - *Tools*: `Curated_Docs_Search`, `CRM_API_Lookup`, `EHR_API_Lookup`.
4. **Handoff AI Agent**
   - *Role*: Synthesizes collected evidence, patient context, and transcribed conversation into a concise, actionable CRM Case format.
   - *Tools*: `Summary_Formatter`, `CRM_Case_Creator`.

### Intake Requirements (Identification & Routing)
- The system attempts deterministic patient identification (`Patient Identification Tool`) matching phone/platform ID. Mark as `new` if no match.
- The **Routing Engine** (deterministic logic) uses outputs from the Triage Agent to assign cases to human agents based on: urgency, agent availability (minimum pool of 4), intent, and loyalty/status tier.

### Security & Admin Requirements
- Role-Based Access Control (RBAC) via JWT authentication.
- Audit logs captured for every CRM/EHR lookup.
- Admin dashboard to modify routing rules/tiers and view KPIs (wait time, answered contacts).

## 5. User Workflow

1. A patient calls the support number or opens the chat widget.
2. The Channel Gateway (Telephony Gateway/WebSockets) streams input to the **Agent Coordinator (Orchestrator)**.
3. Orchestrator invokes `Patient Identification Tool` using the phone number/ID.
4. Orchestrator passes context to the **Triage AI Agent** to determine intent and severity.
5. Orchestrator delegates to the **Dynamic Knowledge Agent** which searches EHR and CRM based on triage intent, executing API calls to gather history.
6. The deterministic **Routing Rules Engine** calculates assignment priorities and checks availability among the 4+ human agents.
7. Orchestrator delegates to the **Handoff AI Agent** to synthesize all data into a standardized structured handoff JSON.
8. The final CRM ticket is created and pushed to the **Agent Dashboard**.
9. The assigned human agent accepts the prepared case and continues communication smoothly.

## 6. Project Structure

```text
patient-assistant/
├── docker-compose.yml
├── .env.example
├── README.md
├── backend/
│   ├── templates/ (Jinja2 HTML files for HTMX)
│   ├── static/ (CSS, JS, Assets)
│   ├── requirements.txt
│   ├── main.py (FastAPI app entrypoint)
│   ├── app/
│   │   ├── api/
│   │   │   ├── routes.py
│   │   │   └── websockets.py
│   │   ├── agents/
│   │   │   ├── orchestrator.py
│   │   │   ├── triage_agent.py
│   │   │   ├── knowledge_agent.py
│   │   │   └── handoff_agent.py
│   │   ├── core/
│   │   │   ├── config.py (Settings & Env parsing)
│   │   │   └── security.py (JWT/RBAC)
│   │   ├── integrations/
│   │   │   ├── crm_client.py
│   │   │   ├── ehr_client.py
│   │   │   └── telephony_client.py
│   │   └── services/
│   │       ├── db.py (SQLAlchemy Setup)
│   │       ├── routing_engine.py
│   │       └── patient_identification.py
│   └── tests/
│       └── test_agents.py
└── docs/
```

## 7. Project Architecture

The system operates around a central API that manages external channels and internal AI orchestration via LangGraph (or similar state-machine based agent framework).

- **Gateway Layer:** Exposes `/webhook/telephony` and `/ws/chat/{client_id}`.
- **Orchestration Layer:** Maintains state (Patient Info, Transcript, Triage Result, Gathered Documents). Transitions iteratively between specialized AI Agents.
- **Hybrid Knowledge Base:** Combines FAISS/Chroma for vector-searchable curated documents, combined with strict REST API calls to the EHR mock database for real-patient history. Not relying purely on RAG semantic search.
- **Data Layer:** SQLite storing `Users`, `Interactions`, `AuditLogs`, `AgentProfiles`.

## 8. Mermaid Architecture Diagram

```mermaid
flowchart TD
    %% Patient Entry
    Patient[Patient] --> Phone[Phone Channel / Telephony Gateway]
    Patient --> Chat[Chat Channel / WebSockets]
    Phone --> Gateway[Channel Gateway API]
    Chat --> Gateway

    %% Internal Auth Boundary
    AdminDashboard[Admin Dashboard (HTMX/Jinja2)] --> IAM[JWT Auth / RBAC Middleware]
    AgentDashboard[Agent Dashboard (HTMX/Jinja2)] --> IAM
    IAM --> API[FastAPI Backend]

    Gateway --> API

    %% Orchestrator & Agents (Shared State)
    API --> Coordinator[Orchestrator Agent]

    Coordinator --> |1. Call| PatientIDTool[Patient ID Tool]
    Coordinator --> |2. Delegate| Triage[Triage AI Agent]
    Coordinator <--> |3. Delegate| DynamicKnowledge[Dynamic Knowledge Agent]
    Coordinator <--> |4. Delegate| HandoffAgent[Handoff AI Agent]
    
    Coordinator --> |5. Call| RoutingTool[Deterministic Routing Engine]

    %% Tools Layer
    DynamicKnowledge -.-> ToolLayer[Provider / Tool Layer]
    Triage -.-> ToolLayer

    ToolLayer --> Docs[Curated Docs & FAISS/Vector DB]
    ToolLayer --> Rules[Deterministic Rules Config]
    ToolLayer --> CRM[CRM DB]
    ToolLayer --> EHR[EHR DB]

    %% Handoff & Routing execution
    HandoffAgent --> Case[(Prepared CRM Case)]
    RoutingTool --> AgentPool[Human Agent Pool: Min 4 Agents]
    Case -.-> AgentPool

    AgentPool --> AgentDashboard

    AdminDashboard --> Rules
    AdminDashboard --> Metrics[SQLite Metrics / Dashboard]

    %% Auditing
    API --> Audit[(Audit Log Table)]
    Coordinator --> Audit
    ToolLayer --> Audit
```

## 9. API Specification & Execution Endpoints

### Key Endpoints
1. `POST /api/webhooks/telephony`: Receives incoming incoming call webhooks, maps them to agent initialization.
2. `WS /api/chat`: WebSocket for real-time text chat, pushing user chunks into the Orchestrator state.
3. `POST /api/admin/rules`: Update routing rules (JSON).
4. `GET /api/agent/cases`: Fetch assigned cases for a human agent.

### Data Models (Draft)
- **AgentState**: `messages` (list), `patient_id` (str), `intent` (str), `urgency` (enum), `knowledge_context` (str), `handoff_payload` (json).
- **Patient**: `id`, `phone`, `email`, `loyalty_tier`.

## 10. Execution & Local Setup Guide

To be fully executable, the project utilizes Docker Compose and python virtual environments.

### Prerequisites
- Python 3.11+
- Docker & Docker Compose
- SIP Trunk or WebRTC testing setup
- OpenAI API Key (or equivalent LLM provider)

### Setup Instructions

1. **Environment Config:**
   Create `backend/.env` with the following:
   ```env
   OPENAI_API_KEY=sk-xxxx
   DATABASE_URL=sqlite:///./patient_assistant.db
   TELEPHONY_API_KEY=xxxx
   TELEPHONY_WEBHOOK_URL=xxxx
   JWT_SECRET=super_secret_key
   ```

2. **Run Backend (Local Dev):**
   ```bash
   cd backend
   python -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt
   uvicorn main:app --reload --port 8000
   ```

3. **Run via Docker (Full Stack):**
   ```bash
   docker-compose up --build -d
   ```
   *(This triggers the unified Backend API + HTMX Frontend on port 8000)*

## 11. Proposed Development Workflow

### Phase 1: Foundation (Days 1-2)
- Configure SQLite schema via Alembic.
- Scaffold FastAPI and HTMX & Jinja2 boilerplates. Include JWT auth for admin/agents test endpoints.

### Phase 2: Agent Framework & Tools (Days 3-5)
- Implement LangGraph `StateGraph`.
- Define node scripts: `orchestrator.py`, `triage_agent.py`, `knowledge_agent.py`, `handoff_agent.py`.
- Build deterministic tools: Patient ID via mock CRM DB lookup, Routing rules engine.

### Phase 3: External Integrations (Days 6-7)
- Bind telephony webhooks to the LangGraph invocation.
- Convert WebSocket streams for chat integration.

### Phase 4: Frontend Dashboards (Days 8-10)
- Agent Dashboard: Real-time UI updating when the `Handoff Agent` commits a case.
- Admin Dashboard: Adjust routing variables and view analytics.

## 12. PLAN.md Acceptance Checklist

- [x] Includes an industry-standard problem statement.
- [x] Includes exactly 3 distinct user stories.
- [x] Includes a PRD.
- [x] Includes an FRD.
- [x] Includes a user workflow.
- [x] Includes a project structure.
- [x] Includes a project architecture.
- [x] Includes a Mermaid code fence for the project architecture.
- [x] Includes a proposed development workflow.
- [x] Clearly represents the project as an agentic AI system.
- [x] Includes an agent coordinator and specialized AI agents.
- [x] Includes at least 4 specialized AI agent roles.
- [x] Mentions phone and chat as version 1 channels.
- [x] Mentions triage and handoff as version 1 automation.
- [x] Mentions CRM and EHR integrations.
- [x] Mentions that the knowledge base is hybrid and not RAG alone.
- [x] Mentions minimum agents = 4.
- [x] Mentions HIPAA-like privacy, auditability, and access control.
- [x] Includes Execution Instructions and Setup Guide (New Requirement for Development Readiness).
- [x] Includes specific Tech Stack & API Specification (New Requirement for Development Readiness).
