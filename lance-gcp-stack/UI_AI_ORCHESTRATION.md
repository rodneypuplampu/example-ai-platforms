# 🧠 Platform Interface: AI Orchestration & Integration
**Target Persona:** AI Engineers, Data Engineers, and Application Developers

## 1. Role and Purpose
This interface is where the actual intelligence of the platform is managed. It controls the "Brain" (the ReAct Orchestrator) and the "Library" (the Ingestion pipeline and Vector Store). It provides developers with the tools to register new integrations, monitor data pipelines, and evaluate the quality of the AI's reasoning.

## 2. Key Dashboard Views

### A. Ingestion & Storage Command (The External Brain)
*   **Cocoindex ETL Pipeline Tracker:** Visualizes the status of declarative data syncs from disparate sources (Snowflake, BigQuery, GCS Buckets, PySpark streams).
*   **Higress-Native Splitter Analytics:** Shows how effectively Markdown AST and code blocks are being preserved during chunking.
*   **LanceDB Vector Store Metrics:** Displays index sizes, partition health, and tenant-specific storage utilization.

### B. Agent Orchestration & CRAG Evaluation
*   **Tri-Route Dispatch Monitor:** A visual breakdown of how the Orchestrator is routing user queries (Simple Vector Route vs. Complex HyDE Route vs. External Web Route).
*   **CRAG (Corrective RAG) Evaluator Board:** The primary quality control view. It categorizes recent retrievals as *Correct*, *Ambiguous*, or *Incorrect*, showing how often the system successfully performed Knowledge Refinement or had to fallback to an external Tavily web search to prevent hallucination.
*   **RRF (Reciprocal Rank Fusion) Tuning:** Allows engineers to visualize how Dense (LanceDB) and Sparse (Vertex Search) signals are being merged, and adjust the Metadata-Aware Boost Ranker weights.

### C. Integration & N×M Abstraction (MCP & Apigee)
*   **MCP Tool Registry:** A developer portal where new tools (e.g., a new Spanner inventory lookup tool) are registered to the FastMCP server. It displays the JSON-RPC schema that is exposed to the Orchestrator.
*   **Apigee X Integration Sandbox:** An interface to manage the OpenAPI specs, OAuth bindings, and Spike Arrest policies specifically applied to MCP external actions.

## 3. Key Administrative Actions
*   Manually trigger a Cocoindex incremental or full ETL pipeline run.
*   Register a new foundation model (e.g., upgrading from Gemini 1.5 Pro to Gemini 2.0) in the Orchestrator's toolkit.
*   Deploy and test new MCP tools before promoting them to production.
*   Adjust the HyDE prompt generation parameters to improve semantic matching.
