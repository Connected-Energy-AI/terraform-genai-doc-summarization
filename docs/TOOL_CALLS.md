# Tool Calls

This document provides comprehensive documentation of all tool calls, API integrations, and service interactions used by the Generative AI Document Summarization agent.

## Overview

The document summarization agent integrates with multiple Google Cloud services through their respective Python client libraries. This document details each tool call, its purpose, parameters, and expected responses.

## Tool Call Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Document Summarization Agent                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐    │
│   │  Functions  │   │ Document AI │   │  Vertex AI  │   │  BigQuery   │    │
│   │  Framework  │   │   Client    │   │   Client    │   │   Client    │    │
│   └─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘    │
│          │                 │                 │                 │           │
└──────────┼─────────────────┼─────────────────┼─────────────────┼───────────┘
           │                 │                 │                 │
           ▼                 ▼                 ▼                 ▼
    ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
    │   Cloud     │   │  Document   │   │   Vertex    │   │  BigQuery   │
    │  Functions  │   │     AI      │   │     AI      │   │   Service   │
    └─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘
```

## Cloud Functions Framework

### Cloud Event Handler

**Tool:** `functions_framework.cloud_event`

**Purpose:** Register the main entry point for Eventarc-triggered Cloud Functions.

**Usage:**
```python
import functions_framework
from cloudevents.http import CloudEvent

@functions_framework.cloud_event
def on_cloud_event(event: CloudEvent) -> None:
    """Process a new document from an Eventarc event."""
    # Event processing logic
```

**Event Structure:**
```python
event.data = {
    "id": str,           # Event ID
    "bucket": str,       # Source bucket name
    "name": str,         # Object name/path
    "contentType": str,  # MIME type
    "timeCreated": str,  # ISO 8601 timestamp
}
```

## Document AI Tool Calls

### Client Initialization

**Tool:** `documentai.DocumentProcessorServiceClient`

**Purpose:** Initialize Document AI client with regional endpoint.

**Usage:**
```python
from google.api_core.client_options import ClientOptions
from google.cloud import documentai

documentai_client = documentai.DocumentProcessorServiceClient(
    client_options=ClientOptions(
        api_endpoint=f"{docai_location}-documentai.googleapis.com"
    )
)
```

**Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `client_options` | `ClientOptions` | Configuration options |
| `api_endpoint` | `str` | Regional API endpoint |

### Batch Process Documents

**Tool:** `documentai_client.batch_process_documents`

**Purpose:** Perform OCR on documents stored in Cloud Storage.

**Usage:**
```python
operation = documentai_client.batch_process_documents(
    request=documentai.BatchProcessRequest(
        name=processor_id,
        input_documents=documentai.BatchDocumentsInputConfig(
            gcs_documents=documentai.GcsDocuments(
                documents=[
                    documentai.GcsDocument(
                        gcs_uri=input_file,
                        mime_type=mime_type
                    ),
                ],
            ),
        ),
        document_output_config=documentai.DocumentOutputConfig(
            gcs_output_config=documentai.DocumentOutputConfig.GcsOutputConfig(
                gcs_uri=output_gcs_path,
            ),
        ),
    ),
)

# Wait for operation to complete
operation.result()
```

**Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `str` | Full processor ID path |
| `input_documents` | `BatchDocumentsInputConfig` | Input document configuration |
| `document_output_config` | `DocumentOutputConfig` | Output destination configuration |

**Response:**
```python
# Access metadata after completion
metadata = documentai.BatchProcessMetadata(operation.metadata)
output_gcs_path = metadata.individual_process_statuses[0].output_gcs_destination
```

### Parse Document from JSON

**Tool:** `documentai.Document.from_json`

**Purpose:** Parse Document AI output JSON into Document object.

**Usage:**
```python
blob_contents = blob.download_as_bytes()
document = documentai.Document.from_json(
    blob_contents,
    ignore_unknown_fields=True
)
text = document.text
```

## Vertex AI / Generative AI Tool Calls

### Client Initialization

**Tool:** `genai.Client`

**Purpose:** Initialize Vertex AI Generative AI client.

**Usage:**
```python
from google import genai

client = genai.Client(
    vertexai=True,
    project=project,
    location=location
)
```

**Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `vertexai` | `bool` | Use Vertex AI backend |
| `project` | `str` | Google Cloud project ID |
| `location` | `str` | Cloud region |

### Generate Content

**Tool:** `client.models.generate_content`

**Purpose:** Generate text summaries using Gemini model.

**Usage:**
```python
from google.genai.types import GenerateContentConfig

response = client.models.generate_content(
    model="gemini-2.0-flash",
    contents=doc_text,
    config=GenerateContentConfig(
        system_instruction=[
            "Give me a summary of the following text."
            "Use simple language and give examples."
        ]
    ),
)

summary = response.text
```

**Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `model` | `str` | Model identifier |
| `contents` | `str` | Input text to summarize |
| `config` | `GenerateContentConfig` | Generation configuration |

**Configuration Options:**
```python
GenerateContentConfig(
    system_instruction=[str],    # System prompts
    temperature=float,           # Creativity (0-1)
    top_p=float,                 # Nucleus sampling
    top_k=int,                   # Top-k sampling
    max_output_tokens=int,       # Maximum response length
    stop_sequences=[str],        # Stop generation triggers
)
```

## Cloud Storage Tool Calls

### Client Initialization

**Tool:** `storage.Client`

**Purpose:** Initialize Cloud Storage client.

**Usage:**
```python
from google.cloud import storage

storage_client = storage.Client()
```

### List Blobs

**Tool:** `storage_client.list_blobs`

**Purpose:** List objects in a Cloud Storage bucket.

**Usage:**
```python
for blob in storage_client.list_blobs(output_bucket, prefix=output_prefix):
    blob_contents = blob.download_as_bytes()
    # Process blob contents
```

**Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `bucket_or_name` | `str` | Bucket name |
| `prefix` | `str` | Filter by prefix |

### Download Blob

**Tool:** `blob.download_as_bytes`

**Purpose:** Download blob contents as bytes.

**Usage:**
```python
blob_contents = blob.download_as_bytes()
```

## BigQuery Tool Calls

### Client Initialization

**Tool:** `bigquery.Client`

**Purpose:** Initialize BigQuery client.

**Usage:**
```python
from google.cloud import bigquery

bq_client = bigquery.Client(project=project)
```

### Get Table

**Tool:** `bq_client.get_table`

**Purpose:** Get reference to BigQuery table.

**Usage:**
```python
table = bq_client.get_table(f"{bq_dataset}.{bq_table}")
```

### Insert Rows

**Tool:** `bq_client.insert_rows`

**Purpose:** Insert data into BigQuery table.

**Usage:**
```python
bq_client.insert_rows(
    table=bq_client.get_table(f"{bq_dataset}.{bq_table}"),
    rows=[
        {
            "event_id": event_id,
            "time_uploaded": time_uploaded,
            "time_processed": datetime.now(),
            "document_path": doc_path,
            "document_text": doc_text,
            "document_summary": doc_summary,
        },
    ],
)
```

**Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `table` | `Table` | Target table reference |
| `rows` | `list[dict]` | Rows to insert |

## Terraform Tool Calls

### Project Services Module

**Tool:** `terraform-google-modules/project-factory/google//modules/project_services`

**Purpose:** Enable required Google Cloud APIs.

**Usage:**
```hcl
module "project_services" {
  source                      = "terraform-google-modules/project-factory/google//modules/project_services"
  version                     = "~> 18.0"
  disable_services_on_destroy = var.disable_services_on_destroy
  project_id                  = var.project_id

  activate_apis = [
    "aiplatform.googleapis.com",
    "artifactregistry.googleapis.com",
    "bigquery.googleapis.com",
    "cloudbuild.googleapis.com",
    "cloudfunctions.googleapis.com",
    "cloudresourcemanager.googleapis.com",
    "compute.googleapis.com",
    "config.googleapis.com",
    "documentai.googleapis.com",
    "eventarc.googleapis.com",
    "iam.googleapis.com",
    "run.googleapis.com",
    "serviceusage.googleapis.com",
    "storage-api.googleapis.com",
    "storage.googleapis.com",
  ]
}
```

## API Dependencies

### Required APIs

| API | Purpose |
|-----|---------|
| `aiplatform.googleapis.com` | Vertex AI for LLM |
| `artifactregistry.googleapis.com` | Container images |
| `bigquery.googleapis.com` | Data storage |
| `cloudbuild.googleapis.com` | Build functions |
| `cloudfunctions.googleapis.com` | Serverless compute |
| `cloudresourcemanager.googleapis.com` | Resource management |
| `compute.googleapis.com` | Compute resources |
| `config.googleapis.com` | Infrastructure Manager |
| `documentai.googleapis.com` | OCR processing |
| `eventarc.googleapis.com` | Event routing |
| `iam.googleapis.com` | Access control |
| `run.googleapis.com` | Cloud Run |
| `serviceusage.googleapis.com` | Service management |
| `storage-api.googleapis.com` | Storage JSON API |
| `storage.googleapis.com` | Cloud Storage |

## Python Dependencies

### Core Dependencies

```
cloudevents>=1.10.1
functions-framework>=3.5.0
google-cloud-bigquery>=3.15.0
google-cloud-documentai>=2.24.0
google-cloud-storage>=2.14.0
google-genai>=0.2.2
```

### Test Dependencies

```
mypy
pytest
ruff
types-protobuf
```

## Error Handling Patterns

### Exception Logging

```python
try:
    process_document(...)
except Exception as e:
    logging.exception(e, stack_info=True)
```

### Retry Logic

For transient failures, consider implementing exponential backoff:

```python
from google.api_core.retry import Retry

@Retry(predicate=retry.if_exception_type(Exception))
def robust_api_call():
    # API call with automatic retry
    pass
```

## Rate Limits and Quotas

| Service | Limit | Unit |
|---------|-------|------|
| Document AI (batch) | 500 | Pages per request |
| Document AI (online) | 15 | Pages per document |
| Vertex AI | Varies | Requests per minute |
| BigQuery | 10,000 | Rows per insert |
| Cloud Functions | 1,000 | Instances (default) |

## Related Documentation

- [Agent Workflows](./AGENT_WORKFLOWS.md)
- [Document Types](./DOCUMENT_TYPES.md)
- [Training Data and Evaluations](./TRAINING_DATA_AND_EVALS.md)
- [Agentic Orchestration](./AGENTIC_ORCHESTRATION.md)
