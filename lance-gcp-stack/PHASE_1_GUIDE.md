# Phase 1: Foundation & Repository Setup

This guide provides a comprehensive, step-by-step walkthrough for executing Phase 1 of the Enterprise AI Platform rollout. This phase establishes the core infrastructure, repository structure, and security foundation required for subsequent development.

## Step 1: Initialize the Monorepo

We will use a monorepo approach to manage all microservices, agents, and infrastructure as code in one place.

### 1.1 Scaffold the Directory Structure
Run the following commands in your terminal to generate the exact repository structure required:

```bash
# Create the root platform directory
mkdir ai-platform && cd ai-platform

# 1. CI/CD & Governance
mkdir -p .github/workflows
touch .github/CODEOWNERS

# 2. Multi-Agent System Workspace
mkdir -p agents/orchestrator_agent agents/ingestion_agent
touch agents/orchestrator_agent/__init__.py agents/orchestrator_agent/agent.py agents/orchestrator_agent/router.py
touch agents/ingestion_agent/__init__.py agents/ingestion_agent/agent.py agents/ingestion_agent/pubsub_listener.py

# 3. APIGEE X API Management Layer
mkdir -p api_gateway/apigee_x_configs/proxies api_gateway/apigee_x_configs/policies
touch api_gateway/apigee_x_configs/openapi-spec.yaml

# 4. Model Context Protocol (MCP) Server
mkdir -p mcp_server/tools
touch mcp_server/__init__.py mcp_server/server.py
touch mcp_server/tools/spanner_inventory.py mcp_server/tools/enterprise_actions.py

# 5. ETL & Data Connectors
mkdir -p data_pipeline/cocoindex_etl data_pipeline/sources
touch data_pipeline/cocoindex_etl/__init__.py data_pipeline/cocoindex_etl/pipeline.py data_pipeline/cocoindex_etl/higress_splitter.py
touch data_pipeline/sources/bigquery_htap.py data_pipeline/sources/snowflake_erp.py data_pipeline/sources/gcs_buckets.py

# 6. LlamaIndex Semantic Search Manager
mkdir -p semantic_search
touch semantic_search/__init__.py semantic_search/llamaindex_manager.py semantic_search/hyde_generator.py semantic_search/rrf_merger.py semantic_search/crag_evaluator.py

# 7. Vector Database Destination
mkdir -p vector_database
touch vector_database/__init__.py vector_database/lancedb_client.py vector_database/schema.py

# 8. Hybrid Fallback Search
mkdir -p enterprise_search
touch enterprise_search/__init__.py enterprise_search/vertex_search_client.py

# 9. Foundation Models
mkdir -p models
touch models/__init__.py models/gemini_pro.py models/gemini_flash.py models/semantic_cache.py

# 10. Responsible AI Layer
mkdir -p security_and_governance
touch security_and_governance/model_armor.py security_and_governance/dlp_redactor.py

# 11. Root level files
touch pyproject.toml README.md
```

### 1.2 Initialize Version Control
```bash
git init
git add .
git commit -m "chore: initial commit of enterprise ai platform structure"
```

## Step 2: Configure GCP Environment

This step ensures your Google Cloud environment is properly configured to host the platform's infrastructure.

### 2.1 Set Project and Authentication
```bash
# Login to Google Cloud
gcloud auth login

# Set your target project ID (replace with your actual project ID)
export PROJECT_ID="your-enterprise-project-id"
gcloud config set project $PROJECT_ID
```

### 2.2 Enable Required APIs
Enable the necessary Google Cloud services for the platform:
```bash
gcloud services enable \
    compute.googleapis.com \
    storage-api.googleapis.com \
    storage-component.googleapis.com \
    aiplatform.googleapis.com \
    discoveryengine.googleapis.com \
    apigee.googleapis.com \
    bigquery.googleapis.com \
    run.googleapis.com \
    container.googleapis.com \
    accesscontextmanager.googleapis.com

### 2.3 Establish Networking Foundation (VPC-SC & PGA)
As mandated by the `NETWORKING_INFRASTRUCTURE.md`, we must establish a zero-trust network perimeter.

**1. Create Custom VPC and Subnets with Private Google Access:**
```bash
# Create the enterprise VPC
gcloud compute networks create enterprise-ai-vpc --subnet-mode=custom

# Create a subnet with Private Google Access enabled
gcloud compute networks subnets create ai-agent-subnet \
    --network=enterprise-ai-vpc \
    --range=10.0.0.0/24 \
    --region=us-central1 \
    --enable-private-ip-google-access
```

**2. Define the VPC Service Controls (VPC-SC) Perimeter:**
```bash
# Create the service perimeter to prevent data exfiltration
gcloud access-context-manager perimeters create ai_platform_perimeter \
    --title="Enterprise AI Platform Perimeter" \
    --resources="projects/$PROJECT_ID" \
    --restricted-services="storage.googleapis.com,aiplatform.googleapis.com,discoveryengine.googleapis.com" \
    --policy="YOUR_POLICY_NAME"
```
```

## Step 3: IAM & Least Privilege

We will establish discrete Service Accounts (SAs) to ensure each component only has the permissions it absolutely needs.

### 3.1 Create Service Accounts
```bash
# SA for the Data Ingestion Agent
gcloud iam service-accounts create ingestion-agent-sa \
    --description="Service account for ETL and data pipeline ingestion" \
    --display-name="Ingestion Agent SA"

# SA for the Orchestrator Agent
gcloud iam service-accounts create orchestrator-agent-sa \
    --description="Service account for the ReAct Orchestrator" \
    --display-name="Orchestrator Agent SA"

# SA for the MCP Server
gcloud iam service-accounts create mcp-server-sa \
    --description="Service account for MCP tool execution" \
    --display-name="MCP Server SA"
```

### 3.2 Assign Roles to Service Accounts

**Ingestion Agent (Needs access to GCS, BigQuery, and LanceDB storage):**
```bash
# Read access to raw data buckets
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:ingestion-agent-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/storage.objectViewer"

# Write access to the LanceDB vector bucket
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:ingestion-agent-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/storage.objectAdmin"

# BigQuery data viewer
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:ingestion-agent-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/bigquery.dataViewer"
```

**Orchestrator Agent (Needs access to Vertex AI for Gemini models):**
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:orchestrator-agent-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/aiplatform.user"
```

## Step 4: Dependency Management

We use `pyproject.toml` to manage our core dependencies in a standardized way.

### 4.1 Define `pyproject.toml`
Populate the `pyproject.toml` file at the root of the repository:

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "enterprise-ai-platform"
version = "0.1.0"
description = "Enterprise multi-agent AI platform with MCP and LanceDB"
requires-python = ">=3.10"
dependencies = [
    "llama-index-core>=0.10.0",
    "llama-index-llms-gemini",
    "llama-index-vector-stores-lancedb",
    "lancedb",
    "cocoindex",
    "mcp",
    "google-cloud-discoveryengine",
    "google-cloud-storage",
    "google-cloud-bigquery",
    "httpx",
    "fastapi",
    "uvicorn"
]

[project.optional-dependencies]
dev = [
    "pytest",
    "black",
    "flake8"
]
```

### 4.2 Establish Virtual Environment
Create an isolated environment and install the dependencies:
```bash
# Move into the platform directory if not already there
cd ai-platform

# Create the virtual environment
python3 -m venv .venv

# Activate the virtual environment (Linux/macOS)
source .venv/bin/activate
# For Windows: .venv\Scripts\activate

# Install dependencies
pip install --upgrade pip
pip install -e .[dev]
```

## Completion of Phase 1
You have now established the structural, security, and dependency foundations for the AI platform. 

**Deployment Checklist:**
- [ ] Directory structure matches architecture spec
- [ ] GCP Project initialized with required APIs enabled
- [ ] Service accounts created with least-privilege IAM roles
- [ ] Python virtual environment active with all core packages installed
