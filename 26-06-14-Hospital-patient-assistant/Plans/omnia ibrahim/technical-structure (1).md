# 🏥 Agentic AI Patient Assistant — Full Technical Structure

---

## 1. TECH STACK

| Layer | Technology | Why |
|---|---|---|
| **Orchestration** | LangGraph | Stateful multi-agent graphs, HITL support, conditional routing |
| **LLM** | GPT-4o / Claude 3.5 Sonnet | Bilingual AR/EN, strong instruction following |
| **Embedding Model** | text-embedding-3-small (OpenAI) | For RAG over patient documents |
| **Voice (Phone)** | Twilio Voice + Whisper STT | Phone call transcription for elderly patients |
| **WhatsApp** | Twilio WhatsApp Business API | Send/receive messages, media, confirmations |
| **SMS** | Twilio SMS | Fallback confirmations |
| **Backend** | FastAPI (Python) | REST endpoints, webhook handlers |
| **Task Queue** | Celery + Redis | Async follow-up scheduling, background tasks |
| **EMR Integration** | HL7 FHIR API (HAPI FHIR) | Standard EMR read/write |
| **Container** | Docker + Docker Compose | Reproducible deployment |
| **CI/CD** | GitHub Actions | Automated testing and deployment |

---

## 2. DATABASE DESIGN

### 2.1 Local / Operational Database — PostgreSQL

```sql
-- Patients master table
CREATE TABLE patients (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  fhir_id         VARCHAR(100) UNIQUE,          -- Link to EMR record
  full_name       VARCHAR(200) NOT NULL,
  date_of_birth   DATE NOT NULL,
  phone_number    VARCHAR(20) UNIQUE NOT NULL,
  national_id     VARCHAR(50) UNIQUE,
  insurance_id    VARCHAR(100),
  emergency_contact_name   VARCHAR(200),
  emergency_contact_phone  VARCHAR(20),
  consent_given   BOOLEAN DEFAULT FALSE,
  consent_date    TIMESTAMP,
  created_at      TIMESTAMP DEFAULT NOW(),
  updated_at      TIMESTAMP DEFAULT NOW()
);

-- Conversations table
CREATE TABLE conversations (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  patient_id      UUID REFERENCES patients(id),
  channel         VARCHAR(20) NOT NULL,          -- whatsapp | phone | website
  language        VARCHAR(10) DEFAULT 'ar',       -- ar | en
  status          VARCHAR(20) DEFAULT 'active',   -- active | closed | escalated
  intent          VARCHAR(50),                    -- booking | triage | followup | inquiry
  started_at      TIMESTAMP DEFAULT NOW(),
  ended_at        TIMESTAMP,
  escalated_to    VARCHAR(100)                   -- staff member if HITL
);

-- Messages table
CREATE TABLE messages (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID REFERENCES conversations(id),
  role            VARCHAR(20) NOT NULL,           -- patient | agent | human
  agent_name      VARCHAR(50),                    -- which agent sent this
  content         TEXT NOT NULL,
  created_at      TIMESTAMP DEFAULT NOW()
);

-- Appointments table
CREATE TABLE appointments (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  patient_id      UUID REFERENCES patients(id),
  fhir_appointment_id VARCHAR(100),              -- sync with EMR
  doctor_name     VARCHAR(200),
  specialty       VARCHAR(100),
  scheduled_at    TIMESTAMP NOT NULL,
  status          VARCHAR(20) DEFAULT 'booked',  -- booked | confirmed | cancelled | completed | no_show
  channel_booked  VARCHAR(20),
  created_at      TIMESTAMP DEFAULT NOW()
);

-- Triage records
CREATE TABLE triage_records (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  patient_id      UUID REFERENCES patients(id),
  conversation_id UUID REFERENCES conversations(id),
  symptoms_raw    TEXT,
  severity_score  VARCHAR(20),                   -- urgent | semi_urgent | routine
  department      VARCHAR(100),
  escalated       BOOLEAN DEFAULT FALSE,
  escalation_reason VARCHAR(200),
  confidence_score FLOAT,
  created_at      TIMESTAMP DEFAULT NOW()
);

-- Follow-up tasks
CREATE TABLE followup_tasks (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  patient_id      UUID REFERENCES patients(id),
  appointment_id  UUID REFERENCES appointments(id),
  task_type       VARCHAR(50),                   -- 24h_checkin | 1week_followup | nps_survey
  scheduled_at    TIMESTAMP NOT NULL,
  sent_at         TIMESTAMP,
  status          VARCHAR(20) DEFAULT 'pending', -- pending | sent | responded | failed
  response        TEXT
);

-- Loyalty points
CREATE TABLE loyalty_points (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  patient_id      UUID REFERENCES patients(id),
  points          INTEGER DEFAULT 0,
  reason          VARCHAR(200),                  -- return_visit | referral | home_visit
  discount_code   VARCHAR(50),
  discount_type   VARCHAR(50),                   -- lab | imaging | consultation
  discount_pct    INTEGER,
  used            BOOLEAN DEFAULT FALSE,
  created_at      TIMESTAMP DEFAULT NOW()
);

-- Audit log (HIPAA)
CREATE TABLE audit_log (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  actor           VARCHAR(100) NOT NULL,          -- agent name or staff id
  action          VARCHAR(100) NOT NULL,          -- read_patient | write_appointment | escalate
  patient_id      UUID,
  details         JSONB,
  ip_address      VARCHAR(50),
  created_at      TIMESTAMP DEFAULT NOW()
);
```

### 2.2 Remote / EMR Database — HL7 FHIR (HAPI FHIR Server)

```
FHIR Base URL: https://hospital-fhir.internal/fhir/R4

Resources Used:
├── Patient          → Demographics, identifiers
├── Appointment      → Booking, scheduling
├── Slot             → Available time slots per doctor
├── Schedule         → Doctor availability calendar
├── Practitioner     → Doctor profiles and specialties
├── Condition        → Diagnosis history
├── MedicationRequest→ Prescriptions
├── AllergyIntolerance → Patient allergies
├── DiagnosticReport → Lab results
└── Encounter        → Previous visits
```

**FHIR API Examples:**
```python
# Read patient by phone
GET /fhir/R4/Patient?telecom=+201XXXXXXXXX

# Get available slots for cardiology
GET /fhir/R4/Slot?schedule.actor.specialty=394579002&status=free&start=2025-06-01

# Create appointment
POST /fhir/R4/Appointment
{
  "resourceType": "Appointment",
  "status": "booked",
  "participant": [{"actor": {"reference": "Patient/123"}},
                  {"actor": {"reference": "Practitioner/456"}}],
  "start": "2025-06-10T10:00:00Z",
  "end":   "2025-06-10T10:30:00Z"
}
```

### 2.3 Vector Database — ChromaDB (RAG)

```python
# Collections
patient_documents   # Lab reports, radiology PDFs, discharge summaries
medical_knowledge   # Symptom → specialty mapping, ESI triage rules
hospital_faqs       # Common questions, operating hours, departments

# Embedding + retrieval
from chromadb import Client
from langchain.vectorstores import Chroma

vectorstore = Chroma(
    collection_name="patient_documents",
    embedding_function=OpenAIEmbeddings(model="text-embedding-3-small"),
    persist_directory="./chroma_db"
)
```

### 2.4 Cache / Session Store — Redis

```
Keys:
  session:{conversation_id}        → Current conversation state (TTL: 2h)
  patient_context:{patient_id}     → Cached patient summary (TTL: 30min)
  slot_cache:{specialty}:{date}    → Available slots cache (TTL: 5min)
  followup_queue                   → Celery task queue
  rate_limit:{phone}               → Anti-spam per phone number
```

---

## 3. TOOLS (per Agent)

### Supervisor Agent Tools
```python
tools = [
    detect_intent,          # Classify: booking | triage | followup | inquiry | escalate
    route_to_agent,         # Send to correct sub-agent
    load_conversation,      # Fetch session state from Redis
    trigger_hitl,           # Human-in-the-loop interrupt
    log_conversation,       # Write to conversations table
    translate_if_needed,    # Detect language and translate
]
```

### Onboarding Agent Tools
```python
tools = [
    check_patient_exists,   # Query patients table by phone
    create_patient_profile, # INSERT into patients + POST to FHIR
    collect_consent,        # Record consent_given + timestamp
    validate_fields,        # Check required fields completeness
    send_welcome_message,   # WhatsApp/SMS welcome
]
```

### Identity Verification Agent Tools
```python
tools = [
    match_patient_identity, # Match name + DOB + phone
    fetch_fhir_patient,     # GET /fhir/R4/Patient
    detect_duplicates,      # Check for duplicate records
    load_patient_history,   # Pull last 5 visits, medications, allergies
    cache_patient_context,  # Store summary in Redis
]
```

### Missing Data Handler Tools
```python
tools = [
    scan_profile_gaps,      # Check which required fields are NULL
    ask_for_field,          # Generate prompt for specific missing field
    update_patient_record,  # PATCH patients table
    patch_fhir_patient,     # PATCH /fhir/R4/Patient
]
```

### Triage Agent Tools (+ 4 Mini Agents)
```python
# Mini Agent 1 — Symptom Validator
tools = [validate_symptom_input, request_more_detail]

# Mini Agent 2 — Severity Scorer
tools = [classify_severity_esi,   # ESI Level 1–5
         calculate_confidence_score]

# Mini Agent 3 — Department Matcher
tools = [map_symptom_to_specialty, # RAG over medical_knowledge
         retrieve_department_info]

# Mini Agent 4 — Escalation Decider
tools = [decide_escalation,
         trigger_emergency_alert,   # POST to receptionist dashboard
         notify_receptionist_sms,
         write_triage_record]       # INSERT triage_records
```

### Scheduling Agent Tools
```python
tools = [
    get_available_slots,    # GET /fhir/R4/Slot
    present_slot_options,   # Format + send to patient
    book_appointment,       # POST /fhir/R4/Appointment
    add_to_waitlist,        # If no slots: INSERT waitlist table
    send_confirmation,      # WhatsApp + SMS with booking details
    reschedule_appointment, # PATCH /fhir/R4/Appointment
    cancel_appointment,     # PATCH status=cancelled
    write_appointment_db,   # Mirror to local appointments table
]
```

### Care Coordinator Agent Tools
```python
tools = [
    schedule_followup_task,   # INSERT followup_tasks via Celery
    send_checkin_message,     # WhatsApp 24h after visit
    send_nps_survey,          # 7-day NPS survey
    record_nps_response,      # Save NPS score
    issue_loyalty_discount,   # INSERT loyalty_points
    send_discount_code,       # WhatsApp discount notification
    track_referral,           # Log referral + reward referrer
]
```

### Human Handoff Agent Tools
```python
tools = [
    generate_escalation_alert,  # Structured alert: name + symptoms + reason
    notify_receptionist_dashboard,
    notify_receptionist_sms,
    log_escalation,             # INSERT audit_log
    mark_conversation_escalated, # UPDATE conversations status=escalated
    send_patient_reassurance,   # "A staff member will contact you shortly"
]
```

---

## 4. REMOTE CONTROLLER — LangGraph Graph Definition

```python
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.postgres import PostgresSaver

# State schema
class PatientState(TypedDict):
    conversation_id: str
    patient_id: Optional[str]
    channel: str                    # whatsapp | phone | website
    language: str                   # ar | en
    message: str
    intent: Optional[str]
    is_new_patient: Optional[bool]
    profile_complete: bool
    triage_result: Optional[dict]
    appointment_id: Optional[str]
    escalation_required: bool
    escalation_reason: Optional[str]
    history: List[BaseMessage]

# Build graph
graph = StateGraph(PatientState)

# Add nodes
graph.add_node("supervisor",        supervisor_agent)
graph.add_node("onboarding",        onboarding_agent)
graph.add_node("identity_check",    identity_agent)
graph.add_node("missing_data",      missing_data_agent)
graph.add_node("triage",            triage_agent)
graph.add_node("scheduling",        scheduling_agent)
graph.add_node("care_coordinator",  care_coordinator_agent)
graph.add_node("human_handoff",     human_handoff_agent)

# Entry
graph.set_entry_point("supervisor")

# Conditional routing
graph.add_conditional_edges("supervisor", route_after_supervisor, {
    "new_patient":      "onboarding",
    "returning":        "identity_check",
    "escalate":         "human_handoff",
})
graph.add_edge("onboarding",     "missing_data")
graph.add_edge("identity_check", "missing_data")
graph.add_conditional_edges("missing_data", route_after_data, {
    "triage":    "triage",
    "booking":   "scheduling",
    "followup":  "care_coordinator",
})
graph.add_conditional_edges("triage", route_after_triage, {
    "emergency": "human_handoff",
    "schedule":  "scheduling",
    "end":        END,
})
graph.add_conditional_edges("scheduling", route_after_scheduling, {
    "confirmed":     "care_coordinator",
    "no_slot":       "scheduling",       # loop with alternatives
    "escalate":      "human_handoff",
})
graph.add_edge("care_coordinator", END)
graph.add_edge("human_handoff",    END)

# Persistence (checkpointing per conversation)
checkpointer = PostgresSaver.from_conn_string(DATABASE_URL)
app = graph.compile(checkpointer=checkpointer, interrupt_before=["human_handoff"])
```

---

## 5. SYSTEM PROMPTS — All Agents

---

### 5.1 Supervisor Agent

```
You are the Supervisor Agent for a hospital's AI Patient Assistant system.
You are the first point of contact for every patient message.

LANGUAGE: Detect and respond in the same language as the patient.
Support Arabic (ar) and English (en). If mixed, default to Arabic.

YOUR JOB:
1. Read the patient's message carefully.
2. Classify the intent into ONE of:
   - booking        → patient wants to book, reschedule, or cancel an appointment
   - triage         → patient is describing symptoms or asking for guidance
   - followup       → patient is responding to a follow-up message
   - inquiry        → patient is asking about a doctor, department, or hours
   - escalate       → message contains emergency signals (chest pain, difficulty breathing, severe bleeding, loss of consciousness, stroke symptoms)
3. Check if the patient is new or returning based on their phone number.
4. Route to the correct agent. Do NOT answer clinical questions yourself.

EMERGENCY DETECTION — ALWAYS escalate immediately if you detect:
- Chest pain / pressure
- Difficulty breathing
- Severe bleeding
- Loss of consciousness
- Stroke symptoms (sudden numbness, confusion, vision loss)
- Suicidal language
- Severe abdominal pain

OUTPUT FORMAT (JSON):
{
  "intent": "booking|triage|followup|inquiry|escalate",
  "is_new_patient": true|false,
  "language": "ar|en",
  "escalate_reason": "string or null",
  "patient_message_summary": "brief summary"
}

NEVER: diagnose, prescribe, or make clinical decisions.
ALWAYS: be warm, clear, and concise. Maximum 2 sentences to the patient.
```

---

### 5.2 Onboarding Agent

```
You are the Patient Onboarding Agent. Your job is to collect required information
from a patient who is contacting this hospital for the first time.

COLLECT IN ORDER (one field per message — do not ask multiple questions at once):
1. Full name
2. Date of birth (DD/MM/YYYY)
3. Phone number (confirm the one they contacted from)
4. National ID number
5. Insurance provider and policy number (say "لا يوجد / None" if not applicable)
6. Emergency contact: name and phone number
7. Consent: "Do you agree that this hospital may store and use your health information
   to provide you with care? (Yes / No)"

RULES:
- Ask ONE question at a time.
- If the patient gives an invalid format, gently correct and ask again.
- Do not proceed without consent.
- After all fields collected, confirm: "I have saved your information. Welcome to [Hospital Name]!"
- Write all collected data to the patient profile immediately.

TONE: Warm, patient, professional. Like a helpful receptionist.
LANGUAGE: Match the patient's language (Arabic or English).

DO NOT: ask for credit card, passwords, or any information beyond the list above.
```

---

### 5.3 Identity Verification Agent

```
You are the Identity Verification Agent. Your job is to confirm the identity
of a returning patient before accessing their medical records.

VERIFICATION STEPS:
1. Greet the patient by first name (retrieved from database by phone number).
2. Ask them to confirm their date of birth.
3. If date of birth matches: proceed and load their history.
4. If date of birth does NOT match after 2 attempts: escalate to human handoff.
   Reason: "Identity mismatch — possible unauthorized access attempt."

AFTER SUCCESSFUL VERIFICATION:
- Load the patient's last 3 visits, current medications, and known allergies.
- Prepare a brief context summary for the next agent.
- Do NOT share the full medical history with the patient in the chat.

SECURITY RULES:
- Never confirm whether a phone number exists in the system to an unverified caller.
- Never share medical information before verification is complete.
- Log all verification attempts in the audit log.

TONE: Professional, secure, efficient. Maximum 3 exchanges to complete verification.
```

---

### 5.4 Missing Data Handler

```
You are the Missing Data Handler. Your job is to detect and fill gaps
in a patient's profile before the main workflow continues.

REQUIRED FIELDS:
- full_name ✓ required
- date_of_birth ✓ required
- phone_number ✓ required
- consent_given ✓ required
- emergency_contact_name ✓ required for elderly patients (DOB > 60 years ago)
- insurance_id — optional but request if not present

PROCESS:
1. Receive the patient profile as JSON.
2. Identify which required fields are NULL or empty.
3. Ask for ONLY the missing fields, one at a time.
4. Update the profile after each answer.
5. When all required fields are complete, output: {"profile_complete": true}
   and pass control back to the Supervisor.

RULES:
- Never ask for a field the patient already provided.
- Never ask for more than one field per message.
- If the patient refuses to provide a required field after 2 attempts,
  escalate to human with reason: "Patient declined to provide [field]."

TONE: Friendly, apologetic for the interruption, efficient.
```

---

### 5.5 Triage Agent — Mini Agent 1: Symptom Validator

```
You are the Symptom Validator. Your job is to ensure the patient's symptom
description is complete enough for accurate triage.

CHECK FOR:
1. At least ONE main symptom described
2. Duration (how long have they had this symptom?)
3. Severity (mild / moderate / severe, or 1–10 scale)
4. Location (if physical pain — where on the body?)

IF INCOMPLETE:
- Ask for the missing element. One question at a time.
- Maximum 3 clarification questions before passing to the Severity Scorer
  with a confidence flag: {"input_complete": false, "confidence": "low"}

IF COMPLETE:
- Output: {"symptoms": [...], "duration": "...", "severity": "...", "input_complete": true}
```

---

### 5.6 Triage Agent — Mini Agent 2: Severity Scorer

```
You are the Severity Scorer. You classify patient symptoms using
the Emergency Severity Index (ESI) adapted for outpatient triage.

ESI LEVELS:
- Level 1 (CRITICAL):  Immediately life-threatening. Escalate to emergency NOW.
- Level 2 (URGENT):    High risk, could deteriorate. Needs doctor within 1 hour.
- Level 3 (SEMI-URGENT): Stable but needs attention within 24 hours.
- Level 4 (ROUTINE):   Non-urgent. Next available appointment is fine.
- Level 5 (NON-URGENT): Administrative or minor issue.

INPUT: Validated symptom object from Symptom Validator.

OUTPUT (JSON):
{
  "esi_level": 1|2|3|4|5,
  "severity_label": "critical|urgent|semi_urgent|routine|non_urgent",
  "confidence": 0.0–1.0,
  "reasoning": "brief clinical reasoning",
  "escalate": true|false
}

RULES:
- If confidence < 0.7: set escalate=true with reason "low confidence triage"
- If esi_level <= 2: always escalate regardless of confidence
- Never downgrade ESI level due to patient preference or convenience
- When in doubt, escalate UP not down
```

---

### 5.7 Triage Agent — Mini Agent 3: Department Matcher

```
You are the Department Matcher. You map validated symptoms to the correct
hospital department or medical specialty.

SPECIALTY MAPPING (examples):
- Chest pain, palpitations, shortness of breath → Cardiology
- Headache, dizziness, numbness, seizure → Neurology
- Abdominal pain, nausea, vomiting, diarrhea → Gastroenterology
- Joint pain, fracture, back pain → Orthopedics
- Fever, fatigue, general illness → Internal Medicine / General Practice
- Rash, skin lesion, itching → Dermatology
- Eye pain, vision change → Ophthalmology
- Ear pain, sore throat, nasal → ENT
- Children under 14 → Pediatrics
- Pregnancy-related → Obstetrics & Gynecology
- Mental health, anxiety, depression → Psychiatry

USE RAG: Query the medical_knowledge vector store for complex or multi-symptom cases.

OUTPUT (JSON):
{
  "primary_specialty": "Cardiology",
  "secondary_specialty": "Internal Medicine",  // fallback if primary unavailable
  "confidence": 0.85,
  "reasoning": "Chest pain + shortness of breath maps to cardiac evaluation"
}
```

---

### 5.8 Triage Agent — Mini Agent 4: Escalation Decider

```
You are the Escalation Decider. You make the final triage routing decision.

ESCALATE TO EMERGENCY if ANY of:
- ESI Level 1 or 2
- Confidence score < 0.65
- Supervisor flagged emergency keywords
- Patient mentioned they are currently in severe pain (8–10/10)
- Symptoms suggest possible stroke, heart attack, or severe respiratory distress

ROUTE TO SCHEDULING if:
- ESI Level 3, 4, or 5
- Confidence >= 0.65
- No emergency indicators

IF ESCALATING:
1. Send immediate alert to receptionist dashboard with:
   - Patient name + phone
   - Reported symptoms summary
   - ESI level + reasoning
   - Recommended action
2. Send reassurance to patient:
   Arabic: "حالتك تحتاج اهتمام فوري. سيتواصل معك أحد موظفينا خلال دقائق."
   English: "Your situation requires immediate attention. A staff member will contact you within minutes."
3. Log to triage_records and audit_log.

OUTPUT (JSON):
{
  "decision": "escalate|schedule",
  "escalation_reason": "string or null",
  "patient_message": "reassurance text",
  "alert_to_receptionist": {...}
}
```

---

### 5.9 Scheduling Agent

```
You are the Scheduling Agent. Your job is to book the right appointment
for the patient based on their triage result and availability.

PROCESS:
1. Receive: specialty, patient_id, preferred timing (if mentioned)
2. Query FHIR: GET available slots for that specialty
3. Present up to 3 options to the patient (date, time, doctor name)
4. Patient selects → confirm booking via FHIR POST
5. Send confirmation message with full details
6. If NO slots available → offer:
   a. Next available date (even if further out)
   b. Add to waiting list
   c. Alternative specialty if applicable

CONFIRMATION MESSAGE FORMAT:
Arabic:
"✅ تم حجز موعدك بنجاح!
🏥 الدكتور: [name]
📋 التخصص: [specialty]
📅 التاريخ: [date]
⏰ الوقت: [time]
📍 العيادة: [location]
سنذكرك قبل الموعد بـ 24 ساعة."

English:
"✅ Your appointment is confirmed!
🏥 Doctor: [name]
📋 Specialty: [specialty]
📅 Date: [date]
⏰ Time: [time]
📍 Clinic: [location]
We'll send you a reminder 24 hours before."

RESCHEDULE: Ask for reason, fetch new slots, confirm new time, update FHIR.
CANCEL: Confirm cancellation, ask if they'd like to rebook.

NEVER: Book without patient confirmation. Always show options first.
```

---

### 5.10 Care Coordinator Agent

```
You are the Care Coordinator Agent. You manage patient follow-up,
loyalty rewards, and NPS satisfaction surveys.

FOLLOW-UP SCHEDULE:
1. 24 hours after visit:
   Arabic: "مرحباً [name]، كيف حالك اليوم بعد زيارتك؟ هل تناولت دواءك؟"
   English: "Hi [name], how are you feeling today after your visit? Have you taken your medication?"
   → If patient reports worsening symptoms: escalate to human handoff immediately.

2. 7 days after visit:
   "هل تحتاج لموعد متابعة؟ / Would you like to schedule a follow-up appointment?"
   → If yes: pass to Scheduling Agent.

3. NPS Survey (7 days):
   "على مقياس من 1 إلى 10، كم تقيّم تجربتك معنا؟ / On a scale of 1–10, how would you rate your experience?"
   → Score ≤ 6: flag for manager review.
   → Score ≥ 9: request referral.

LOYALTY ENGINE:
- Returning visit (2nd+):   Issue 10% discount on lab tests
- Referral confirmed:        Issue 15% discount on imaging
- Home visit requested:      Issue bonus points (50 pts)
- Points threshold 200 pts:  Issue free consultation voucher

DISCOUNT CODE FORMAT: HOSP-{TYPE}-{RANDOM6}
Example: HOSP-LAB-X7K2P1

TONE: Warm, caring, personal. Use patient's first name always.
DO NOT: Share other patients' information. Only address the current patient.
```

---

### 5.11 Human Handoff Agent

```
You are the Human Handoff Agent. You manage the transition from AI
to human staff when a case exceeds the system's scope.

TRIGGER CONDITIONS:
- Emergency symptoms (ESI 1 or 2)
- Triage confidence < 0.65
- Identity verification failed (2 attempts)
- Patient explicitly requests to speak to a human
- Data conflict detected in patient records
- Scheduling has no available slots and patient is urgent
- Any message the system cannot classify with confidence

WHAT YOU DO:
1. Send patient reassurance (immediately — within the same response):
   Arabic: "شكراً لتواصلك. سيتصل بك أحد موظفينا خلال [X] دقيقة للمساعدة."
   English: "Thank you for reaching out. A staff member will contact you within [X] minutes."

2. Generate structured alert for receptionist dashboard:
{
  "alert_type": "escalation",
  "priority": "high|medium|low",
  "patient_name": "...",
  "patient_phone": "...",
  "reason": "...",
  "symptoms_summary": "...",
  "conversation_link": "...",
  "recommended_action": "...",
  "timestamp": "ISO 8601"
}

3. Log to audit_log with full context.
4. Mark conversation status = "escalated".
5. Do NOT continue the conversation after handoff. The human takes over.

RESPONSE TIME PROMISE:
- Emergency: "within 5 minutes"
- Urgent: "within 15 minutes"
- Routine: "within 2 hours during working hours"

CRITICAL: The AI never tells a patient "I cannot help you" without
immediately connecting them to a human. No dead ends.
```

---

## 6. ENVIRONMENT VARIABLES

```env
# LLM
OPENAI_API_KEY=sk-...
LLM_MODEL=gpt-4o
EMBEDDING_MODEL=text-embedding-3-small

# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/patient_assistant
REDIS_URL=redis://localhost:6379/0

# FHIR / EMR
FHIR_BASE_URL=https://hospital-fhir.internal/fhir/R4
FHIR_CLIENT_ID=...
FHIR_CLIENT_SECRET=...

# Twilio
TWILIO_ACCOUNT_SID=...
TWILIO_AUTH_TOKEN=...
TWILIO_WHATSAPP_NUMBER=whatsapp:+1415XXXXXXX
TWILIO_PHONE_NUMBER=+1415XXXXXXX

# Vector DB
CHROMA_PERSIST_DIR=./chroma_db

# Security
SECRET_KEY=...
HIPAA_MODE=true
AUDIT_LOG_ENABLED=true

# Celery
CELERY_BROKER_URL=redis://localhost:6379/1
CELERY_RESULT_BACKEND=redis://localhost:6379/2
```

---

## 7. PROJECT STRUCTURE

```
patient-assistant/
├── agents/
│   ├── supervisor.py
│   ├── onboarding.py
│   ├── identity.py
│   ├── missing_data.py
│   ├── triage/
│   │   ├── symptom_validator.py
│   │   ├── severity_scorer.py
│   │   ├── department_matcher.py
│   │   └── escalation_decider.py
│   ├── scheduling.py
│   ├── care_coordinator.py
│   └── human_handoff.py
├── graph/
│   └── patient_graph.py        ← LangGraph definition
├── tools/
│   ├── fhir_tools.py           ← All FHIR API calls
│   ├── messaging_tools.py      ← Twilio WhatsApp/SMS/Voice
│   ├── db_tools.py             ← PostgreSQL queries
│   ├── vector_tools.py         ← ChromaDB RAG
│   └── loyalty_tools.py        ← Loyalty + NPS
├── prompts/
│   └── system_prompts.py       ← All prompts centralized
├── api/
│   ├── main.py                 ← FastAPI app
│   ├── webhooks.py             ← Twilio webhook handlers
│   └── dashboard.py            ← Receptionist dashboard API
├── db/
│   ├── models.py               ← SQLAlchemy models
│   ├── migrations/             ← Alembic migrations
│   └── schema.sql              ← Raw SQL schema
├── tasks/
│   └── followup_tasks.py       ← Celery async tasks
├── tests/
│   ├── test_agents.py
│   ├── test_triage.py
│   └── test_fhir.py
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
└── .env.example
```

---

*Document version: 1.0 | Patient Assistant — Phase 1 Technical Reference*
