# Phase 2: Knowledge Ingestion & Vector Infrastructure

This guide details the implementation of Phase 2, which establishes the core data foundation for the Enterprise AI Platform. It covers deploying the LanceDB vector database on Google Cloud Storage, constructing the Cocoindex ETL pipeline for disparate data sources, and automating the ingestion process using GCP Pub/Sub.

## Step 1: Deploy Vector Database (LanceDB on GCS)

LanceDB provides a serverless vector database that runs directly on top of object storage. We will configure it to use a Google Cloud Storage (GCS) bucket as its backing store.

### 1.1 Create the GCS Bucket
First, create the storage bucket that will house the LanceDB vector files. Run this in your terminal:

```bash
# Create a standard storage bucket in a specific region (e.g., us-central1)
gcloud storage buckets create gs://ai-platform-lancedb-store \
    --project=$PROJECT_ID \
    --location=us-central1 \
    --uniform-bucket-level-access

# Ensure the Ingestion Agent SA has the correct permissions (if not done in Phase 1)
gcloud storage buckets add-iam-policy-binding gs://ai-platform-lancedb-store \
    --member="serviceAccount:ingestion-agent-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/storage.objectAdmin"
```

### 1.2 Initialize LanceDB Client
Populate `vector_database/lancedb_client.py` to handle connections to the vector store:

```python
import lancedb
import os

def get_lancedb_connection():
    """Establishes a connection to the LanceDB instance on GCS."""
    # Ensure Google Cloud credentials are set in the environment
    uri = "gs://ai-platform-lancedb-store/vectors"
    
    try:
        db = lancedb.connect(uri)
        print(f"Successfully connected to LanceDB at {uri}")
        return db
    except Exception as e:
        print(f"Failed to connect to LanceDB: {e}")
        raise

if __name__ == "__main__":
    get_lancedb_connection()
```

## Step 2: Build Cocoindex Pipelines

We will use Cocoindex to declaratively manage ETL from our enterprise sources into LanceDB, utilizing the Higress-Native AST Markdown splitter to preserve document structure.

### 2.1 Configure the Pipeline Definition
Populate `data_pipeline/cocoindex_etl/pipeline.py` with the following implementation:

```python
import os
import cocoindex as cc

def build_enterprise_etl_pipeline():
    """Builds the ETL pipeline managed by Cocoindex for LanceDB destination."""
    
    # 1. Define Disparate Data Sources
    # BigQuery for HTAP workloads
    bq_source = cc.sources.BigQuery(
        project_id=os.getenv("PROJECT_ID"),
        dataset="htap_inventory", 
        table="spanner_sync"
    )
    
    # Snowflake for ERP data
    snowflake_source = cc.sources.Snowflake(
        account=os.getenv("SNOWFLAKE_ACCOUNT"),
        warehouse="RETAIL_WH", 
        database="LEGACY_ERP"
    )
    
    # Google Cloud Storage for raw Document AI/PDFs and blueprints
    gcs_source = cc.sources.GoogleCloudStorage(
        bucket="ai-platform-blueprints-raw", 
        prefix="incoming/"
    )

    # 2. Construct Pipeline & Transformations
    pipeline = cc.Pipeline(
        name="omnichannel_etl_pipeline",
        sources=[bq_source, snowflake_source, gcs_source],
        transformations=[
            # Higress-Native Splitter preserves structure (Markdown AST/Tables)
            cc.transforms.DocumentSplitter(method="ast_markdown", max_tokens=1024),
            cc.transforms.Embedder(model="models/text-embedding-004")
        ],
        # 3. LanceDB Vector DB Destination
        destination=cc.destinations.LanceDB(
            uri="gs://ai-platform-lancedb-store/vectors",
            table_name="enterprise_knowledge"
        )
    )
    return pipeline

def run_initial_sync():
    """Executes a full initial hydration of the vector database."""
    print("Starting initial data hydration sync...")
    pipeline = build_enterprise_etl_pipeline()
    pipeline.run(incremental=False)
    print("Initial sync complete.")

if __name__ == "__main__":
    run_initial_sync()
```

## Step 3: Automate Ingestion

To ensure the vector database stays synchronized with the source systems, we will set up GCP Pub/Sub to trigger incremental runs whenever new data arrives.

### 3.1 Create Pub/Sub Topic and Subscription
Run the following commands to create the event infrastructure:

```bash
# Create a Pub/Sub topic for ingestion triggers
gcloud pubsub topics create data-ingestion-triggers --project=$PROJECT_ID

# Create a subscription for the ingestion agent to pull from
gcloud pubsub subscriptions create ingestion-agent-sub \
    --topic=data-ingestion-triggers \
    --project=$PROJECT_ID
```

*(Optional)* If you want GCS to automatically publish to this topic when new PDFs/markdown land in the raw bucket:
```bash
# Create the raw bucket first
gcloud storage buckets create gs://ai-platform-blueprints-raw --project=$PROJECT_ID --location=us-central1

# Configure GCS to publish Object Finalize events to the Pub/Sub topic
gcloud storage service-agent --project=$PROJECT_ID | grep -o 'service-[0-9]*@gs-project-accounts.iam.gserviceaccount.com' | xargs -I {} gcloud pubsub topics add-iam-policy-binding data-ingestion-triggers --member="serviceAccount:{}" --role="roles/pubsub.publisher"

gcloud storage buckets notifications create gs://ai-platform-blueprints-raw \
    --topic=data-ingestion-triggers \
    --event-types=OBJECT_FINALIZE
```

### 3.2 Implement the Pub/Sub Listener
Populate `agents/ingestion_agent/pubsub_listener.py` to listen for events and trigger the ETL pipeline:

```python
import os
import json
from google.cloud import pubsub_v1
from data_pipeline.cocoindex_etl.pipeline import build_enterprise_etl_pipeline

PROJECT_ID = os.getenv("PROJECT_ID")
SUBSCRIPTION_ID = "ingestion-agent-sub"

def trigger_etl_run(incremental=True):
    """Triggers the Cocoindex ETL run."""
    print(f"Triggering ETL Pipeline (Incremental: {incremental})")
    pipeline = build_enterprise_etl_pipeline()
    pipeline.run(incremental=incremental)

def callback(message: pubsub_v1.subscriber.message.Message) -> None:
    print(f"Received message: {message.data.decode('utf-8')}")
    try:
        # Acknowledge the message immediately
        message.ack()
        
        # Trigger an incremental sync when new data is detected
        trigger_etl_run(incremental=True)
    except Exception as e:
        print(f"Error processing message: {e}")

def start_listening():
    """Starts the Pub/Sub listener."""
    subscriber = pubsub_v1.SubscriberClient()
    subscription_path = subscriber.subscription_path(PROJECT_ID, SUBSCRIPTION_ID)
    
    print(f"Listening for messages on {subscription_path}...\n")
    streaming_pull_future = subscriber.subscribe(subscription_path, callback=callback)
    
    try:
        streaming_pull_future.result()
    except KeyboardInterrupt:
        streaming_pull_future.cancel()

if __name__ == "__main__":
    start_listening()
```

## Completion of Phase 2
You have now configured the vector database, established the ETL pipelines for data ingestion, and set up an event-driven automation system.
