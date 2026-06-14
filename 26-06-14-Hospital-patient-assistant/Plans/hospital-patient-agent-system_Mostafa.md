# Hospital AI Patient Agent System
## Architecture & Solution Design

---

## 1. Problem Statement

Modern hospital patient assistants are constrained by human limitations, leading to degraded patient experience and operational inefficiencies. The current system suffers from four critical pain points that this multi-agent AI solution aims to eliminate.

| Pain Point | Current Impact |
|---|---|
| **High Wait Time** | Patients wait extensively before reaching an assistant |
| **Limited Availability** | System not responsive 24/7; peak hours cause bottlenecks |
| **No Patient History** | Every interaction starts from scratch, zero context |
| **Untapped Loyalty** | Known patients with chronic conditions treated as strangers |

---

## 2. Realistic Assumptions

> All assumptions below are grounded in standard hospital infrastructure.

| # | Assumption | Rationale |
|---|---|---|
| 1 | Hospital has an **EHR system** (e.g., Epic, Cerner) with patient history, chronic conditions, medications, and past visits | Industry standard for any mid-to-large hospital |
| 2 | Hospital has a **scheduling system** with doctor availability and department calendars | Standard operational tooling |
| 3 | Patients have registered **profiles** with phone numbers and basic medical flags | Required for loyalty identification |
| 4 | An **emergency dispatch API** exists or can be integrated | Realistic for hospitals with ambulance services |
| 5 | After each visit, the hospital system **logs the actual department** the patient was routed to | Used for triage performance feedback loop |
| 6 | Historical **wait time data** exists as a baseline for measuring improvement | Available from current system logs |
| 7 | Patients interact via **Voice Calls** and **Web Chat** as primary channels | Defined scope of interaction layer |

---

## 3. Solution Overview

Replace the human patient assistant with a **multi-agent AI system** composed of 5 specialized agents orchestrated by a central hub. The system understands natural language, accesses patient history, triages symptoms using LLM reasoning, manages scheduling, and handles emergencies proactively.

---

## 4. System Architecture

### 4.1 High-Level Architecture

```mermaid
flowchart TD
    subgraph INPUT["Input Layer (Channel Adapter)"]
        VC["📞 Voice Call\nSTT → Text"]
        WC["💻 Web Chat\nDirect Text"]
    end

    subgraph CORE["Core Agent System"]
        OA["🧠 Orchestrator Agent\n(Hub — Maintains Control)"]

        PIA["📋 Patient Intelligence Agent\n(Tool Call)"]
        CTA["🔬 Clinical Triage Agent\n(Tool Call)"]
        OPS["📅 Operations Agent\n(Tool Call)"]
        EA["🚨 Emergency Agent\n(Handoff)"]
    end

    subgraph OPTIONAL["Optional Extension"]
        PCM["⏰ Proactive Care Module\n(Scheduler / Event Trigger)"]
    end

    subgraph KB["Knowledge Base Layer"]
        EHR["🗄️ EHR Database\n(Structured — Patient Records)"]
        RAG1["📚 Medical Knowledge RAG\n(Symptoms · Conditions · Protocols)"]
        RAG2["📁 Hospital Operations RAG\n(Departments · Policies · FAQ)"]
        OPS_DB["📆 Operational DB\n(Schedules · Queue · Availability)"]
    end

    subgraph OUTPUT["Output Layer"]
        TTS["🔊 TTS → Voice Response"]
        WR["💬 Web Chat Response"]
        NOTIF["📱 SMS / Notification"]
    end

    VC --> OA
    WC --> OA

    OA -->|"Tool Call"| PIA
    OA -->|"Tool Call"| CTA
    OA -->|"Tool Call"| OPS
    OA -->|"Handoff on Red Flag"| EA

    CTA -->|"red_flag=true\n→ signals Orchestrator"| OA

    PCM -->|"triggers on schedule\nor EHR event"| EA

    PIA --> EHR
    CTA --> RAG1
    OA  --> RAG2
    OPS --> OPS_DB
    OPS --> EHR

    EA --> NOTIF

    OA --> TTS
    OA --> WR
    EA --> TTS
    EA --> WR
```

---

### 4.2 Agent Interaction & Control Flow

```mermaid
sequenceDiagram
    actor P as Patient
    participant IL as Input Layer
    participant OA as Orchestrator Agent
    participant PIA as Patient Intelligence Agent
    participant CTA as Clinical Triage Agent
    participant OPS as Operations Agent
    participant EA as Emergency Agent

    P->>IL: Contact (Voice / Web Chat)
    IL->>OA: Normalized text input

    OA->>PIA: Tool Call — get_patient_context(identifier)
    PIA-->>OA: {history, conditions, loyalty_status, chronic_flags}

    alt Patient mentions symptoms
        OA->>CTA: Tool Call — triage(symptoms, patient_context)
        CTA-->>OA: {department, urgency, confidence, red_flag}

        alt red_flag = true
            OA->>EA: HANDOFF — full control transferred
            EA->>EA: dispatch_ambulance()
            EA->>EA: alert_medical_team()
            EA->>EA: notify_family()
            EA-->>P: Emergency response + confirmation
        else urgency = HIGH / MEDIUM / LOW
            OA->>OPS: Tool Call — find_availability(department, urgency)
            OPS-->>OA: {slots, estimated_wait}
            OA-->>P: Routing decision + appointment options
        end

    else Patient requests appointment
        OA->>OPS: Tool Call — book_appointment(patient_id, dept, preference)
        OPS-->>OA: {confirmation, doctor, time, location}
        OA-->>P: Appointment confirmed
    end
```

---

### 4.3 Emergency Flow (Detailed)

```mermaid
flowchart TD
    A["Patient sends message"] --> B["Orchestrator receives input"]
    B --> C["Patient Intelligence Agent\nretrieves patient context"]
    C --> D{"Known patient\nwith chronic condition?"}

    D -->|"Yes"| E["Clinical Triage Agent\n(enriched with history)"]
    D -->|"No"| F["Clinical Triage Agent\n(symptom-only reasoning)"]

    E --> G{"Red Flag\nDetected?"}
    F --> G

    G -->|"Yes"| H["Orchestrator triggers\nEmergency HANDOFF"]
    G -->|"No"| I["Normal Routing\nto Operations Agent"]

    H --> J["Emergency Agent takes full control"]
    J --> K["Dispatch Ambulance API"]
    J --> L["Alert Medical Team"]
    J --> M["Notify Family"]
    J --> N["Update Patient Record\n(event logged)"]
    J --> O["Confirm to Patient"]

    style H fill:#ff4444,color:#fff
    style J fill:#ff4444,color:#fff
    style K fill:#ff6666,color:#fff
    style L fill:#ff6666,color:#fff
    style M fill:#ff6666,color:#fff
```

---

### 4.4 Proactive Care Module (Optional Extension)

```mermaid
flowchart LR
    subgraph PCM["⏰ Proactive Care Module"]
        SCH["Scheduler\nRuns on cron / event"]
        CHK["Check chronic patients\nin EHR"]
        EVT["Event Triggers:\n• Missed medication\n• Abnormal reading\n• Appointment due"]
    end

    subgraph EA["🚨 Emergency Agent"]
        EA_LOGIC["Evaluate trigger\nseverity"]
        DISPATCH["Dispatch Ambulance"]
        REMIND["Send Reminder\n/ Follow-up"]
    end

    SCH --> CHK
    CHK --> EVT
    EVT -->|"Critical event"| DISPATCH
    EVT -->|"Routine event"| REMIND
    EVT --> EA_LOGIC
    EA_LOGIC --> DISPATCH
    EA_LOGIC --> REMIND
```

---

## 5. Agent Specifications

### Agent 1 — Orchestrator Agent

| Attribute | Detail |
|---|---|
| **Role** | Central hub; maintains full conversation state and control |
| **Control Type** | Always in control — never fully hands off except to Emergency Agent |
| **Inputs** | Normalized text from Input Layer |
| **Outputs** | Final response to patient (text or voice) |

**Tools:**
- `identify_patient(phone / name / ID)` → calls Patient Intelligence Agent
- `detect_intent(message)` → classifies: appointment / symptom / emergency / inquiry
- `retrieve_hospital_faq(query)` → RAG on hospital policies & procedures
- `route_to_agent(agent_name, payload)` → dispatches to appropriate agent

---

### Agent 2 — Patient Intelligence Agent

| Attribute | Detail |
|---|---|
| **Role** | Single source of truth for patient context |
| **Control Type** | Tool Call — stateless, returns data and exits |
| **Inputs** | Patient identifier (phone / ID) |
| **Outputs** | Patient context package |

**Tools:**
- `get_patient_profile(id)` → EHR lookup: demographics, contact info
- `get_medical_history(id)` → past diagnoses, surgeries, allergies
- `get_chronic_conditions(id)` → flagged long-term conditions
- `get_loyalty_status(id)` → visit frequency, registration date, loyalty tier
- `get_last_visit(id)` → most recent interaction details

**Output Schema:**
```json
{
  "patient_id": "P-00123",
  "name": "Ahmed Hassan",
  "age": 67,
  "chronic_conditions": ["Diabetes Type 2", "Hypertension"],
  "current_medications": ["Metformin", "Amlodipine"],
  "loyalty_status": "HIGH",
  "last_visit": "2025-05-20",
  "primary_doctor": "Dr. Salah — Endocrinology",
  "emergency_contact": "+20-1XX-XXXXXXX"
}
```

---

### Agent 3 — Clinical Triage Agent

| Attribute | Detail |
|---|---|
| **Role** | Symptom understanding and department routing using LLM reasoning |
| **Control Type** | Tool Call — Orchestrator stays in control; red flag signals trigger Emergency handoff |
| **Inputs** | Raw symptoms + patient context from Agent 2 |
| **Outputs** | Structured routing decision |

> **Key Design Decision:** This agent uses **real LLM reasoning** over medical knowledge — not rule-based mapping. The same symptom ("I feel dizzy") produces different routing decisions based on patient age, chronic conditions, and symptom context.

**Tools:**
- `retrieve_medical_knowledge(symptoms)` → RAG on symptom-condition mappings
- `get_department_catalog()` → hospital department list and specialties
- `check_red_flags(symptoms)` → fast-path check before full reasoning
- `assess_urgency(symptoms, patient_context)` → urgency classification

**Red Flag Fast Path (bypasses full reasoning):**
```
"Severe chest pain"          → Emergency immediately
"Loss of consciousness"      → Emergency immediately
"Severe difficulty breathing"→ Emergency immediately
"Signs of stroke"            → Emergency immediately
```

**Confidence Threshold Logic:**
```
confidence ≥ 0.80  →  Route to recommended department
confidence 0.60–0.79 →  Ask one clarifying question, re-evaluate
confidence < 0.60  →  Escalate to human agent
```

**Output Schema:**
```json
{
  "recommended_department": "Cardiology",
  "urgency_level": "HIGH",
  "confidence_score": 0.89,
  "red_flag_detected": false,
  "reasoning": "Patient (67y, hypertensive) reports chest pressure
                with shortness of breath. Symptom profile aligns
                with possible cardiac event. Immediate evaluation
                recommended.",
  "alternative_departments": ["Emergency", "Internal Medicine"],
  "clarifying_questions": [],
  "escalate_to_human": false
}
```

---

### Agent 4 — Operations Agent

| Attribute | Detail |
|---|---|
| **Role** | Scheduling, queue management, and availability |
| **Control Type** | Tool Call — stateless, handles booking logic and returns confirmation |
| **Inputs** | Department + urgency level + patient preferences |
| **Outputs** | Appointment confirmation or queue status |

**Tools:**
- `get_doctor_availability(department, urgency)` → available slots ranked by urgency priority
- `book_appointment(patient_id, doctor_id, slot)` → creates appointment in scheduling system
- `get_queue_status(department)` → current wait time estimation
- `send_confirmation(patient_contact, appointment_details)` → SMS / notification
- `reschedule_appointment(appointment_id, new_slot)` → handles changes

---

### Agent 5 — Emergency Agent

| Attribute | Detail |
|---|---|
| **Role** | Handles all emergency scenarios; executes full response sequence |
| **Control Type** | **Full Handoff** — takes complete control from Orchestrator |
| **Inputs** | Patient context + triage output + emergency trigger reason |
| **Outputs** | Dispatch confirmation + patient notification + record update |

**Tools:**
- `dispatch_ambulance(patient_location, condition_summary)` → triggers emergency dispatch API
- `alert_medical_team(department, patient_context)` → notifies ER team
- `notify_emergency_contact(contact_info, message)` → alerts family
- `update_patient_record(patient_id, event)` → logs emergency event in EHR
- `send_patient_confirmation(channel, message)` → confirms action to patient

**Triggers:**
- Reactive: red flag detected in triage flow
- Proactive *(optional)*: scheduler finds critical event in EHR monitoring

---

## 6. Knowledge Base Architecture

```mermaid
flowchart TD
    subgraph KB["Knowledge Base Layer"]
        subgraph STRUCT["Structured Data (Direct DB Query)"]
            EHR["EHR Database\n• Patient profiles\n• Medical history\n• Chronic conditions\n• Medications"]
            SCHED["Scheduling DB\n• Doctor availability\n• Department calendars\n• Appointment records"]
        end

        subgraph RAG_LAYER["RAG — Retrieval Augmented Generation"]
            MED_RAG["Medical Knowledge Base\n• Symptom-condition mappings\n• Triage protocols (Manchester)\n• Red flag symptom lists\n• ICD-10 references"]
            HOS_RAG["Hospital Operations RAG\n• Department catalog & specialties\n• Hospital policies & procedures\n• Patient FAQ\n• Emergency protocols"]
        end
    end

    PIA["Patient Intelligence Agent"] --> EHR
    OPS["Operations Agent"] --> SCHED
    OPS --> EHR
    CTA["Clinical Triage Agent"] --> MED_RAG
    OA["Orchestrator Agent"] --> HOS_RAG
```

| Knowledge Type | Storage | Used By | Update Frequency |
|---|---|---|---|
| Patient Records | EHR DB (structured) | Agent 2, Agent 4 | Real-time |
| Doctor Schedules | Scheduling DB | Agent 4 | Real-time |
| Medical Knowledge | RAG index | Agent 3 | Monthly / on update |
| Hospital Policies | RAG index | Orchestrator | Quarterly / on update |
| Emergency Protocols | RAG index | Agent 3, Agent 5 | On update |

---

## 7. Channel Layer (Input / Output)

```mermaid
flowchart LR
    subgraph IN["Input Channels"]
        VC["📞 Voice Call"]
        WC["💻 Web Chat"]
    end

    subgraph ADAPTER["Channel Adapter Layer"]
        STT["STT Engine\n(Speech-to-Text)"]
        NORM["Text Normalizer\n(Arabic / English)"]
    end

    subgraph OUT["Output Channels"]
        TTS["🔊 TTS Engine\n(Text-to-Speech)"]
        WR["💬 Web Response"]
        SMS["📱 SMS / Push Notification"]
    end

    VC --> STT --> NORM
    WC --> NORM
    NORM -->|"Unified text"| OA["Orchestrator Agent"]
    OA --> TTS --> VC_OUT["📞 Voice Response"]
    OA --> WR
    EA["Emergency Agent"] --> SMS
```

> The Orchestrator always receives and processes **unified normalized text** regardless of channel. Channel-specific rendering happens in the Output Layer.

---

## 8. Tool Call vs. Handoff Decision

```mermaid
flowchart TD
    A["Orchestrator receives\npatient intent"] --> B["Always: Tool Call to\nPatient Intelligence Agent"]
    B --> C{"Symptoms\nmentioned?"}

    C -->|"Yes"| D["Tool Call to\nClinical Triage Agent"]
    C -->|"No"| E["Tool Call to\nOperations Agent"]

    D --> F{"red_flag\ndetected?"}
    F -->|"Yes"| G["⚠️ HANDOFF\nEmergency Agent\ntakes full control"]
    F -->|"No"| E

    E --> H["Orchestrator composes\nfinal response"]
    G --> I["Emergency Agent executes\ncomplete response sequence"]

    style G fill:#ff4444,color:#fff
    style I fill:#ff4444,color:#fff
```

| Agent | Control Type | Reason |
|---|---|---|
| Patient Intelligence | Tool Call | Stateless data retrieval |
| Clinical Triage | Tool Call | Returns routing decision; Orchestrator stays in control |
| Operations | Tool Call | Stateless booking and scheduling |
| Emergency | **Full Handoff** | Time-critical; needs to own the full response sequence |

---

## 9. Performance Measurement

### 9.1 Assumed Feedback Loop

> **Assumption:** After each patient visit, the hospital system logs the actual department visited. This enables comparison with the system's recommended department.

### 9.2 Metrics per Agent

```mermaid
flowchart LR
    subgraph M1["Orchestrator Metrics"]
        M1A["Intent Classification Accuracy"]
        M1B["First Contact Resolution Rate\n(no human escalation needed)"]
        M1C["Session Completion Rate"]
    end

    subgraph M2["Patient Intelligence Metrics"]
        M2A["Patient Identification Accuracy"]
        M2B["History Retrieval Latency"]
    end

    subgraph M3["Clinical Triage Metrics"]
        M3A["Routing Accuracy\n(recommended vs actual dept)"]
        M3B["Confidence Calibration\n(does confidence reflect accuracy?)"]
        M3C["Red Flag Recall\n(did we catch all true emergencies?)"]
        M3D["False Positive Rate\n(unnecessary emergency dispatches)"]
    end

    subgraph M4["Operations Metrics"]
        M4A["Average Wait Time\n(before vs after)"]
        M4B["Appointment Booking Success Rate"]
        M4C["Schedule Utilization Rate"]
    end

    subgraph M5["Emergency Metrics"]
        M5A["Emergency Response Time\n(red flag → dispatch)"]
        M5B["False Dispatch Rate"]
        M5C["Proactive Alert Accuracy"]
    end
```

### 9.3 System-Level KPIs

| KPI | Measurement Method | Target |
|---|---|---|
| Average Wait Time Reduction | Historical baseline vs live average | > 60% reduction |
| First Contact Resolution Rate | Sessions resolved without human | > 80% |
| Triage Routing Accuracy | Recommended dept vs actual dept (logged) | > 85% |
| Emergency Response Time | Timestamp: red flag detected → dispatch | < 90 seconds |
| Red Flag Recall | True emergency caught / total true emergencies | > 99% |
| False Emergency Dispatch Rate | Unnecessary dispatches / total dispatches | < 2% |
| Patient Satisfaction Score | Post-session rating (1–5) | > 4.2 avg |

---

## 10. Complete Data Flow Summary

```mermaid
flowchart TD
    P["👤 Patient"] --> IL["Input Layer\n(Voice STT / Web Chat)"]
    IL --> OA["🧠 Orchestrator Agent"]

    OA --> PIA["📋 Patient Intelligence Agent\n→ returns patient context"]
    PIA --> EHR_DB[("EHR Database")]

    OA --> CTA["🔬 Clinical Triage Agent\n→ returns routing decision"]
    CTA --> MED_RAG[("Medical Knowledge RAG")]

    OA --> OPS["📅 Operations Agent\n→ returns booking confirmation"]
    OPS --> SCHED_DB[("Scheduling DB")]

    OA -->|"Red Flag Handoff"| EA["🚨 Emergency Agent"]
    EA --> DISPATCH["Ambulance Dispatch API"]
    EA --> ALERT["Medical Team Alert"]
    EA --> FAMILY["Family Notification"]
    EA --> EHR_UPDATE[("EHR — Event Log")]

    OA --> OUT["Output Layer\n(Voice TTS / Web / SMS)"]
    EA --> OUT

    PCM["⏰ Proactive Care Module\n(Optional)"] -->|"Scheduled trigger"| EA

    style EA fill:#cc2222,color:#fff
    style DISPATCH fill:#cc4444,color:#fff
    style ALERT fill:#cc4444,color:#fff
    style FAMILY fill:#cc4444,color:#fff
```

---

## 11. Architecture Decision Log

| Decision | Options Considered | Chosen | Reason |
|---|---|---|---|
| Agent control model | Full handoff vs Hub-and-Spoke vs Hybrid | **Hybrid** | Orchestrator controls all except Emergency |
| Clinical Triage type | Rule-based vs LLM Reasoning | **LLM Reasoning** | Natural language symptoms require real understanding |
| Emergency control | Tool Call vs Handoff | **Handoff** | Time-critical; needs to own full response sequence |
| Clinical Triage control | Handoff vs Tool Call | **Tool Call** | Returns a decision; Orchestrator acts on it |
| Knowledge Base | DB only vs RAG only vs Both | **Both** | Structured for patient data, RAG for medical knowledge |
| Channel scope | Single vs Multi-channel | **Multi** (Voice + Web) | Realistic patient interaction coverage |
| Proactive module | Built-in vs Optional Extension | **Optional** | Core system works without it; adds complexity |
| Confidence fallback | Human escalation vs Re-prompt | **Threshold-based** | <0.60 → human; 0.60–0.79 → clarify; >0.80 → route |

---

*Document generated as part of Hospital AI Patient Agent System — Architecture & Solution Design*
*All assumptions are realistic and grounded in standard hospital infrastructure.*
