# 📥 Data Ingestion Agent (`ingestion_agent/`)

## 1. Overview and Role
The **Ingestion Agent** acts as the dedicated Extract, Transform, and Load (ETL) workhorse of the Enterprise AI Platform. While the Orchestrator Agent handles user queries and reasoning, the Ingestion Agent operates asynchronously to ensure that the "External Brain" (the non-parametric memory stored in vector databases like LanceDB) is continuously updated, structured, and securely ingested.

Its primary responsibilities include:
- **Connecting to Disparate Sources:** Securely reaching into ERP systems, HTAP databases, and blob storage to fetch raw data.
- **Handling Multi-Modal Data Types:** Processing not just text, but images, audio, video, and raw unstructured blobs.
- **Structural Preservation:** Utilizing tools like the **Higress-Native Splitter** to parse Abstract Syntax Trees (AST) and maintain document hierarchies (headers, code blocks, tables) rather than relying on arbitrary character-count chunks.
- **Continuous Hydration:** Listening for real-time events (via Pub/Sub or API calls) to keep the vector database fresh, preventing "knowledge cutoffs."

---

## 2. Architecture & Example Integration

The Ingestion Agent is decoupled from the main Orchestrator, running as an independent microservice. Below is an architectural outline of how it interacts with the broader enterprise ecosystem.

### A. FastAPI Entry Point
The Ingestion Agent exposes a `FastAPI` service. This provides an HTTP interface for webhooks and manual triggers from other systems.

```python
from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel
from ingestion_agent.pipeline import process_source

app = FastAPI(title="Ingestion Agent API")

class IngestionRequest(BaseModel):
    source_type: str  # e.g., "gcs_blob", "snowflake_erp", "apigee_mcp"
    uri: str
    incremental: bool = True

@app.post("/trigger-sync")
async def trigger_sync(request: IngestionRequest, background_tasks: BackgroundTasks):
    """Webhook to trigger a data ingestion job asynchronously."""
    background_tasks.add_task(process_source, request.source_type, request.uri, request.incremental)
    return {"status": "Ingestion job queued", "target": request.uri}
```

### B. Apigee X & MCP Server for Disparate Sources
The Ingestion Agent must securely traverse the enterprise network. It utilizes **Apigee X** as the security gateway and the **Model Context Protocol (MCP)** to standardize connections.

1. **Structured Databases (Snowflake, BigQuery):**
   The agent queries these through Apigee X proxies, which enforce OAuth and rate limiting. It converts structured rows into textual narratives (e.g., converting a row of inventory into a descriptive sentence) before embedding.
   
2. **Unstructured Data via MCP (PDFs, Images, Video, Audio):**
   To avoid writing custom API integrations for every data source, the agent leverages the MCP Server. If a new video lands in GCS, the MCP Server exposes a `fetch_and_transcribe_media` tool.

```python
# Conceptual interaction via MCP client inside the Ingestion Agent
async def fetch_data(uri: str):
    # The MCP Server abstracts the complexity of connecting to SharePoint, GCS, or an internal Wiki
    media_blob = await mcp_client.call_tool("fetch_blob", {"uri": uri})
    return media_blob
```

---

## 3. Supporting Agents

To handle complex multi-modal data, the Ingestion Agent delegates to highly specialized **Supporting Agents**:

* **Vision/OCR Agent:** When the Ingestion Agent receives an image or a scanned PDF, it passes the blob to the Vision Agent (powered by Gemini Pro Vision). This agent extracts text, interprets charts, and returns clean Markdown.
* **Transcription/Audio Agent:** Processes audio and video blobs (e.g., recorded Zoom meetings or training videos), generating timestamped transcripts and summarizing key topics.
* **Data Quality (QA) Agent:** Before any parsed text is embedded and loaded into LanceDB, the QA Agent reviews the output for completeness, formatting errors, and sensitive data (interacting with `dlp_redactor.py` to mask PII/PHI).

---

## 4. Orchestration Agent Control

The **ReAct Orchestrator Agent** acts as the "brain" managing the entire lifecycle. It does not perform ingestion itself; instead, it *controls* the Ingestion Agent through adaptive routing and delegation logic.

1. **User-Triggered Execution:** If a user types, *"Please update the knowledge base with the new Q3 compliance video at gs://bucket/video.mp4"*, the Orchestrator maps this intent to its `trigger_ingestion_agent` tool and makes a POST request to the Ingestion Agent's FastAPI endpoint.
2. **State Monitoring:** The Orchestrator can poll the Ingestion Agent's status endpoints to report progress back to the user.
3. **Agent-to-Agent (A2A) Delegation:** The Orchestrator abstracts the complexity. The user simply asks for a sync, and the Orchestrator determines which MCP tools and Ingestion pipelines need to be activated based on the data type.

---

## 5. CRAG Best Practices & Recommendations

Based on the **Corrective Retrieval Augmented Generation (CRAG)** framework and Full-Link Optimization strategies, the Ingestion Agent must format data in a way that maximizes the effectiveness of downstream evaluation.

### Best Practices for Ingestion to Support CRAG:

1. **Structure-Aware Partitioning (Higress-Native Splitter):**
   * **Do not use naive character-count splitters.** 
   * The Ingestion Agent must parse the Abstract Syntax Tree (AST) of documents (e.g., Markdown or HTML). 
   * **Why?** CRAG evaluates chunks as *Correct*, *Ambiguous*, or *Incorrect*. If a chunk is cut mid-sentence or separates a code block from its explanation, the CRAG Evaluator will flag it as *Ambiguous* or *Incorrect*, leading to unnecessary external web searches or discarded context. Context bubbles must remain semantically whole.

2. **Metadata Injection for the Boost Ranker:**
   * The Ingestion Agent must append rigorous metadata to every vector (e.g., `document_type`, `version`, `authoritative_status`).
   * **Why?** To prevent "metadata blindness," the search infrastructure uses a Boost Ranker to apply arithmetic modifiers (prioritizing V2.0 over V1.0). The Ingestion Agent is solely responsible for ensuring this metadata is accurately extracted and attached during the ETL pipeline.

3. **Placeholder Preservation for PII:**
   * During ingestion, if the Data Quality Agent identifies PII, it should use structural placeholders (e.g., `[PERSON_NAME]`) rather than deleting the text.
   * **Why?** This preserves the grammatical structure of the sentence, allowing the downstream LLM and CRAG evaluator to understand the relationships between entities without violating security perimeters.

4. **Multi-Granular Span Organization:**
   * When processing complex documents (like tables), the Ingestion Agent should index data hierarchically. A small chunk (e.g., a table row) should retain pointers to its parent context (e.g., the table header).
   * **Why?** During the Retrieval phase, if a highly specific chunk is found, the system can expand the "Context Bubble" to pull the surrounding structure, providing the CRAG evaluator with sufficient context to mark the retrieval as *Correct* and proceed to Knowledge Refinement.
