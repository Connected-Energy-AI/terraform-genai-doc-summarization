# Agent Workflows

This document provides a comprehensive overview of the document summarization agent's workflows, from document ingestion to summary generation and storage.

## Overview

The Generative AI Document Summarization solution implements an event-driven architecture using Google Cloud services. The primary agent workflow consists of multiple stages that process documents uploaded to Cloud Storage.

## High-Level Workflow Diagram

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │     │                 │     │                 │
│   Cloud Storage │────▶│    Eventarc     │────▶│ Cloud Function  │────▶│    BigQuery     │
│   (Document     │     │    (Trigger)    │     │   (Webhook)     │     │   (Storage)     │
│    Upload)      │     │                 │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘     └─────────────────┘
                                                        │
                                          ┌─────────────┼─────────────┐
                                          │             │             │
                                          ▼             ▼             ▼
                                   ┌───────────┐ ┌───────────┐ ┌───────────┐
                                   │Document AI│ │ Vertex AI │ │Cloud      │
                                   │   (OCR)   │ │ (Gemini)  │ │Storage    │
                                   └───────────┘ └───────────┘ │(Temp)     │
                                                               └───────────┘
```

## Workflow Stages

### Stage 1: Document Upload Trigger

**Trigger Type:** `google.cloud.storage.object.v1.finalized`

When a document is uploaded to the designated Cloud Storage bucket (`summary-docs-*`), Eventarc detects the upload event and triggers the webhook Cloud Function.

**Event Data Structure:**
```json
{
  "id": "<event_id>",
  "bucket": "<bucket_name>",
  "name": "<filename>",
  "contentType": "<mime_type>",
  "timeCreated": "<ISO_8601_timestamp>"
}
```

### Stage 2: Document Text Extraction

The webhook function receives the cloud event and initiates the document processing pipeline:

1. **Document AI OCR Processing**
   - Uses `batch_process_documents` API for handling documents up to 500 pages
   - Extracts text content from various document formats
   - Stores intermediate OCR results in temporary Cloud Storage location

2. **Text Aggregation**
   - Collects all text chunks from OCR output
   - Concatenates text for summarization input

### Stage 3: AI-Powered Summarization

**Model:** Gemini 2.0 Flash (`gemini-2.0-flash`)

**System Instruction:**
```
Give me a summary of the following text.
Use simple language and give examples.
```

The agent uses Vertex AI's Generative AI capabilities to produce concise, accessible summaries of the extracted document text.

### Stage 4: Result Storage

The final summary and metadata are stored in BigQuery for analysis and retrieval:

**BigQuery Schema:**
| Field | Type | Description |
|-------|------|-------------|
| `event_id` | STRING | The Eventarc trigger event ID |
| `time_uploaded` | TIMESTAMP | When the document was uploaded |
| `time_processed` | TIMESTAMP | When processing completed |
| `document_path` | STRING | Cloud Storage path to document |
| `document_text` | STRING | Extracted text content |
| `document_summary` | STRING | AI-generated summary |

## Workflow Execution Details

### Entry Point Function

```python
@functions_framework.cloud_event
def on_cloud_event(event: CloudEvent) -> None:
    """Process a new document from an Eventarc event."""
```

### Processing Pipeline

```python
def process_document(
    event_id: str,
    input_bucket: str,
    filename: str,
    mime_type: str,
    time_uploaded: datetime,
    project: str,
    location: str,
    docai_processor_id: str,
    docai_location: str,
    output_bucket: str,
    bq_dataset: str,
    bq_table: str,
) -> None:
    """Main document processing pipeline."""
```

## Error Handling

The workflow implements exception handling at the top level:

```python
try:
    process_document(...)
except Exception as e:
    logging.exception(e, stack_info=True)
```

Errors are logged but do not propagate, ensuring the Cloud Function completes gracefully.

## Retry and Recovery

- **Eventarc**: Provides automatic retry for transient failures
- **Document AI**: Uses batch processing for reliability with large documents
- **BigQuery**: Supports idempotent row insertion

## Concurrency and Scaling

- **Cloud Functions**: Automatically scales based on incoming events
- **Document AI**: Batch processing handles queue management
- **BigQuery**: Scales for high-throughput data ingestion

## Monitoring and Observability

### Logging

The workflow emits structured logs at each stage:
- `📖 {event_id}: Getting document text`
- `📝 {event_id}: Summarizing document`
- `🗃️ {event_id}: Writing document summary to BigQuery`
- `✅ {event_id}: Done!`

### Metrics

Key metrics to monitor:
- Document processing latency
- OCR success rate
- Summary generation time
- BigQuery insertion success rate

## Configuration

### Environment Variables

| Variable | Description |
|----------|-------------|
| `PROJECT_ID` | Google Cloud project ID |
| `LOCATION` | Cloud region for Vertex AI |
| `OUTPUT_BUCKET` | Bucket for temporary files |
| `DOCAI_PROCESSOR` | Document AI processor ID |
| `DOCAI_LOCATION` | Document AI region |
| `BQ_DATASET` | BigQuery dataset name |
| `BQ_TABLE` | BigQuery table name |
| `LOG_EXECUTION_ID` | Enable execution ID logging |

## Related Documentation

- [Document Types](./DOCUMENT_TYPES.md)
- [Tool Calls](./TOOL_CALLS.md)
- [Training Data and Evaluations](./TRAINING_DATA_AND_EVALS.md)
- [Agentic Orchestration](./AGENTIC_ORCHESTRATION.md)
