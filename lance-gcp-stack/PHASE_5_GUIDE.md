# Phase 5: Security, UI Integration, and Go-Live

The final phase focuses on securing the multi-agent system, connecting the front-end user interface, containerizing the application for scale, and moving to production deployment.

## Step 1: Responsible AI Governance

Before opening the platform to users, we must enforce deterministic security using Google Cloud Data Loss Prevention (DLP) and prompt injection filtering.

### 1.1 Implement DLP Redaction
Populate `security_and_governance/dlp_redactor.py`:

```python
import os
from google.cloud import dlp_v2

def redact_pii(text_input: str) -> str:
    """Scans and masks PII before it enters the LLM context window."""
    dlp = dlp_v2.DlpServiceClient()
    project = os.getenv("PROJECT_ID")
    parent = f"projects/{project}/locations/global"

    # Define what to inspect (e.g., Email, Phone numbers, Credit Cards)
    inspect_config = {
        "info_types": [
            {"name": "EMAIL_ADDRESS"},
            {"name": "PHONE_NUMBER"},
            {"name": "CREDIT_CARD_NUMBER"}
        ]
    }

    # Define how to redact (Replace with a structural placeholder)
    deidentify_config = {
        "info_type_transformations": {
            "transformations": [
                {
                    "primitive_transformation": {
                        "replace_with_info_type_config": {}
                    }
                }
            ]
        }
    }

    item = {"value": text_input}
    
    try:
        response = dlp.deidentify_content(
            request={
                "parent": parent,
                "deidentify_config": deidentify_config,
                "inspect_config": inspect_config,
                "item": item,
            }
        )
        return response.item.value
    except Exception as e:
        print(f"DLP Redaction failed: {e}")
        # Fail closed: Do not return unredacted text if DLP fails
        return "[REDACTED_DUE_TO_SYSTEM_ERROR]"
```

### 1.2 Model Armor Integration
Update the orchestrator agent to wrap incoming queries in the DLP redactor and any prompt injection filters before passing the query to the Gemini models.

## Step 2: CI/CD and Containerization

The microservices (Orchestrator API, Ingestion Listener, MCP Server) must be containerized to run securely and independently on Google Cloud Run.

### 2.1 Create the Orchestrator Dockerfile
Create a `Dockerfile` at the root of the repository for the Orchestrator API:

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY pyproject.toml .
RUN pip install .

# Copy source code
COPY . /app

# Expose the API port (Assuming a FastAPI wrapper around the Orchestrator)
EXPOSE 8080

# Start the API Server
CMD ["uvicorn", "api_gateway.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

### 2.2 Define the Cloud Build Workflow
Create `.github/workflows/deploy.yaml` to trigger builds on commit:

```yaml
name: Deploy to Cloud Run

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Authenticate to Google Cloud
      uses: google-github-actions/auth@v1
      with:
        credentials_json: '${{ secrets.GCP_CREDENTIALS }}'
        
    - name: Set up Cloud SDK
      uses: google-github-actions/setup-gcloud@v1
      
    - name: Build and Push Docker image
      run: |
        gcloud builds submit --tag gcr.io/${{ secrets.PROJECT_ID }}/orchestrator-api
        
    - name: Deploy to Cloud Run
      run: |
        gcloud run deploy orchestrator-api \
          --image gcr.io/${{ secrets.PROJECT_ID }}/orchestrator-api \
          --platform managed \
          --region us-central1 \
          --ingress internal-and-cloud-load-balancing \
          --no-allow-unauthenticated
```

### 2.3 Load Balancer Topologies
To comply with `NETWORKING_INFRASTRUCTURE.md`, the Cloud Run service must sit behind a strict load balancing hierarchy.

**1. Internal Application Load Balancer (ILB):**
Create a regional ILB with a Serverless NEG (Zonal NEG) pointing to the Cloud Run service. Expose this ILB as a Private Service Connect (PSC) Service Attachment so Apigee X can reach it.

**2. Global External Application Load Balancer (ALB):**
Provision a Global External ALB to serve as the public front door. Attach **Cloud Armor** and **Model Armor** to this ALB, and configure its backend to route via PSC to your Apigee X tenant.

## Step 3: Frontend Connectivity

The frontend uses Tiered IAM and stateless Firebase Auth. The frontend will communicate with the deployed Orchestrator Cloud Run service.

### 3.1 Validate the API Connection
Ensure the React/Vanilla JS application points to the newly deployed Cloud Run URL:

```javascript
// Example Frontend API Call
async function sendQueryToOrchestrator(query, idToken) {
    const ORCHESTRATOR_URL = "https://orchestrator-api-xxx-uc.a.run.app";
    
    const response = await fetch(`${ORCHESTRATOR_URL}/chat`, {
        method: "POST",
        headers: {
            "Content-Type": "application/json",
            "Authorization": `Bearer ${idToken}` // Validated by API Gateway/Cloud Run
        },
        body: JSON.stringify({ query: query })
    });
    
    return await response.json();
}
```

## Step 4: End-to-End Validation & Go-Live

Before full release, simulate high concurrency and ensure cross-module integration.

### 4.1 UAT Checklist
- **Security Check:** Try prompt injections (e.g., "Ignore previous instructions"). Ensure `model_armor` blocks it.
- **Privacy Check:** Enter a fake credit card number in the chat. Ensure it returns as `[CREDIT_CARD_NUMBER]`.
- **Integration Check:** Trigger an ingestion event (upload a file to GCS) and verify it appears in a LanceDB search 30 seconds later.
- **IAM Check:** Ensure Tier 2 (Customer Admins) cannot access Tier 1 (Super Admin) MCP endpoints.

### 4.2 Production Turnover
Once the UAT is complete, merge the `develop` branch to `main`. Your GitHub Actions CI/CD pipeline will automatically build and deploy the production services.

## Completion of Phase 5
Congratulations! You have completed the infrastructure paving, agent orchestration, and security deployment for the Enterprise AI Platform.

**Deployment Checklist:**
- [ ] `dlp_redactor.py` injected into the prompt pipeline.
- [ ] Dockerfiles created for all microservices.
- [ ] GitHub Actions CI/CD pipeline implemented.
- [ ] Frontend successfully authenticating and querying the Orchestrator API.
- [ ] UAT Validation Complete.

*End of the Enterprise AI Platform Build Guide.*
