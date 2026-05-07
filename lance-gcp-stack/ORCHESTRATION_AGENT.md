# 🧠 Orchestration Agent (`orchestration_agent/`)

## 1. Overview and Role
The **Orchestration Agent** serves as the central brain and routing hub of the Enterprise AI Platform. Rather than performing every task itself, it operates using a "Reasoning and Acting" (ReAct) paradigm to decompose user intents and delegate them to highly specialized tools and sub-agents. 

Its primary role is to solve the **N×M integration challenge**: seamlessly connecting diverse foundational models (Gemini) to disparate data sources, enterprise APIs, and stateless frontend applications, all while enforcing strict security perimeters.

---

## 2. Integration Ecosystem & Architecture

The Orchestration Agent sits at the intersection of various technologies, acting as the traffic cop for the enterprise architecture.

### A. Core AI & Data Integrations
*   **Gemini & LlamaIndex:** The orchestrator utilizes **Gemini 1.5 Pro** as its primary reasoning engine. **LlamaIndex** acts as the structural framework, equipping the agent with the necessary abstractions to execute "Tool Calls" and process complex semantic retrievals.
*   **LanceDB:** The primary non-parametric memory store. The orchestrator queries LanceDB for ultra-low latency semantic lookups and vector-based semantic caching.
*   **Cocoindex & Enterprise Data (Snowflake, BigQuery, PySpark):** The orchestrator triggers Cocoindex declarative ETL pipelines to sync data from Snowflake (ERP data), BigQuery (HTAP workloads), PySpark (dataset streams and large batches), and GCS Buckets (raw unstructured text/media).
*   **Web UI Frontend Apps:** The orchestrator serves as the backend intelligence for stateless React or Vanilla JS applications, processing requests validated by tiered IAM and Firebase Auth.

### B. FastAPI Entry Point
The Orchestration Agent interacts with the frontend via a robust **FastAPI** service, which handles authentication and streaming responses.

```python
from fastapi import FastAPI, Depends, HTTPException
from pydantic import BaseModel
from orchestration_agent.agent import orchestrator_agent

app = FastAPI(title="AI Orchestrator API")

class ChatRequest(BaseModel):
    query: str
    user_id: str

@app.post("/v1/chat")
async def chat_endpoint(request: ChatRequest):
    """Entry point for Web UI Frontend Apps."""
    # The agent processes the query, decides which tools to use, and returns a response
    try:
        response = orchestrator_agent.chat(request.query)
        return {"response": str(response), "sources": response.source_nodes}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### C. Apigee X & MCP Server Interaction
To securely traverse the network and interact with disparate data types (images, audio, blobs, structured databases), the Orchestrator does not write custom API integrations. Instead, it relies on the **Model Context Protocol (MCP)** and **Apigee X**.

*   **Apigee X:** Acts as the secure API gateway. Any tool the orchestrator uses to access external databases (like Spanner inventory) must pass through Apigee X for OAuth validation and Spike Arrest policies.
*   **MCP Server:** The orchestrator communicates with an MCP Server via JSON-RPC. The MCP server exposes standardized tools for reading disparate sources. If the user asks the orchestrator to "summarize the architecture diagram," the orchestrator uses an MCP tool to fetch the raw image blob from GCS, passes it to a Vision model, and returns the result.

---

## 3. Supporting Agents & Delegation Control

The Orchestrator Agent is supported by a team of specialized sub-agents. It controls them by passing well-defined inputs and waiting for structured outputs.

1.  **The Data Ingestion Agent:** 
    *   *Role:* The dedicated ETL workhorse.
    *   *Control:* The orchestrator triggers this agent when a user explicitly asks to sync a system or if a knowledge gap is detected. The Ingestion Agent, in turn, manages its own sub-agents (Vision/OCR, Transcription, and Data QA) to process raw blobs and videos into vectorizable text.
2.  **Semantic Search / RAG Manager:**
    *   *Role:* Executes the "Full-Link" retrieval pipeline.
    *   *Control:* When the orchestrator detects a request for factual documentation, it delegates the query to this manager, which utilizes LanceDB (Dense) and Vertex Search (Sparse/Hybrid) merged via Reciprocal Rank Fusion (RRF).
3.  **Security & Redaction Agent (Model Armor/DLP):**
    *   *Role:* Acts as a mandatory interceptor.
    *   *Control:* The orchestrator routes all incoming Web UI queries through this agent first to redact PII (replacing names with `[PERSON_NAME]`) and filter prompt injections before the LLM processes the text.

---

## 4. CRAG Best Practices & Adaptive Routing

To ensure high-fidelity, hallucination-resistant outputs, the Orchestration Agent implements **Corrective Retrieval Augmented Generation (CRAG)** strategies drawn from the `reference.md` enterprise guidelines.

### Adaptive Routing
The Orchestrator employs a Tri-Route Dispatch system before executing any RAG pipeline:
*   **Simple Route:** For direct factual queries, it routes directly to LanceDB.
*   **Complex Route:** For ambiguous technical queries, it utilizes **Hypothetical Document Embeddings (HyDE)**. The orchestrator asks Gemini to generate a "fake/hypothetical" answer first, and uses that generated text to search the vector database, drastically improving recall for lexically thin queries.
*   **External Route:** If the data is out-of-domain or requires real-time facts, it skips internal databases entirely and routes to a web search tool (e.g., Tavily).

### CRAG Evaluator Logic
When the Orchestrator delegates a query to the Semantic Search agent, it enforces a CRAG Evaluation gate before sending the final answer to the Web UI:

1.  **Correct (High Confidence):** If the retrieved context perfectly matches the query, the orchestrator performs *Knowledge Refinement*—compressing the context to remove noisy sentences, saving tokens and improving precision.
2.  **Ambiguous (Medium Confidence):** If the internal documentation is sparse, the orchestrator merges the refined internal data with an external web search to construct a comprehensive response.
3.  **Incorrect (Low Confidence):** If the vector search returns irrelevant data, the orchestrator *discards* the internal context entirely. It autonomously triggers the External Route to prevent the LLM from hallucinating an answer based on bad data.

By operating as an orchestrator rather than a monolithic script, this agent ensures the AI Platform remains modular, secure, and highly deterministic.
