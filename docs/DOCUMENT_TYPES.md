# Document Types

This document describes the document types supported by the Generative AI Document Summarization agent and how they are processed.

## Overview

The document summarization agent leverages Google Cloud Document AI's OCR capabilities to extract text from various document formats. The agent automatically detects and processes documents based on their MIME type.

## Supported Document Formats

### PDF Documents

| MIME Type | Extension | Description |
|-----------|-----------|-------------|
| `application/pdf` | `.pdf` | Portable Document Format files |

**Characteristics:**
- Most commonly used format for business documents
- Supports text-based and scanned documents
- Maximum pages per batch: 500 (via batch processing)
- Supports multi-page documents

**Best Practices:**
- Use native PDF (not scanned) for best OCR results
- Ensure documents are not password-protected
- Optimal resolution: 300 DPI for scanned documents

### Image Documents

| MIME Type | Extension | Description |
|-----------|-----------|-------------|
| `image/png` | `.png` | Portable Network Graphics |
| `image/jpeg` | `.jpg`, `.jpeg` | JPEG images |
| `image/gif` | `.gif` | Graphics Interchange Format |
| `image/tiff` | `.tiff`, `.tif` | Tagged Image File Format |
| `image/bmp` | `.bmp` | Bitmap Image File |
| `image/webp` | `.webp` | WebP image format |

**Characteristics:**
- Single-page document processing
- OCR optimized for printed and handwritten text
- Supports various color depths and resolutions

**Best Practices:**
- Use high-resolution images (300+ DPI)
- Ensure good contrast between text and background
- Avoid heavy compression that may degrade text quality

### Office Documents

| MIME Type | Extension | Description |
|-----------|-----------|-------------|
| `application/vnd.openxmlformats-officedocument.wordprocessingml.document` | `.docx` | Microsoft Word documents |
| `application/msword` | `.doc` | Legacy Word documents |

**Note:** Office documents may require conversion to PDF for optimal processing.

## Document Processing Details

### OCR Processor Configuration

The solution uses Document AI's OCR Processor (`OCR_PROCESSOR`) for text extraction:

```hcl
resource "google_document_ai_processor" "ocr" {
  project      = module.project_services.project_id
  location     = var.documentai_location
  display_name = local.ocr_processor_name
  type         = "OCR_PROCESSOR"
}
```

### Processing Pipeline

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Document      │────▶│   Document AI   │────▶│   Extracted     │
│   (Any Format)  │     │   OCR Engine    │     │   Text          │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

### Batch Processing vs Online Processing

The solution uses **batch processing** instead of online processing:

| Feature | Batch Processing | Online Processing |
|---------|-----------------|-------------------|
| Pages per document | Up to 500 | Up to 15 |
| Processing time | Asynchronous | Synchronous |
| Use case | Large documents | Quick responses |

**Batch Processing Code:**
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
                gcs_uri=f"gs://{temp_bucket}/ocr/{input_file.split('gs://')[-1]}",
            ),
        ),
    ),
)
```

## Document Structure Recognition

### Supported Elements

Document AI OCR extracts the following elements:

- **Text blocks**: Paragraphs and text regions
- **Lines**: Individual lines of text
- **Words**: Individual words and tokens
- **Characters**: Character-level recognition
- **Tables**: Tabular data structures
- **Forms**: Key-value pairs in forms

### Language Support

Document AI supports 200+ languages including:

- English
- Spanish
- French
- German
- Chinese (Simplified & Traditional)
- Japanese
- Korean
- Arabic
- Hindi
- And many more...

## Document Size Limits

| Limit Type | Value |
|------------|-------|
| Maximum file size | 20 MB (online), 1 GB (batch) |
| Maximum pages (online) | 15 pages |
| Maximum pages (batch) | 500 pages |
| Maximum resolution | 4000 x 4000 pixels |
| Minimum resolution | 50 x 50 pixels |

## Quality Requirements

For optimal OCR results:

1. **Resolution**: 300 DPI recommended for scanned documents
2. **Contrast**: Clear distinction between text and background
3. **Orientation**: Upright orientation preferred (auto-rotation available)
4. **Quality**: Minimal noise, smudges, or artifacts
5. **Fonts**: Standard fonts yield better results

## Document Preprocessing

While the agent processes documents directly, these preprocessing steps improve results:

### Recommended Preprocessing

1. **Deskewing**: Correct tilted scans
2. **Denoising**: Remove background noise
3. **Contrast enhancement**: Improve text visibility
4. **Resolution upscaling**: Enhance low-resolution documents

### Handling Special Cases

| Case | Recommendation |
|------|----------------|
| Handwritten text | Use higher resolution, clear handwriting |
| Mixed content | Process separately when possible |
| Complex layouts | Document AI handles multi-column layouts |
| Embedded images | Text within images is extracted |

## Error Scenarios

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Empty text extraction | Poor image quality | Improve source quality |
| Garbled text | Unsupported font/encoding | Convert to standard format |
| Partial extraction | Corrupt file | Re-upload valid file |
| Processing timeout | Very large document | Split into smaller parts |

## Testing Documents

For testing the document summarization pipeline, consider these document types:

1. **Research papers**: Multi-page PDF with complex formatting
2. **Business reports**: Tables, charts, and text
3. **Legal documents**: Dense text with specialized terminology
4. **Invoices**: Structured forms with key-value pairs
5. **Handwritten notes**: To test handwriting recognition

## Related Documentation

- [Agent Workflows](./AGENT_WORKFLOWS.md)
- [Tool Calls](./TOOL_CALLS.md)
- [Training Data and Evaluations](./TRAINING_DATA_AND_EVALS.md)
- [Agentic Orchestration](./AGENTIC_ORCHESTRATION.md)
