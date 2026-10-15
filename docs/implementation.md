# Implementation

## Deployment Sequence

1. Prepared the Google Cloud project and enabled required APIs
2. Deployed a mock hospital identity provider
3. Deployed the private Patient Records MCP service
4. Configured Model Armor templates
5. Reviewed and validated the A2A Agent Card contract
6. Registered MCP and required platform endpoints in Agent Registry
7. Created Agent Gateway with IAP authorization in dry-run mode
8. Deployed the A2A guideline specialist and clinical assistant
9. Granted egress and switched IAP from dry run to enforcement
10. Attached Model Armor to Agent Gateway as CONTENT_AUTHZ
11. Ran the final acceptance test suite

## Key Implementation Details

### Agent Identity

The clinical assistant was deployed with a dedicated Agent Identity rather than relying on a shared runtime service account. This allowed policies to bind to one specific deployed agent.

### Registry-driven discovery

The MCP toolset was resolved through Agent Registry rather than from a hard-coded service URL. The same registry entry also served as a governed destination for Gateway and IAM policy attachment.

### Gateway rollout

The IAP extension was first deployed in dry-run mode. Egress grants were applied before enforcement was enabled, reducing the risk of cutting off the agent during policy rollout.

### Read-only enforcement

The MCP policy allowed only approved read operations and necessary lifecycle methods such as initialize and tools/list. The write tool remained blocked at the gateway.

### Model Armor integration

Model Armor was validated first with standalone sanitize calls. It was then attached inside the agent through ADK callbacks and at the gateway for content authorization.

### Troubleshooting performed

The implementation required hands-on troubleshooting, including:

- completing the Python environment setup with uv
- restoring gcloud authentication
- resolving a quota-project issue for Model Armor API calls
- correcting Model Armor REST request headers
- handling IAM propagation delay for Agent Gateway permissions
- resolving generated Agent Registry resource IDs instead of assuming service names
- validating response-side blocks through raw Model Armor logs when helper output lagged

These troubleshooting steps were part of the deployment and validation work.
