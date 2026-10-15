# Secure AI Agent Governance on Google Cloud

Hands-on implementation of a governed AI agent architecture using Google Cloud Agent Runtime, Agent Identity, Agent Registry, Agent Gateway, Identity-Aware Proxy (IAP), Model Armor, MCP, A2A, Cloud Run, Vertex AI, and Google ADK.

## Project Overview

This repository documents a completed secure-agent governance lab focused on workload identity, on-behalf-of user authorization, least-privilege egress, content inspection, tool authorization, and end-to-end auditability.

The implementation models a clinical assistant that:

- runs on Google Cloud Agent Runtime
- uses a dedicated SPIFFE-based Agent Identity
- discovers governed services through Agent Registry
- routes governed outbound traffic through Agent Gateway
- uses IAP for egress authorization
- uses Model Armor for prompt-injection and response screening
- calls a private Patient Records MCP service
- delegates guideline questions to a separate A2A specialist
- preserves clinician identity separately from agent workload identity

## Architecture

```text
Clinician
   |
   | Hospital JWT
   v
Clinical Assistant
Agent Runtime
   |
   | Agent Identity / SPIFFE
   v
Agent Gateway
   |
   +--> IAP REQUEST_AUTHZ
   |      - default deny
   |      - read-only MCP policy
   |      - destination-level egress control
   |
   +--> Model Armor CONTENT_AUTHZ
          - direct prompt injection screening
          - indirect prompt injection screening
          - synthetic PHI / MRN response screening
   |
   +--> Private Patient Records MCP on Cloud Run
   |
   +--> Guideline Agent over A2A
   |
   +--> Required Google platform endpoints
```

See [docs/architecture.md](docs/architecture.md) for the full design.

## Security Controls Implemented

### 1. Per-agent workload identity

The clinical assistant was deployed with a dedicated SPIFFE-style Agent Identity tied to the specific Agent Runtime deployment.

This identity was used as the policy subject for Cloud Run invocation, Agent Gateway egress authorization, and Google platform API access.

### 2. Private MCP backend

The Patient Records MCP service remained private on Cloud Run. Direct anonymous access and direct use of a clinician token were rejected. The service required the agent's own Google workload identity to reach the backend.

### 3. On-behalf-of clinician authorization

Two identities were intentionally kept separate:

- Agent Identity: proves which deployed agent is making the request
- Clinician JWT: proves which clinician is acting through the agent

This allowed the backend to enforce patient-level care-team isolation while the platform enforced workload-level access.

### 4. Read-only MCP egress policy

IAP egress authorization allowed only approved read operations:

- whoami
- list_my_patients
- get_patient_summary
- get_recent_labs
- get_clinical_notes

The write operation `add_progress_note` was denied at the gateway before reaching the backend.

### 5. Direct prompt injection protection

A direct instruction-override attack was detected and blocked by Model Armor before normal model processing.

### 6. Indirect prompt injection protection

A synthetic referral document containing embedded malicious instructions was returned by the MCP backend. Model Armor inspected the response at the gateway, detected the prompt-injection payload with high confidence, and blocked it before it reached the agent.

### 7. Synthetic PHI / MRN protection

A strict response template used a custom Sensitive Data Protection detector for a synthetic MRN pattern. A patient summary containing the synthetic MRN was blocked while a benign labs response was allowed.

### 8. End-to-end auditability

Gateway logs exposed per-policy decisions, including:

- IAP allowed + Model Armor allowed
- IAP allowed + Model Armor denied
- IAP denied before downstream processing

## Acceptance Test Results

| Scenario | Result | Enforcing control |
|---|---|---|
| Clinician reads own patient panel | PASS | MCP care-team authorization |
| Clinician requests another clinician's patient | BLOCKED | MCP row-level authorization |
| Direct prompt injection | BLOCKED | Model Armor |
| Indirect prompt injection in tool response | BLOCKED | Model Armor gateway CONTENT_AUTHZ |
| Unauthorized write operation | BLOCKED | IAP read-only CEL policy |
| End-to-end identity check | PASS | Agent Identity + clinician token passthrough |
| Gateway audit evidence | PASS | Agent Gateway logs |

Final deployed-stack test result:

```text
4 passed, 4 skipped
```

The skipped cases were gateway scenarios validated separately through live gateway evidence rather than asserted by the automated test suite.

## Technologies

- Google Cloud Agent Runtime
- Agent Identity / SPIFFE
- Agent Registry
- Agent Gateway
- Identity-Aware Proxy
- Model Armor
- Model Context Protocol
- Agent2Agent protocol
- Cloud Run
- Vertex AI / Gemini
- Google ADK
- Cloud Logging
- Python
- IAM / CEL

## Repository Contents

- [docs/architecture.md](docs/architecture.md) — architecture and trust boundaries
- [docs/implementation.md](docs/implementation.md) — deployment sequence and implementation notes
- [docs/security-controls.md](docs/security-controls.md) — security-control breakdown
- [docs/testing-results.md](docs/testing-results.md) — adversarial and acceptance-test evidence

## Security and Privacy Notes

This repository intentionally excludes access tokens, JWT values, service-account keys, billing identifiers, Cloud Shell credentials, verification codes, live secrets, and environment files.

Deployment-specific identifiers are omitted or generalized where they are not necessary to explain the design.

## Attribution

This portfolio project is based on a hands-on Google Cloud agent governance workshop and accompanying lab materials.

This repository documents my own deployment, configuration, troubleshooting, testing, validation, and security analysis. Original workshop materials and source code remain the property of their respective authors.

## Author

Sam Etemad  
IT Infrastructure, Cloud & Security  
https://sametemad.ca
