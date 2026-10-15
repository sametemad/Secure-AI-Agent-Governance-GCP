# Architecture

## Purpose

The goal of this implementation was to build an AI agent that could use enterprise tools without relying on prompts alone for security.

The design separates workload identity, clinician identity, network egress authorization, content inspection, and backend authorization into independent controls.

## Trust Boundaries

### User to agent

The frontend flow issues a clinician token from a mock hospital identity provider. The clinician token represents the human user and is passed through the governed workflow for application-level authorization.

### Agent Runtime

The clinical assistant runs in Agent Runtime with its own dedicated Agent Identity. The identity is represented as a SPIFFE-style principal associated with the deployed reasoning engine.

### Agent Gateway

Governed outbound traffic is routed through Agent Gateway. The gateway provides registry-based destination control, authorization callouts, policy enforcement, and policy-specific audit logging.

### IAP authorization

IAP evaluates whether the agent identity may reach a registered destination. For MCP, the policy also evaluates tool metadata and permits only an explicit read-only allowlist.

### Model Armor

Model Armor is attached in two places:

1. Inside the agent runtime through the ADK plugin lifecycle
2. At Agent Gateway as a CONTENT_AUTHZ service extension

The gateway placement provides response-side inspection outside normal agent logic.

### Private MCP backend

The Patient Records MCP service runs privately on Cloud Run.

The backend receives two distinct credentials:
- the agent's Google identity for service invocation
- the clinician JWT for patient-level authorization

### A2A specialist

The guideline specialist is deployed separately and discovered through an A2A Agent Card.

## Request Flow

```text
Clinician
  |
  | clinician JWT
  v
Clinical Assistant
  |
  | dedicated Agent Identity
  v
Agent Gateway
  |
  +--> IAP REQUEST_AUTHZ
  |
  +--> Model Armor CONTENT_AUTHZ
  |
  +--> Agent Registry destination
          |
          +--> MCP backend
          +--> A2A specialist
          +--> Google platform endpoints
```

## Security Design Principle

Each layer answers a different question:

- Agent Identity: which deployed workload is calling?
- IAP: may this workload reach this destination or tool?
- Clinician JWT: which human is acting?
- MCP backend: may this clinician access this patient?
- Model Armor: is the content safe to pass?
- Gateway logs: which control made the decision?
