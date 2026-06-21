# Hospital AI Patient Agent

An agent-only hospital patient assistant built with Python and the OpenAI Agents SDK.

This repository does not contain a web application, REST API, FastAPI backend, or database. Its scope is limited to implementing the agents, their tools and handoffs, then evaluating their behavior locally.

> This project is an educational prototype. It must not diagnose patients, replace clinicians, or connect to real patients or emergency services without clinical, legal, privacy, and security review.

## Project status

The project currently contains the file structure only. Agent implementation and tests will be added gradually.

## Planned agents

| Agent | Responsibility |
|---|---|
| Orchestrator | Understands the request and delegates it to the correct specialist agent. |
| Patient Intelligence | Retrieves or simulates the patient context required by other agents. |
| Clinical Triage | Evaluates symptoms, detects uncertainty, and recommends a safe next step. |
| Operations | Handles departments, appointment availability, and booking tasks. |
| Emergency | Handles red-flag cases and returns the required escalation instructions. |

## Planned workflow

```text
Patient input
     |
     v
Orchestrator
     |
     +--> Patient Intelligence
     |
     +--> Clinical Triage
     |         |
     |         +--> Emergency, when a red flag is detected
     |         |
     |         +--> Operations, when an appointment is needed
     |
     +--> Final response
```

The Orchestrator is the entry point. Specialist agents receive only the context required for their tasks. Safety-critical uncertainty must produce a human-escalation result instead of an invented answer.

## Project structure

```text
hospital-ai-patient-agent_MH74/
|-- .env
|-- .env.example
|-- pyproject.toml
|-- README.md
|-- src/
|   |-- main.py
|   |-- agents/
|   |   |-- orchestrator/
|   |   |   |-- agent.py
|   |   |   |-- prompts/prompt.md
|   |   |   `-- tools/
|   |   |-- patient_intelligence/
|   |   |   |-- agent.py
|   |   |   |-- prompts/prompt.md
|   |   |   `-- tools/patient_tools.py
|   |   |-- clinical_triage/
|   |   |   |-- agent.py
|   |   |   |-- prompts/prompt.md
|   |   |   `-- tools/triage_tools.py
|   |   |-- operations/
|   |   |   |-- agent.py
|   |   |   |-- prompts/prompt.md
|   |   |   `-- tools/appointment_tools.py
|   |   `-- emergency/
|   |       |-- agent.py
|   |       |-- prompts/prompt.md
|   |       `-- tools/emergency_tools.py
|   |-- config/
|   |-- graph/
|   |-- schemas/
|   `-- tools/
|       `-- knowledge_tools.py
|-- data/
|   `-- evals/
`-- tests/
    |-- unit/
    |-- integration/
    |-- evals/
    `-- safety/
```

### Directory responsibilities

- `src/agents/<agent>/agent.py`: The OpenAI Agents SDK definition for one agent.
- `src/agents/<agent>/prompts`: Instructions owned by that agent.
- `src/agents/<agent>/tools`: Tools used only by that agent.
- `src/tools`: Shared tools that can be used by multiple agents.
- `src/graph`: Conversation state, routing, and handoff coordination.
- `src/schemas`: Structured input and output models.
- `src/config`: Environment and model configuration.
- `tests/unit`: Deterministic tool and schema tests.
- `tests/integration`: Multi-agent routing and handoff tests.
- `tests/evals`: Expected agent behavior and quality evaluations.
- `tests/safety`: Emergency detection, unsafe-output, and prompt-injection tests.
- `data/evals`: Evaluation cases and expected outcomes.

## Requirements

- Python 3.12 or later
- An OpenAI API key
- A Python virtual environment

## Local setup

Create and activate a virtual environment:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

Install the initial development dependencies:

```powershell
pip install openai-agents python-dotenv pydantic pytest pytest-asyncio
```

Add your API key to `.env`:

```env
OPENAI_API_KEY=your_openai_api_key
OPENAI_MODEL=gpt-5.4-mini
```

Never commit `.env` or a real API key to source control.

## Running the agent

After `src/main.py` is implemented:

```powershell
python -m src.main
```

## Running tests

Run all tests:

```powershell
pytest
```

Run one test category:

```powershell
pytest tests/unit
pytest tests/integration
pytest tests/evals
pytest tests/safety
```

## Recommended implementation order

1. Add settings and environment loading.
2. Define structured schemas for triage and emergency results.
3. Implement deterministic tools with mocked hospital data.
4. Implement the specialist agents.
5. Implement the Orchestrator and agent handoffs.
6. Add a local command-line entry point in `src/main.py`.
7. Add unit and integration tests.
8. Build safety and evaluation datasets before using realistic scenarios.

## Testing goals

The final evaluation should verify behavior rather than exact wording:

- Correct agent selection and handoff.
- Valid structured outputs.
- Reliable red-flag detection.
- Safe handling of low confidence and missing patient information.
- No fabricated appointments, patient records, or emergency actions.
- Resistance to instructions that attempt to bypass medical safety rules.
- No live API calls in unit tests; use mocks or stubs.

## Official OpenAI Agents SDK documentation

- [OpenAI Agents SDK for Python](https://openai.github.io/openai-agents-python/)
- [Quickstart](https://openai.github.io/openai-agents-python/quickstart/)
- [Agents](https://openai.github.io/openai-agents-python/agents/)
- [Tools](https://openai.github.io/openai-agents-python/tools/)
- [Handoffs](https://openai.github.io/openai-agents-python/handoffs/)
- [Tracing](https://openai.github.io/openai-agents-python/tracing/)
