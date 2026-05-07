# 🌐 AI Platform: Networking & Infrastructure Architecture

In an industrialized enterprise architecture like the **AI Platform**, networking must be treated as a core security control, not just a plumbing exercise. When dealing with Large Language Models (LLMs), sensitive vector data, and autonomous Model Context Protocol (MCP) servers, traditional perimeter defense is insufficient. 

To securely connect the **Stateless Web UI**, **Apigee X**, **Cloud Run / GKE**, and the **Orchestrator/Ingestion/MCP Agents**, the architectural standard relies on **Zero-Trust Networking, Private Service Connect (PSC), and VPC Service Controls (VPC-SC)**.

---

## 1. The "Golden Path" Traffic Flow
Here is how a user request (e.g., submitting a chat query via the React/Vue.js frontend) securely flows through the architecture:

```text
[📱 Web UI Frontend (React/Vue) + Firebase Stateless Auth] 
       │ (HTTPS / TLS 1.3 / JWT Token)
       ▼
[🛡️ Global External ALB] ──[ Cloud Armor WAF & Model Armor ]
       │ (Private Service Connect - NEG)
       ▼
[🐝 Apigee X Tenant VPC] ──(OAuth Validation, Spike Arrest, DLP Redaction)
       │ (Southbound Private Service Connect)
       ▼
[========== CUSTOMER ENTERPRISE VPC (VPC-SC Cryptographic Perimeter) ==========]
       │
[🕸️ Internal Application Load Balancer] (Zonal NEGs)
       │
       ├──> 🧠 [Orchestration Agent (FastAPI / Cloud Run)]
       │         │ (Local stdio or remote SSE via JSON-RPC)
       │         ├──> 🔌 [FastMCP Server] (Standardized Tools)
       │         │         ▼
       │         │    [External APIs: Snowflake, BigQuery, Spanner]
       │         │
       │         ├──> 🔎 [Semantic Search / LanceDB on GCS & Vertex Search]
       │         │
       │         └──> 📥 [Ingestion Agent (FastAPI / Cloud Run)]
       │                   ▼
       │              [Cocoindex ETL Pipeline] <── (Pub/Sub Triggers)
       ▼
[☁️ Google Backend APIs: Vertex AI / Gemini API, Cloud Storage]
```

---

## 2. Edge & Ingress Networking (The Front Door)
The entry point must absorb attacks, inspect payloads, and route traffic privately to the API Gateway.

*   **Global External Application Load Balancers (ALB):** Terminate TLS at the edge using Google’s global Anycast IPs. The backend microservices (Orchestrator and Ingestion agents) are never exposed directly to the internet.
*   **Firebase Stateless Auth & Tiered IAM:** The frontend verifies users (Tier 1 Super Admin, Tier 2 Customer Admin, Tier 3 Premium User) and passes the stateless JWT to the backend.
*   **Attach Cloud Armor & Model Armor:** Apply Cloud Armor to block DDoS and OWASP Top 10 attacks. Integrate **Model Armor** at the gateway to sanitize incoming prompts for injection attacks or malicious intent.

## 3. API Gateway Management (Apigee X Integration)
Apigee X acts as the secure, governed choke-point between the frontend and the internal AI agents.

*   **Southbound Private Service Connect (PSC):** Expose your internal Cloud Run or GKE deployments as a PSC Service Attachment. Apigee X connects to this endpoint securely without complex VPC peering.
*   **Offload Security and DLP:** Use Apigee X to enforce Spike Arrests (preventing token exhaustion/denial-of-wallet) and execute Data Loss Prevention (DLP) redaction (e.g., masking PII with `[PERSON_NAME]`) before the payload hits the Orchestrator.
*   **Streaming Optimization:** The Orchestrator relies heavily on streaming LLM responses. Configure Apigee X proxies to **disable payload buffering** for streaming endpoints, preventing the network from holding the AI's generated response until completion.

## 4. Agent Infrastructure (Cloud Run / GKE)
The microservices environment hosts the Orchestrator Agent, Ingestion Agent, and the FastMCP Server.

*   **Orchestration Agent (ReAct Router):** A FastAPI service acting as the central intelligence hub, making routing decisions to Semantic Search, Ingestion, or MCP.
*   **Ingestion Agent (Asynchronous ETL):** A FastAPI service that also listens to GCP Pub/Sub topics. When a new file lands in the raw GCS bucket, Pub/Sub triggers the Ingestion Agent to execute the Cocoindex ETL pipeline and update the LanceDB vector store.
*   **MCP Server (N×M Integration):** The Model Context Protocol server operates within the internal network. If handling sensitive data specifically for the Orchestrator, it runs via standard input/output (`stdio`) locally. For shared enterprise actions, it runs as a standalone service communicating via Server-Sent Events (SSE).

## 5. Backend Connectivity & Exfiltration Prevention
The agents must reach external databases (Snowflake, BigQuery), vector stores (LanceDB), and LLM APIs (Gemini) securely.

*   **Workload Identity Federation:** Never use long-lived Service Account Keys. The Orchestrator and Ingestion agents inherit short-lived, auto-rotating tokens with least privilege (e.g., `roles/storage.objectAdmin` for LanceDB access).
*   **Private Google Access (PGA):** Ensure the subnets hosting the agents have PGA enabled. This forces the agents' API calls to Gemini or Vertex Search to route strictly through Google's internal network backbone.
*   **VPC Service Controls (VPC-SC):** This is the ultimate cryptographic perimeter. Wrap the entire project—Apigee X, Cloud Run/GKE, Cloud Storage (LanceDB data), BigQuery, and Vertex AI—inside a VPC-SC perimeter. This prevents unauthorized exfiltration; even a compromised Workload Identity token cannot pull data out of the defined corporate perimeter.

---

## 6. Implementation Rollout Mapping
To successfully build this zero-trust architecture, the networking components are deployed across the following phases of the **AI Platform Build Guide**:

*   **[Phase 1](PHASE_1_GUIDE.md):** Establishes the core VPC, Subnets with Private Google Access (PGA), and the VPC Service Controls (VPC-SC) perimeter to secure GCS and Vertex AI.
*   **[Phase 3](PHASE_3_GUIDE.md):** Configures Apigee X API management, establishing the Southbound Private Service Connect (PSC) NEGs and disabling streaming payload buffering.
*   **[Phase 5](PHASE_5_GUIDE.md):** Deploys the internal microservices to Cloud Run behind the Internal Application Load Balancer (ILB), and configures the Global External ALB with Cloud Armor.