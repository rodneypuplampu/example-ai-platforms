# 🛡️ Platform Interface: Cybersecurity, Monitoring & Ops
**Target Persona:** Cybersecurity Analysts, SecOps, and SRE (Site Reliability Engineers)

## 1. Role and Purpose
This interface is the defensive shield and the diagnostic heart of the platform. It provides specialized views into the deterministic security filters protecting the LLMs, and offers deep OpenTelemetry instrumentation to troubleshoot complex, multi-hop agentic logic that traditional logging cannot capture.

## 2. Key Dashboard Views

### A. WAF & Edge Threat Map
*   **Cloud Armor Live Feed:** Visualizes mitigated DDoS attacks, OWASP Top 10 exploits, and botnet scraping attempts occurring at the Global External ALB.
*   **Geographic IP Filtering Board:** Manages and audits allowed corporate IP ranges and Identity-Aware Proxy (IAP) blocks.

### B. Responsible AI & Prompt Governance
*   **Model Armor Dashboard:** The core GenAI security view. It displays real-time intercepts of prompt injection attacks (e.g., "Ignore previous instructions") and tracks blocks for unsafe or biased content.
*   **DLP Redaction Audit Log:** An auditable ledger showing the synchronous redaction of PII (Personally Identifiable Information). Analysts can view metrics on how frequently placeholders (e.g., `[CREDIT_CARD_NUMBER]`) are swapped into the text stream before it reaches the reasoning engine.

### C. OpenTelemetry & MLOps Troubleshooting
*   **Trace & Span Explorer:** The critical diagnostic tool for agentic workflows. Instead of flat logs, this visualizes the full lifecycle of a request as a Directed Acyclic Graph (DAG). An SRE can drill down to see exactly how long the hybrid vector search took vs. the Model Armor check.
*   **Latency & Semantic Cache Heatmap:** Monitors the Dynamic Threshold semantic caching (0.95/0.98). Allows SREs to investigate if a drop in cache hit rates is causing latency spikes.
*   **Vertex ML Metadata Lineage:** A governance view to trace the "family tree" of every dataset and parameter that led to a specific LLM decision, essential for compliance audits.

## 3. Key Administrative Actions
*   Update Model Armor safety thresholds or custom blocklists.
*   Add new `infoTypes` (e.g., a new proprietary internal ID format) to the DLP Redactor.
*   Query trace IDs to perform root cause analysis (RCA) on high-latency AI requests.
