# Patient AI Assistant
### *A 24/7 Digital Front Door for the Hospital* (Business Proposal)

Hospitals lose time, money, and patient trust to a problem that is largely **administrative**: phone lines are jammed, receptionists repeat the same questions, clinicians lack context when patients call, and there is no consistent way to determine who needs help first.

The **Patient AI Assistant** is a 24/7 intelligent front door that answers calls and chats, remembers every patient, routes them to the right resource, and flags emergencies in seconds. It works alongside staff *— not in place of them —* and is designed to deliver measurable savings within the first year of deployment.

---

## 1. The Problem

Hospitals face the same five frustrations every day. None of them require new medicine to solve; they require better **coordination**.

### 1.1 Wasted Time
* Patients repeat the same information to every person they speak with.  
* Receptionists spend 40–60% of their day on questions that do not need a human.  
* Clinicians enter visits without context, so the first 3–5 minutes are spent catching up.

### 1.2 Limited Availability
* Phone lines are closed at night and on weekends.  
* Booking the right specialist can take multiple callbacks.  
* Emergency room capacity is invisible to patients until they arrive.

### 1.3 Long Call Times
* Average wait to reach a human: 4–12 minutes during peak hours.  
* 20–30% of callers abandon before being answered.  
* Each abandoned call is a lost visit and a frustrated patient.

### 1.4 No Continuity
* Every call starts from zero.  
* Clinicians do not remember a patient they saw 3 months ago.  
* Follow-ups depend on the patient remembering to call back.

### 1.5 No Prioritization
* Calls are answered in the order they arrive.  
* A chest-pain patient may wait behind a prescription refill.  
* Risk is identified too late.

---

## 2. The Solution and Who It Serves

```mermaid
flowchart TB
    AI["Patient AI Assistant: Always-on, multilingual, remembers everything"]

    AI -->|Faster answers, no repetition, 24/7 access| P["Patients"]
    AI -->|Fewer routine calls, focus on care| R["Reception Staff"]
    AI -->|Patient summary before they join| C["Clinicians: Nurses and GPs"]
    AI -->|Early warning, pre-arrival context| E["Emergency Team"]
    AI -->|Live dashboards, capacity insight| M["Hospital Management"]
    AI -->|Audit logs, compliance reports| IT["IT and Compliance"]

    P -->|Loyalty and reviews| H["Hospital"]
    R -->|Lower workload| H
    C -->|More time for care| H
    E -->|Better outcomes| H
    M -->|Lower cost, higher revenue| H
    IT -->|Lower risk| H
```

---

## 3. Patient Journey

```mermaid
flowchart LR
    A["Patient contact: call, chat, app"] --> B{"New or returning?"}
    B -->|Returning| C["Greeted by name, history loaded (History Agent)"]
    B -->|New| D["Quick 30-sec registration (Comm Agent)"]
    C --> E["Understands need in plain language (Comm Agent)"]
    D --> E
    E --> F{"Need type?"}
    F -->|Routine| G["Books slot or answers question (Scheduling Agent)"]
    F -->|Medical| H["Asks symptom questions (Triage Agent)"]
    F -->|Emergency| I["Pages clinician in under 30 sec (Escalation Agent)"]
    H --> J{"Risk level evaluation"}
    J -->|Low| K["Self-care advice delivered (Triage Agent)"]
    J -->|Medium| L["Same-day appointment booked (Scheduling Agent)"]
    J -->|High| I
    I --> M["Clinician joins with full context (Escalation Agent)"]
    K --> N["Confirmation and follow-up reminder (Comm Agent)"]
    G --> N
    L --> N
    M --> N
    N --> O["Satisfied patient and audit log captured"]
```

---

## 4. Data Requirements

### 4.1 Data Types

| Category | Examples | Source |
| :--- | :--- | :--- |
| **Patient demographics** | Age, sex, language, contact | Hospital registration |
| **Clinical history** | Diagnoses, meds, allergies, prior visits | EHR / mock dataset |
| **Current encounter** | Symptoms, vitals, chief complaint | Patient self-report + devices |
| **Resource data** | Doctor schedules, room status, on-call roster | Hospital admin system |
| **Interaction logs** | Call transcripts, chat history, agent decisions | System-generated |
| **Knowledge base** | Triage protocols, drug interactions | Medical guidelines (WHO, NICE) |

---

## 5. Ethics, Privacy, and Compliance

* **Informed Consent:** Required for AI interaction (verbal at call start, click-through on chat features).
* **Human-in-the-Loop:** Mandatory for any operational decision mapped above the yellow band tier.

> ### The 5-level Emergency Severity Index (ESI) Framework:
> * **Levels 1 & 2 (Red/Orange Bands):** Immediate life threats (e.g., anaphylaxis, severe trauma). System architecture must abort AI interaction, trigger a high-priority WebSocket alert to the charge nurse desk, and provide automatic phone routing.
> * **Level 3 (Yellow Band):** Requires multiple resources but stable vitals. This is where your AI context-gathering shines, summarizing the information for the clinician before they step in.
> * **Levels 4 & 5 (Green/Blue Bands):** Non-urgent schedules/refills. Fully automatable by your assistant framework.

* **Explainability:** Show the patient *why* they were routed (e.g., *"Based on your reported chest pain and history, we are routing you to..."*).
* **Compliance Posture:** Absolute structural alignment with **HIPAA** (US), **GDPR** (EU), and local medical-data sovereignty laws.
* **Bias Checks:** Continuous internal validation of system risk scoring across age, sex, and language subgroups.

---

## Guardrail Placement

### 1. The Output Guardrail
* **Where:** Post-Generation / Pre-Delivery. Positioned directly between the **Triage Agent** and the **User Output Channel**.  
* **Function:** Holds the generated response in a temporary state buffer. It acts as a synchronous interceptor to ensure the AI never delivers a definitive medical diagnosis (e.g., *"You are having a heart attack"*) and strictly sticks to safe triage protocol language.

### 2. The Input Guardrail
* **Where:** Post-Transcription / Pre-Routing. At the absolute front door before the payload hits the **Router Agent**.  
* **Function:** Sanitizes incoming text/audio data to block jailbreak attempts, adversarial prompt injections, or immediate red-band emergencies that must bypass conversational AI entirely.

```mermaid
flowchart TB
    P1["Patient calls, chats, or opens app"] --> C1["Answers in under 2 sec"]
    C1 --> C2["Quick registration if new patient"]
    C2 --> C3["Understands need in plain language"]
    
    C2 -->|If returning patient| H1["Greeted by name, history loaded"]
    H1 --> C3
    
    C3 --> T1["Asks focused symptom questions"]
    T1 --> T2["Computes risk score and band"]
    T2 --> T4{"Risk Assessment"}
    
    T4 -->|Low Risk| T3["Provides self-care advice"]
    T4 -->|Medium Risk| S2["Books same-day urgent slot"]
    T4 -->|High Risk| E1["Pages on-call clinician"]
    
    T3 --> S1["Books routine appointment"]
    C3 --> S1
    E1 --> E2["Clinician joins with full context"]
    
    T3 --> C4["Sends confirmation and reminder"]
    S1 --> C4
    S2 --> C4
    E2 --> C4
    
    C4 --> P2["Receives confirmation and reminder"]
    P2 --> P3["Outcome: satisfied, issue resolved"]
```

---

## Technical System Architecture

```mermaid
flowchart TB
    subgraph PAT["Patients"]
        P1["Patient Voice"]
        P2["Patient Chat and Web"]
    end

    subgraph CLIN["Clinicians"]
        D1["On-call Clinician"]
        D2["GP and Specialist"]
    end

    GW["Intake Gateway: ASR and Channel Router"]

    subgraph ORCH["Orchestration Layer"]
        ORC["Orchestrator: Planner and ReAct Loop"]
        ST[("Session State Memory")]
        POL["Policy Guardrails: Refusals and Red Flags"]
    end

    subgraph AGENTS["Specialist Agents"]
        A1["Triage Agent: Risk Score 0-1"]
        A2["History Agent: Patient Memory"]
        A3["Scheduling Agent: Slots and Resources"]
        A4["Communication Agent: Voice, SMS, Email"]
        A5["Escalation Agent: Clinician Paging"]
    end

    subgraph RAG["RAG Layer: Retrieval-Augmented Generation"]
        EMB["Embedder: bge / OpenAI / Cohere"]
        VKB[("Vector Store KB: Protocols, guidelines, red-flags")]
        VPH[("Vector Store Patient: Episodic memory, prior visits")]
        VFAQ[("Vector Store FAQ: Hospital policies")]
        RER["Reranker: cross-encoder"]
    end

    subgraph DATA["Structured Data"]
        PDB[("Patient DB: Profiles, history")]
        CAL[("Resource Calendar: Doctors, rooms, ER")]
        LOG[("Audit and Metrics Log")]
    end

    subgraph EXT["External Services"]
        TEL["Telephony: Twilio / Vonage"]
        SMS["SMS and Email Gateway"]
        PAG["Pager and Alert System"]
    end

    P1 --> GW
    P2 --> GW
    GW --> ORC
    ORC <--> ST
    ORC --> POL

    ORC --> A1
    ORC --> A2
    ORC --> A3
    ORC --> A4
    ORC --> A5

    %% RAG wiring
    A1 --> EMB
    EMB --> VKB
    VKB --> RER
    RER --> A1

    A2 --> EMB
    EMB --> VPH
    VPH --> RER
    RER --> A2

    A4 --> EMB
    EMB --> VFAQ
    VFAQ --> RER
    RER --> A4

    A2 <--> PDB
    A3 <--> CAL
    A4 --> TEL
    A4 --> SMS
    A5 --> PAG

    A5 --> D1
    A3 --> D2
    A4 --> P1
    A4 --> P2

    ORC --> LOG
    A1 --> LOG
    A2 --> LOG
    A3 --> LOG
    A4 --> LOG
    A5 --> LOG
    EMB --> LOG
    RER --> LOG
```
