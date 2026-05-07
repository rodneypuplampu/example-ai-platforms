# 🐝 Platform Interface: Tenant Workloads & eBPF Networking
**Target Persona:** Platform Engineers, System GKE Admins, and eBPF/Mesh Networking Specialists

## 1. Role and Purpose
Once the core infrastructure is laid, this interface manages the "Engine Room" where the AI microservices actually run. It focuses on multi-tenant isolation, container lifecycle management, and strict zero-trust pod-to-pod networking using GKE Dataplane V2 (Cilium/eBPF) and Cloud Service Mesh.

## 2. Key Dashboard Views

### A. Namespace & Multi-Tenant Resource Map
*   **Tenant Topology:** Visual grouping of pods and containers by namespace (e.g., `orchestrator-ns`, `ingestion-ns`, `mcp-server-ns`).
*   **Workload Identity Federation Tracker:** Displays which Kubernetes Service Accounts (KSAs) are bound to which Google Service Accounts (GSAs). Assures administrators that no long-lived JSON keys are mounted as secrets.
*   **Hardware Allocation (GPU Isolation):** Detailed views of node resource allocation, verifying specific partitioning (e.g., GPU 0 for embeddings, GPU 1 for LLM, GPU 2 for Reranker).

### B. eBPF Network Flow & Policy Engine (GKE Dataplane V2)
*   **Real-Time eBPF Flow Graph:** A highly granular map built on Cilium/eBPF showing exact traffic flows between pods.
*   **Network Policy Verifier:** A "Default Deny" audit board. Visually verifies that the `mcp-spanner-inventory` pod is *only* accepting ingress from the `orchestrator-agent` pod and dropping all other requests.
*   **Dropped Packet Analysis:** Immediate diagnostics for traffic blocked by micro-segmentation rules, helping developers quickly debug "Connection Refused" errors between agents.

### C. Cloud Service Mesh (mTLS) Command
*   **mTLS Enforcement Dashboard:** Verifies that strict Mutual TLS is active for all agent-to-agent communication.
*   **Service Mesh Tracing:** Visualizes the microservice call chain (e.g., Orchestrator -> FastMCP Server -> Apigee X).

## 3. Key Administrative Actions
*   Scale specific agent deployments (e.g., increasing Ingestion Agent replicas during a massive ETL load).
*   Apply or modify Kubernetes Network Policies for new MCP tool deployments.
*   Rotate or update Workload Identity KSA bindings.
