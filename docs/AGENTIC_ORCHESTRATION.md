# Agentic Orchestration and Automation

This document provides comprehensive documentation on the sub-agents, orchestration patterns, and automation pipelines used in the Generative AI Document Summarization solution.

## Overview

The document summarization solution implements a multi-agent architecture where specialized components work together to process documents. This document covers the agent hierarchy, orchestration patterns, and CI/CD automation.

## Agent Architecture

### Agent Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Orchestration Layer                                │
│                          (Eventarc + Cloud Functions)                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│    ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐        │
│    │   OCR Agent      │  │ Summarization    │  │  Storage Agent   │        │
│    │  (Document AI)   │  │    Agent         │  │  (BigQuery)      │        │
│    │                  │  │  (Vertex AI)     │  │                  │        │
│    └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘        │
│             │                     │                     │                   │
│             ▼                     ▼                     ▼                   │
│    ┌──────────────────────────────────────────────────────────────┐        │
│    │                    Main Orchestrator Agent                    │        │
│    │                   (Webhook Cloud Function)                    │        │
│    └──────────────────────────────────────────────────────────────┘        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Sub-Agents

### 1. OCR Agent (Document AI)

**Purpose:** Extract text from documents using Optical Character Recognition.

**Responsibilities:**
- Receive document from Cloud Storage
- Process document through Document AI OCR processor
- Extract text content from all pages
- Handle various document formats (PDF, images)

**Interface:**
```python
def get_document_text(
    input_file: str,         # GCS URI of document
    mime_type: str,          # Document MIME type
    processor_id: str,       # Document AI processor ID
    temp_bucket: str,        # Temporary storage bucket
    docai_location: str,     # Document AI region
) -> Iterator[str]:
    """Extract text from document."""
```

**Configuration:**
| Parameter | Value | Description |
|-----------|-------|-------------|
| Processor Type | `OCR_PROCESSOR` | General OCR |
| Location | Configurable | us, eu |
| Max Pages | 500 | Per batch request |

### 2. Summarization Agent (Vertex AI)

**Purpose:** Generate concise summaries using Large Language Models.

**Responsibilities:**
- Receive extracted text from OCR agent
- Apply summarization prompt and instructions
- Generate coherent summary
- Return summary text

**Interface:**
```python
def summarize_text(
    text: str,               # Input text to summarize
    project: str,            # GCP project ID
    location: str,           # Vertex AI region
) -> str:
    """Generate summary using LLM."""
```

**Configuration:**
| Parameter | Value | Description |
|-----------|-------|-------------|
| Model | gemini-2.0-flash | Gemini model version |
| System Instruction | Summarization prompt | Guide output style |
| Temperature | Default | Controls creativity |

### 3. Storage Agent (BigQuery)

**Purpose:** Persist document metadata and summaries for analysis.

**Responsibilities:**
- Receive processed results
- Format data for BigQuery schema
- Insert records into table
- Handle insertion errors

**Interface:**
```python
def write_to_bigquery(
    event_id: str,           # Event identifier
    time_uploaded: datetime, # Upload timestamp
    doc_path: str,           # Document location
    doc_text: str,           # Extracted text
    doc_summary: str,        # Generated summary
    project: str,            # GCP project ID
    bq_dataset: str,         # Dataset name
    bq_table: str,           # Table name
) -> None:
    """Store results in BigQuery."""
```

**Schema:**
```json
[
  {"name": "event_id", "type": "STRING"},
  {"name": "time_uploaded", "type": "TIMESTAMP"},
  {"name": "time_processed", "type": "TIMESTAMP"},
  {"name": "document_path", "type": "STRING"},
  {"name": "document_text", "type": "STRING"},
  {"name": "document_summary", "type": "STRING"}
]
```

### 4. Event Router Agent (Eventarc)

**Purpose:** Route storage events to processing functions.

**Responsibilities:**
- Monitor Cloud Storage bucket for new objects
- Filter events by bucket and event type
- Trigger Cloud Function with event data
- Manage retry and delivery

**Configuration:**
```hcl
resource "google_eventarc_trigger" "trigger" {
  matching_criteria {
    attribute = "type"
    value     = "google.cloud.storage.object.v1.finalized"
  }
  matching_criteria {
    attribute = "bucket"
    value     = google_storage_bucket.docs.name
  }
  destination {
    cloud_run_service {
      service = google_cloudfunctions2_function.webhook.name
      region  = var.region
    }
  }
}
```

## Orchestration Patterns

### Event-Driven Orchestration

The solution uses event-driven architecture for loose coupling:

```
Document Upload → Eventarc → Cloud Function → Sub-agents → BigQuery
```

**Benefits:**
- Automatic scaling
- Fault tolerance
- Loose coupling between components
- Asynchronous processing

### Sequential Pipeline Pattern

Sub-agents execute in sequence within the orchestrator:

```python
def process_document(...):
    # Stage 1: OCR Agent
    doc_text = get_document_text(...)
    
    # Stage 2: Summarization Agent
    doc_summary = summarize_text(...)
    
    # Stage 3: Storage Agent
    write_to_bigquery(...)
```

### Error Handling Pattern

```python
@functions_framework.cloud_event
def on_cloud_event(event: CloudEvent) -> None:
    try:
        process_document(...)
    except Exception as e:
        logging.exception(e, stack_info=True)
        # Event is not re-delivered (function completes)
```

## Service Accounts and IAM

### Webhook Service Account

```hcl
resource "google_service_account" "webhook" {
  account_id   = local.webhook_sa_name
  display_name = "Doc summary webhook"
}

resource "google_project_iam_member" "webhook" {
  for_each = toset([
    "roles/aiplatform.serviceAgent",
    "roles/bigquery.dataEditor",
    "roles/documentai.apiUser",
  ])
  role = each.key
}
```

### Eventarc Service Account

```hcl
resource "google_service_account" "trigger" {
  account_id   = local.trigger_sa_name
  display_name = "Doc summary Eventarc trigger"
}

resource "google_project_iam_member" "trigger" {
  for_each = toset([
    "roles/eventarc.eventReceiver",
    "roles/run.invoker",
  ])
  role = each.key
}
```

## CI/CD Automation

### GitHub Actions Workflows

#### 1. Lint Workflow (`.github/workflows/lint.yaml`)

**Triggers:**
- Pull request to main branch
- Workflow dispatch (manual)

**Jobs:**
1. **lint**: Terraform and code linting
2. **commitlint**: Commit message validation

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker run --rm -v ${{ github.workspace }}:/workspace \
               ${{ steps.variables.outputs.dev-tools }} /usr/local/bin/test_lint.sh
  
  commitlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install commitlint
        run: npm install -D @commitlint/cli @commitlint/config-conventional
      - name: Validate PR commits
        run: echo "$TITLE" | npx commitlint --verbose
```

#### 2. Webhook Workflow (`.github/workflows/webhook.yml`)

**Triggers:**
- Push to main (webhook/** or *.tf changes)
- Pull request to main
- Daily schedule (UTC 18:00)
- Workflow dispatch

**Jobs:**
1. **lint**: Python code linting with ruff
2. **test**: End-to-end tests with Google Cloud authentication

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
      - name: Install linter
        run: pip install ruff
      - name: Run linter
        run: ruff check .

  test:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/.../workloadIdentityPools/github-actions/providers/github-actions
          service_account: github-actions@project.iam.gserviceaccount.com
      - name: Run tests
        run: pytest --verbose -s
```

#### 3. Release Please (`.github/release-please.yml`)

Automates version management and changelog generation.

#### 4. Renovate (`.github/renovate.json`)

Automates dependency updates.

### Terraform Deployment Automation

#### Deploy Script (`deploy.sh`)

```bash
#!/bin/bash
terraform init
terraform plan -var="project_id=${PROJECT_ID}"
terraform apply -auto-approve -var="project_id=${PROJECT_ID}"
```

#### Infrastructure as Code

```hcl
# main.tf - Core infrastructure
module "project_services" { ... }
resource "google_storage_bucket" "main" { ... }
resource "google_storage_bucket" "docs" { ... }
resource "google_cloudfunctions2_function" "webhook" { ... }
resource "google_eventarc_trigger" "trigger" { ... }
resource "google_document_ai_processor" "ocr" { ... }
resource "google_bigquery_dataset" "main" { ... }
resource "google_bigquery_table" "main" { ... }
```

## Monitoring and Observability

### Logging Structure

```python
print(f"📖 {event_id}: Getting document text")
print(f"📝 {event_id}: Summarizing document")
print(f"  - Text length:    {len(doc_text)} characters")
print(f"  - Summary length: {len(doc_summary)} characters")
print(f"🗃️ {event_id}: Writing document summary to BigQuery")
print(f"✅ {event_id}: Done!")
```

### Metrics Collection Points

| Metric | Source | Purpose |
|--------|--------|---------|
| Document upload rate | Eventarc | Traffic monitoring |
| OCR processing time | Document AI | Performance |
| Summarization latency | Vertex AI | Model performance |
| BigQuery insert success | BigQuery | Data reliability |
| Function execution time | Cloud Functions | Cost optimization |

### Alerting Configuration

Set up alerts for:
- Function error rate > 1%
- Processing latency > 60 seconds
- BigQuery insertion failures
- Document AI quota usage

## Scaling and Performance

### Auto-scaling Configuration

```hcl
resource "google_cloudfunctions2_function" "webhook" {
  service_config {
    available_memory = "1G"
    # Cloud Functions automatically scales based on incoming events
  }
}
```

### Concurrency Considerations

| Component | Concurrency | Notes |
|-----------|-------------|-------|
| Cloud Functions | Auto-scaled | Up to 1000 instances |
| Document AI | Batch | Queue-based processing |
| Vertex AI | Rate-limited | Per-project quotas |
| BigQuery | Streaming | 10K rows/sec per table |

## Extension Points

### Adding New Sub-Agents

1. **Define Interface:**
```python
def new_agent(
    input_data: Any,
    config: dict,
) -> Any:
    """New agent description."""
    pass
```

2. **Integrate with Orchestrator:**
```python
def process_document(...):
    # Existing agents...
    
    # New agent integration
    result = new_agent(doc_summary, config)
```

3. **Add Required Permissions:**
```hcl
resource "google_project_iam_member" "webhook" {
  for_each = toset([
    # Existing roles...
    "roles/newservice.user",
  ])
}
```

### Custom Summarization Strategies

```python
STRATEGIES = {
    'default': {
        'instruction': "Give me a summary...",
        'config': {'temperature': 0.7},
    },
    'executive': {
        'instruction': "Create an executive summary...",
        'config': {'temperature': 0.3},
    },
    'technical': {
        'instruction': "Create a technical summary...",
        'config': {'temperature': 0.5},
    },
}
```

## Best Practices

### Agent Design Principles

1. **Single Responsibility:** Each agent has one clear purpose
2. **Loose Coupling:** Agents communicate through well-defined interfaces
3. **Idempotency:** Operations can be safely retried
4. **Observability:** Comprehensive logging and metrics

### Deployment Best Practices

1. Use unique resource names for multi-environment deployments
2. Enable proper IAM roles with least privilege
3. Implement comprehensive testing before deployment
4. Use infrastructure as code for reproducibility

### Security Considerations

1. **Service Account Isolation:** Separate accounts for each component
2. **Least Privilege:** Minimal required permissions
3. **Network Security:** VPC and firewall configuration
4. **Data Protection:** Encryption at rest and in transit

## Related Documentation

- [Agent Workflows](./AGENT_WORKFLOWS.md)
- [Document Types](./DOCUMENT_TYPES.md)
- [Tool Calls](./TOOL_CALLS.md)
- [Training Data and Evaluations](./TRAINING_DATA_AND_EVALS.md)
