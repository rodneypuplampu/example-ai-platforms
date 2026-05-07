# Phase 3: The N×M Integration Layer (MCP & Apigee X)

This phase establishes the secure, standardized integration layer using the Model Context Protocol (MCP) and Apigee X. By abstracting the N×M connection complexity, we ensure the multi-agent system can securely invoke enterprise actions and retrieve real-time data.

## Step 1: Stand up the MCP Server

We will use the `FastMCP` framework to expose standard JSON-RPC 2.0 endpoints that the Orchestrator Agent can discover and interact with. 

### 1.1 Implement MCP Server Logic
Populate `mcp_server/server.py` to initialize the MCP server and register its endpoints:

```python
import os
import httpx
from mcp.server.fastmcp import FastMCP
from mcp_server.tools.spanner_inventory import check_store_inventory

# Initialize the FastMCP Server
mcp = FastMCP("MfgLogistics_MCP")

# APIGEE X Gateway acts as the secure door to backend APIs
APIGEE_X_GATEWAY = os.getenv("APIGEE_GATEWAY_URL", "https://api.acmecorp.internal")

@mcp.tool()
async def inventory_lookup(sku: str, store_id: str) -> str:
    """
    Tool exposed to Gemini to query real-time HTAP+V inventory.
    Traffic flows securely through APIGEE X for Auth, Rate Limiting, and DLP.
    """
    return await check_store_inventory(sku, store_id, APIGEE_X_GATEWAY)

if __name__ == "__main__":
    # Runs the MCP server using Standard IO (or SSE for remote environments)
    # The Orchestrator interacts via this transport layer
    mcp.run(transport="stdio")
```

### 1.2 Implement Specific Tools
Populate `mcp_server/tools/spanner_inventory.py` to perform the actual external API call via the Apigee Gateway:

```python
import os
import httpx

async def check_store_inventory(sku: str, store_id: str, gateway_url: str) -> str:
    """Queries inventory by routing through Apigee X."""
    
    # In a production environment, this token would be fetched via Google Auth or a dedicated Secret Manager
    token = os.getenv('APIGEE_OAUTH_TOKEN')
    headers = {"Authorization": f"Bearer {token}"}
    
    endpoint = f"{gateway_url}/v1/inventory/stores/{store_id}/skus/{sku}"
    
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(endpoint, headers=headers)
            if response.status_code == 200:
                return response.text
            return f"Inventory API error: {response.status_code} - {response.text}"
        except httpx.RequestError as exc:
            return f"An error occurred while requesting {exc.request.url!r}."
```

## Step 2: Configure APIGEE X API Management Layer

Apigee X serves as the security and governance layer between the MCP Server and the backend microservices. We will define these configurations as code using an OpenAPI spec.

### 2.1 Define OpenAPI Specification
Populate `api_gateway/apigee_x_configs/openapi-spec.yaml` with the API contract:

```yaml
openapi: 3.0.0
info:
  title: Enterprise Logistics API
  version: 1.0.0
servers:
  - url: https://api.acmecorp.internal/v1
paths:
  /inventory/stores/{store_id}/skus/{sku}:
    get:
      summary: Retrieves inventory for a given SKU at a specific store.
      parameters:
        - in: path
          name: store_id
          required: true
          schema:
            type: string
        - in: path
          name: sku
          required: true
          schema:
            type: string
      responses:
        '200':
          description: OK
```

### 2.2 Enforce Policies (Spike Arrest & OAuth)
In an Apigee Proxy deployment, XML policies govern the traffic. Here is a conceptual implementation of a Spike Arrest policy that would be deployed to Apigee:

```xml
<!-- api_gateway/apigee_x_configs/policies/SpikeArrest.xml -->
<SpikeArrest async="false" continueOnError="false" enabled="true" name="Spike-Arrest-1">
    <DisplayName>Spike Arrest 1</DisplayName>
    <Properties/>
    <Identifier ref="request.header.Authorization"/>
    <Rate>30ps</Rate>
</SpikeArrest>
```
*Note: Deploying to Apigee X typically involves packaging these files into an API Proxy bundle and deploying it via the Apigee Management API or a CI/CD pipeline (e.g., using `apigeecli` or Maven plugin).*

### 2.3 Network Alignment: Private Service Connect (PSC)
To align with the zero-trust architecture defined in `NETWORKING_INFRASTRUCTURE.md`, Apigee must route traffic to the backend Internal Load Balancer (ILB) using **Private Service Connect (PSC) Network Endpoint Groups (NEGs)** instead of VPC Peering.

```bash
# Example command to create a PSC NEG pointing to the published backend service
gcloud compute network-endpoint-groups create apigee-psc-neg \
    --network-endpoint-type=private-service-connect \
    --psc-target-service="projects/$PROJECT_ID/regions/us-central1/serviceAttachments/backend-ilb-attachment" \
    --region=us-central1
```

### 2.4 Streaming Optimization (Payload Buffering)
The Model Context Protocol and GenAI workflows heavily rely on Server-Sent Events (SSE). You must explicitly disable payload buffering in the Apigee X proxy configuration for streaming endpoints to prevent the gateway from holding AI tokens until the stream is complete.

## Step 3: Tool Registration and Orchestrator Prep

The Orchestrator Agent must be able to discover the MCP server tools dynamically. While we will build the full Orchestrator in Phase 4, we must verify the MCP connection logic now.

### 3.1 Test Local MCP Connectivity
You can test that the `FastMCP` server exposes its tools correctly by querying the MCP inspector or invoking it directly:

```bash
# Run the FastMCP Inspector to view exposed tools via a local web interface
mcp dev mcp_server/server.py
```
This will start a local UI where you can test the `inventory_lookup` tool, ensuring the JSON-RPC schema is correctly generated before the Orchestrator is plugged in.
