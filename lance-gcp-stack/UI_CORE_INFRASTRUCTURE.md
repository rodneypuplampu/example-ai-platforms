# 🏗️ Platform Interface: Core Infrastructure & Networking
**Target Persona:** Network Engineers, Core GKE Admins, and Cloud Infrastructure Engineers

## 1. Role and Purpose
This frontend interface is dedicated to the foundational layer of the Enterprise AI Platform. Infrastructure engineers use this dashboard to manage the "Golden Path" traffic flow, VPC boundaries, and the underlying compute clusters before any AI logic is applied. It provides a macro-view of the enterprise perimeter.

## 2. Key Dashboard Views

### A. The Edge & Ingress Command Center
*   **Global ALB Traffic Map:** Real-time visualization of incoming traffic across Google’s global edge network (Anycast IPs).
*   **Private Service Connect (PSC) Status:** A visual graph showing the health and active connections of the Southbound PSC Network Endpoint Groups (NEGs) routing traffic from the External Load Balancers to the Apigee X API Gateway.
*   **Latency & Payload Buffering:** Metrics displaying end-to-end latency and verifying that payload buffering is explicitly disabled for streaming endpoints (essential for GenAI Server-Sent Events).

### B. Core VPC & Perimeter Management
*   **VPC Service Controls (VPC-SC) Boundary Map:** A cryptographic perimeter viewer. It visually highlights which services (Vertex AI, Cloud Storage) are currently protected inside the VPC-SC boundary.
*   **Egress / Ingress Violation Logs:** Immediate alerts for any API calls that attempt to breach the VPC-SC perimeter (e.g., an unauthorized attempt to exfiltrate data to an external S3 bucket).
*   **Private Google Access (PGA) Verifier:** A checklist view ensuring all subnets correctly resolve Google APIs to `restricted.googleapis.com` (199.36.153.4/30).

### C. Core Compute Provisioning (GKE & Cloud Run)
*   **Cluster Fleet Overview:** High-level metrics of the foundational VPC-native GKE clusters (CPU/Memory utilization at the node-pool level).
*   **Internal Application Load Balancer (ILB) Topology:** Maps out the Zonal NEGs connecting the ILB directly to the GKE Pod IPs, bypassing traditional `kube-proxy` hops for low-latency AI responses.

## 3. Key Administrative Actions
*   Provision new VPC-native private GKE clusters.
*   Update VPC-SC policies (e.g., allowlisting a specific custom domain for egress).
*   Manage edge TLS certificates for the Global External ALB.
