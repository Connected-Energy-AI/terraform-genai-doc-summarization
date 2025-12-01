# Training Data and Evaluations

This document provides comprehensive guidance on training data requirements, evaluation methodologies, and continuous improvement strategies for the Generative AI Document Summarization agent.

## Overview

The document summarization agent leverages pre-trained Large Language Models (LLMs) through Vertex AI. While the base model (Gemini 2.0 Flash) comes pre-trained, this document covers fine-tuning approaches, evaluation frameworks, and quality assessment methodologies.

## Model Architecture

### Current Configuration

| Component | Value |
|-----------|-------|
| Base Model | Gemini 2.0 Flash |
| Model Type | Large Language Model (LLM) |
| Training Type | Pre-trained with optional fine-tuning |
| Inference Mode | Zero-shot with system instructions |

### System Instruction

```python
system_instruction=[
    "Give me a summary of the following text."
    "Use simple language and give examples."
]
```

## Training Data

### Pre-training Data (Base Model)

The Gemini 2.0 Flash model is pre-trained on diverse datasets including:

- Web documents and articles
- Books and publications
- Code repositories
- Scientific papers
- Multilingual content

### Fine-tuning Dataset Requirements

If fine-tuning is required for domain-specific summarization:

#### Dataset Structure

```json
{
  "training_samples": [
    {
      "input": "<full_document_text>",
      "output": "<expected_summary>",
      "metadata": {
        "document_type": "research_paper|legal|business|technical",
        "language": "en",
        "domain": "specific_domain"
      }
    }
  ]
}
```

#### Dataset Size Recommendations

| Fine-tuning Type | Minimum Samples | Recommended |
|------------------|-----------------|-------------|
| Basic adaptation | 100 | 500+ |
| Domain specialization | 500 | 2,000+ |
| Production quality | 2,000 | 10,000+ |

### Data Collection Sources

1. **Internal Documents**
   - Business reports with executive summaries
   - Research papers with abstracts
   - Legal documents with case summaries

2. **Public Datasets**
   - arXiv papers (abstracts as summaries)
   - CNN/DailyMail dataset
   - XSum dataset
   - Multi-News dataset

3. **Synthetic Data Generation**
   - Use larger models to generate training pairs
   - Human validation of synthetic summaries

### Data Quality Requirements

| Requirement | Description |
|-------------|-------------|
| Completeness | Full document text and corresponding summary |
| Accuracy | Summaries must accurately represent content |
| Consistency | Similar documents should have similar summary styles |
| Diversity | Cover various document types and lengths |
| Balance | Even distribution across categories |

### Data Preprocessing Pipeline

```python
def preprocess_training_sample(document: str, summary: str) -> dict:
    """Preprocess a single training sample."""
    return {
        "input": clean_text(document),
        "output": normalize_summary(summary),
        "input_tokens": count_tokens(document),
        "output_tokens": count_tokens(summary),
    }

def validate_sample(sample: dict) -> bool:
    """Validate training sample quality."""
    return (
        len(sample["input"]) > 100 and
        len(sample["output"]) > 50 and
        sample["output_tokens"] < sample["input_tokens"] * 0.3
    )
```

## Evaluation Framework

### Automated Metrics

#### 1. ROUGE Scores

Recall-Oriented Understudy for Gisting Evaluation:

```python
from rouge_score import rouge_scorer

scorer = rouge_scorer.RougeScorer(['rouge1', 'rouge2', 'rougeL'])

def evaluate_rouge(generated: str, reference: str) -> dict:
    """Calculate ROUGE scores."""
    scores = scorer.score(reference, generated)
    return {
        'rouge1': scores['rouge1'].fmeasure,
        'rouge2': scores['rouge2'].fmeasure,
        'rougeL': scores['rougeL'].fmeasure,
    }
```

**Target Scores:**
| Metric | Minimum | Good | Excellent |
|--------|---------|------|-----------|
| ROUGE-1 | 0.35 | 0.45 | 0.55+ |
| ROUGE-2 | 0.15 | 0.20 | 0.25+ |
| ROUGE-L | 0.30 | 0.40 | 0.50+ |

#### 2. BERTScore

Semantic similarity using BERT embeddings:

```python
from bert_score import score

def evaluate_bertscore(generated: str, reference: str) -> dict:
    """Calculate BERTScore."""
    P, R, F1 = score([generated], [reference], lang="en")
    return {
        'precision': P.item(),
        'recall': R.item(),
        'f1': F1.item(),
    }
```

#### 3. Compression Ratio

```python
def compression_ratio(document: str, summary: str) -> float:
    """Calculate compression ratio."""
    return len(summary) / len(document)
```

**Target Range:** 0.05 - 0.20 (5-20% of original length)

### Human Evaluation Criteria

#### Evaluation Rubric

| Criterion | Score | Description |
|-----------|-------|-------------|
| **Accuracy** | 1-5 | Factual correctness of summary |
| **Coverage** | 1-5 | Key points from document included |
| **Coherence** | 1-5 | Logical flow and readability |
| **Conciseness** | 1-5 | Appropriate length without redundancy |
| **Fluency** | 1-5 | Grammar and language quality |

#### Evaluation Template

```markdown
## Summary Evaluation

**Document ID:** ___________
**Evaluator:** ___________
**Date:** ___________

### Scores (1-5)

| Criterion | Score | Notes |
|-----------|-------|-------|
| Accuracy | | |
| Coverage | | |
| Coherence | | |
| Conciseness | | |
| Fluency | | |

### Overall Assessment
- [ ] Acceptable for production
- [ ] Needs improvement
- [ ] Unacceptable

### Comments:
_________________________
```

### Automated Evaluation Pipeline

```python
class SummaryEvaluator:
    """Automated summary evaluation pipeline."""
    
    def __init__(self):
        self.rouge_scorer = rouge_scorer.RougeScorer(['rouge1', 'rouge2', 'rougeL'])
    
    def evaluate(self, generated: str, reference: str, document: str) -> dict:
        """Run full evaluation suite."""
        return {
            'rouge': self._evaluate_rouge(generated, reference),
            'compression': self._compression_ratio(document, generated),
            'length': len(generated),
            'sentence_count': generated.count('.'),
        }
    
    def batch_evaluate(self, samples: list[dict]) -> dict:
        """Evaluate batch of samples and aggregate metrics."""
        results = [
            self.evaluate(s['generated'], s['reference'], s['document'])
            for s in samples
        ]
        return self._aggregate_metrics(results)
```

## Testing Infrastructure

### End-to-End Test

Located in `webhook/test_e2e.py`:

```python
def test_end_to_end(terraform_outputs: dict[str, str]) -> None:
    """End-to-end test for document summarization."""
    process_document(
        event_id=f"webhook-test-{uuid4().hex[:6]}",
        input_bucket="arxiv-dataset",
        filename="arxiv/cmp-lg/pdf/9410/9410009v1.pdf",
        mime_type="application/pdf",
        time_uploaded=datetime.now(),
        project=PROJECT_ID,
        location=LOCATION,
        docai_processor_id=docai_processor_id,
        docai_location="us",
        output_bucket=output_bucket,
        bq_dataset=bq_dataset,
        bq_table=bq_table,
    )
    
    # Verify results in BigQuery
    rows = bq_client.list_rows(table)
    assert len(list(rows)) > 0
```

### Unit Tests

```python
def test_summarization_quality():
    """Test summary quality metrics."""
    document = "Long document text..."
    summary = generate_summary(document)
    
    # Check compression ratio
    assert 0.05 < len(summary) / len(document) < 0.3
    
    # Check summary contains key terms
    key_terms = extract_key_terms(document)
    for term in key_terms[:5]:
        assert term.lower() in summary.lower()

def test_edge_cases():
    """Test edge case handling."""
    # Empty document
    assert generate_summary("") == ""
    
    # Very short document
    short_doc = "Hello world."
    summary = generate_summary(short_doc)
    assert summary == short_doc  # Should not over-summarize
    
    # Non-English content
    foreign_doc = "Bonjour le monde..."
    summary = generate_summary(foreign_doc)
    assert len(summary) > 0
```

## Continuous Improvement

### Feedback Collection

```python
class FeedbackCollector:
    """Collect and store user feedback on summaries."""
    
    def record_feedback(
        self,
        event_id: str,
        rating: int,
        feedback_type: str,
        comment: str = None
    ) -> None:
        """Record user feedback."""
        feedback = {
            'event_id': event_id,
            'rating': rating,
            'feedback_type': feedback_type,
            'comment': comment,
            'timestamp': datetime.now(),
        }
        self._store_feedback(feedback)
```

### A/B Testing Framework

```python
class ABTestRunner:
    """Run A/B tests on different prompts or models."""
    
    def __init__(self, variants: list[dict]):
        self.variants = variants
    
    def select_variant(self, user_id: str) -> dict:
        """Select variant based on user ID hash."""
        variant_index = hash(user_id) % len(self.variants)
        return self.variants[variant_index]
    
    def analyze_results(self) -> dict:
        """Analyze A/B test results."""
        return {
            variant['name']: self._calculate_metrics(variant)
            for variant in self.variants
        }
```

### Model Versioning

```python
MODEL_VERSIONS = {
    'v1': {
        'model': 'gemini-2.0-flash',
        'system_instruction': "Give me a summary...",
        'config': {'temperature': 0.7},
    },
    'v2': {
        'model': 'gemini-2.0-flash',
        'system_instruction': "Create a concise executive summary...",
        'config': {'temperature': 0.5},
    },
}
```

## Benchmark Datasets

### Recommended Benchmarks

| Dataset | Description | Size | Use Case |
|---------|-------------|------|----------|
| CNN/DailyMail | News articles | 300K | News summarization |
| XSum | BBC articles | 227K | Extreme summarization |
| arXiv | Scientific papers | 1.2M | Technical summarization |
| PubMed | Medical papers | 200K | Domain-specific |
| MultiNews | Multi-document | 56K | Multi-doc summarization |

### Custom Benchmark Creation

```python
def create_benchmark(
    documents: list[str],
    reference_summaries: list[str],
    metadata: list[dict]
) -> dict:
    """Create custom benchmark dataset."""
    benchmark = {
        'name': 'custom_benchmark_v1',
        'created': datetime.now().isoformat(),
        'samples': [
            {
                'document': doc,
                'reference': ref,
                'metadata': meta,
            }
            for doc, ref, meta in zip(documents, reference_summaries, metadata)
        ],
        'statistics': {
            'total_samples': len(documents),
            'avg_doc_length': sum(len(d) for d in documents) / len(documents),
            'avg_summary_length': sum(len(s) for s in reference_summaries) / len(reference_summaries),
        },
    }
    return benchmark
```

## Performance Monitoring

### Key Performance Indicators (KPIs)

| KPI | Target | Measurement |
|-----|--------|-------------|
| ROUGE-L Score | > 0.40 | Weekly evaluation |
| User Rating | > 4.0/5.0 | Continuous feedback |
| Processing Latency | < 30s | Real-time monitoring |
| Error Rate | < 1% | Daily monitoring |
| Coverage Rate | > 85% | Periodic evaluation |

### Monitoring Dashboard

Track the following metrics:
- Summary generation success rate
- Average processing time
- User feedback distribution
- Error categorization
- Model version performance comparison

## Related Documentation

- [Agent Workflows](./AGENT_WORKFLOWS.md)
- [Document Types](./DOCUMENT_TYPES.md)
- [Tool Calls](./TOOL_CALLS.md)
- [Agentic Orchestration](./AGENTIC_ORCHESTRATION.md)
