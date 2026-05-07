# Phase 4: Multi-Agent Orchestration & Cognitive Search

This phase is the "brain" of the Enterprise AI Platform. We will finalize the semantic search capabilities (Full-Link RAG) and stand up the ReAct Orchestrator Agent, which acts as the primary decision-maker routing tasks to our previously built modules.

## Step 1: Full-Link RAG Pipeline

The cognitive search pipeline uses LanceDB for high-speed dense retrieval and Google Vertex Search as an enterprise-grade hybrid fallback, intelligently merging the signals.

### 1.1 Complete the Semantic Manager
Populate `semantic_search/llamaindex_manager.py`:

```python
import os
import lancedb
from llama_index.core import VectorStoreIndex, QueryBundle
from llama_index.vector_stores.lancedb import LanceDBVectorStore
from google.cloud import discoveryengine_v1 as vertex_search
from semantic_search.rrf_merger import apply_rrf

class MfgLogisticsSemanticSearch:
    def __init__(self, lancedb_uri="gs://ai-platform-lancedb-store/vectors"):
        
        # 1. Connect to LanceDB Vector Destination
        self.db = lancedb.connect(lancedb_uri)
        self.vector_store = LanceDBVectorStore(uri=lancedb_uri, table_name="enterprise_knowledge")
        self.index = VectorStoreIndex.from_vector_store(vector_store=self.vector_store)
        self.lancedb_retriever = self.index.as_retriever(similarity_top_k=5)

        # 2. Connect to Vertex Search API for enterprise fallback / hybrid retrieval
        self.vertex_client = vertex_search.SearchServiceClient()
        self.vertex_config = f"projects/{os.getenv('PROJECT_ID')}/locations/global/collections/default_collection/engines/ai-platform-engine/servingConfigs/default_config"

    def _query_vertex_search(self, query_str: str):
        """Fallback to Vertex Search for sparse/hybrid signals."""
        request = vertex_search.SearchRequest(
            serving_config=self.vertex_config,
            query=query_str,
            page_size=5,
        )
        response = self.vertex_client.search(request)
        return [result.document.content for result in response.results]

    def retrieve_context(self, query: str):
        """Executes Hybrid retrieval using LanceDB and Vertex Search with RRF."""
        query_bundle = QueryBundle(query_str=query)
        
        # A. High-speed Dense Retrieval (LanceDB)
        lancedb_nodes = self.lancedb_retriever.retrieve(query_bundle)
        lancedb_texts = [node.node.text for node in lancedb_nodes]
        
        # B. Enterprise Search Fallback (Vertex Search)
        vertex_texts = self._query_vertex_search(query)
        
        # C. Reciprocal Rank Fusion (RRF) to merge lists
        merged_context = apply_rrf(lancedb_texts, vertex_texts)
        
        return "\n\n".join(merged_context)
```

### 1.2 Implement Reciprocal Rank Fusion (RRF)
Populate `semantic_search/rrf_merger.py` to handle the ranking math:

```python
def apply_rrf(list1: list, list2: list, k=60):
    """
    Combines two ranked lists of strings using Reciprocal Rank Fusion.
    k is a smoothing constant, typically 60.
    """
    scores = {}
    
    # Process first list (LanceDB)
    for rank, doc in enumerate(list1):
        if doc not in scores:
            scores[doc] = 0.0
        scores[doc] += 1.0 / (k + rank + 1)
        
    # Process second list (Vertex Search)
    for rank, doc in enumerate(list2):
        if doc not in scores:
            scores[doc] = 0.0
        scores[doc] += 1.0 / (k + rank + 1)
        
    # Sort by descending RRF score
    sorted_docs = sorted(scores.items(), key=lambda item: item[1], reverse=True)
    
    # Return the top merged documents
    return [doc[0] for doc in sorted_docs[:5]]
```

## Step 2: Instantiate the ReAct Orchestrator

The Orchestrator Agent relies on Gemini 1.5 Pro to evaluate incoming prompts and decide whether to retrieve knowledge, trigger ingestion, or execute a tool.

### 2.1 Build the Agent
Populate `agents/orchestrator_agent/agent.py`:

```python
import os
from llama_index.core.agent import ReActAgent
from llama_index.llms.gemini import Gemini
from llama_index.core.tools import FunctionTool
from semantic_search.llamaindex_manager import MfgLogisticsSemanticSearch

# Initialize Reasoning Engine
llm = Gemini(model="models/gemini-1.5-pro", api_key=os.getenv("GEMINI_API_KEY"))
search_engine = MfgLogisticsSemanticSearch()

def trigger_ingestion_agent(source: str) -> str:
    """Delegates ETL/Ingestion tasks to the Data Ingestion Agent."""
    # In a real setup, this would publish to the Pub/Sub topic from Phase 2
    return f"Triggered ingestion agent to sync disparate source: {source}"

def perform_semantic_search(query: str) -> str:
    """Delegates technical knowledge retrieval to LlamaIndex RAG engine."""
    return search_engine.retrieve_context(query)

def invoke_mcp_api(tool_name: str, params: dict) -> str:
    """Routes an API execution request via the Model Context Protocol Server."""
    # The Orchestrator interacts with the Phase 3 FastMCP server via JSON-RPC/stdio
    return f"MCP execution requested for {tool_name} via Apigee X."

# Register capabilities as tools for the LLM
tools = [
    FunctionTool.from_defaults(fn=trigger_ingestion_agent),
    FunctionTool.from_defaults(fn=perform_semantic_search),
    FunctionTool.from_defaults(fn=invoke_mcp_api)
]

# Initialize Orchestrator
orchestrator_agent = ReActAgent.from_tools(
    tools,
    llm=llm,
    verbose=True,
    system_prompt=(
        "You are the Enterprise Multi-Agent Orchestrator. Route user queries intelligently.\n"
        "1. For ETL or Data Pulls, delegate to the Ingestion Agent.\n"
        "2. For enterprise API tasks (e.g. inventory lookups), delegate to MCP APIs.\n"
        "3. For knowledge, policies, or general questions, delegate to Semantic Search."
    )
)

if __name__ == "__main__":
    # Simple CLI loop for testing
    while True:
        user_input = input("\nEnter query: ")
        if user_input.lower() in ['exit', 'quit']:
            break
        response = orchestrator_agent.chat(user_input)
        print(f"\nResponse: {response}")
```

## Step 3: Agent-to-Agent (A2A) Testing

Now that the orchestrator is wired up, you must validate that its ReAct reasoning accurately delegates to the appropriate sub-system.

### 3.1 Validate Routing
Run the agent in your terminal:
```bash
python agents/orchestrator_agent/agent.py
```

Test the following scenarios:
1. **Semantic Search Test:** Ask *"What are the safety guidelines for operating a forklift?"*
   - *Expected:* The agent should call `perform_semantic_search` to query LanceDB/Vertex.
2. **MCP API Test:** Ask *"What is the inventory for SKU 99281 at store 402?"*
   - *Expected:* The agent should call `invoke_mcp_api`.
3. **Ingestion Test:** Ask *"Please trigger an immediate sync of the Snowflake ERP data."*
   - *Expected:* The agent should call `trigger_ingestion_agent`.

## Completion of Phase 4
You have now established the cognitive orchestrator and the full RAG pipeline, successfully demonstrating A2A delegation.

**Deployment Checklist:**
- [ ] `llamaindex_manager.py` connected to LanceDB and Vertex Search.
- [ ] RRF logic successfully merges signals.
- [ ] ReAct Agent initialized with Gemini 1.5 Pro.
- [ ] Tool routing validated via terminal testing.
