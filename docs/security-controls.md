# Security Controls

## Control Matrix

| Control | Enforcement point | Purpose |
|---|---|---|
| Agent Identity | Agent Runtime | Identifies the exact deployed workload |
| Cloud Run IAM | MCP backend ingress | Restricts service invocation |
| Clinician JWT | MCP application layer | Identifies the human user |
| Care-team authorization | MCP backend | Restricts patient-level records |
| IAP egress policy | Agent Gateway | Restricts destinations and MCP tools |
| Model Armor plugin | Agent Runtime | Screens user prompts and agent content |
| Model Armor CONTENT_AUTHZ | Agent Gateway | Screens governed traffic outside agent code |
| Agent Registry | Agent Platform | Discovery and destination allowlisting |
| Gateway logs | Cloud Logging | Records per-policy authorization outcomes |

## Direct Prompt Injection

A hostile instruction asking the model to ignore previous rules and expose all records was detected before normal model processing.

Result: BLOCKED.

## Indirect Prompt Injection

A synthetic external referral document embedded an instruction intended to override the assistant's behavior.

The backend was authorized to return the document, but the gateway's Model Armor extension inspected the response and blocked it.

Observed evidence included:

```text
filterMatchState: MATCH_FOUND
pi_and_jailbreak: MATCH_FOUND
confidenceLevel: HIGH
sanitizationVerdict: MODEL_ARMOR_SANITIZATION_VERDICT_BLOCK
```

## PHI / MRN Response Control

A strict Model Armor response profile referenced a custom Sensitive Data Protection rule for a synthetic MRN pattern.

The patient-summary response containing the synthetic record number was blocked. A labs response without the MRN remained allowed.

## Write Prevention

The assistant was intentionally granted read-only access to the MCP service.

The write tool was denied by the gateway's IAP authorization policy.

Observed gateway outcome:

```text
add_progress_note -> DENIED
iap=DENIED
```

Backend logs showed no matching write execution.

## Identity Separation

The architecture intentionally separates workload identity from human identity.

Workload identity establishes that the request came from the deployed agent.

Human identity establishes which clinician was acting through the agent.

This separation prevents a valid workload from automatically gaining unrestricted access to every patient record.
