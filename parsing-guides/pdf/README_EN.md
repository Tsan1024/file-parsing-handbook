# PDF Parsing Guide

English | [中文](README.md)

## Overview

PDF (Portable Document Format) is one of the most widely used document formats, but it was designed for "print output" rather than "data exchange," making PDF parsing a persistent challenge. This guide covers two major approaches: traditional protocol-based parsing and the two-stage VLM approach.

## Parsing Approaches

### 1. Protocol-Based Parsing

Traditional approach that reads PDF internal structures programmatically according to the PDF specification. Best suited for well-structured, non-scanned PDFs.

#### 1.1 Text Extraction

Extract characters and position information directly from the PDF content stream.

| Tool | Language | Characteristics |
|------|----------|-----------------|
| PyPDF2 / pypdf | Python | Lightweight, basic extraction, merge/split support |
| pdfplumber | Python | Precise character coordinates, font info, layout analysis |
| PyMuPDF (fitz) | Python | MuPDF-based, extremely fast, text/image/annotation extraction |
| pdf.js | JavaScript | Browser-side parsing, maintained by Mozilla |
| PDFBox | Java | Apache project, comprehensive features |

**Recommendation**: Choose PyMuPDF for speed, pdfplumber for precision and control.

#### 1.2 Table Extraction

Extracting tables from PDFs is notoriously difficult because PDFs store only scattered lines and text, not "table" semantics.

| Tool | Language | Method | Best For |
|------|----------|--------|----------|
| Camelot | Python | Line detection | Tables with visible borders |
| Tabula | Java/Python | Text position clustering | Borderless but aligned tables |
| pdfplumber | Python | Built-in table detection | Medium complexity tables |
| unstructured | Python | Rules + heuristics | General document parsing pipeline |

**Key insight**: Bordered tables are detected by finding regions enclosed by lines; borderless tables rely on column alignment and whitespace gaps to infer column boundaries.

#### 1.3 Layout Analysis

Understanding the physical layout of PDF pages (headings, paragraphs, columns, image positions, etc.).

| Tool | Description |
|------|-------------|
| pdfplumber | Character-level coordinates for building custom layout trees |
| PyMuPDF | Built-in block/line/span hierarchy |
| layoutparser | Deep learning-based layout analysis, multiple model backends |
| unstructured | Built-in partition_pdf, automatic element type detection |

#### 1.4 OCR (Scanned Document Processing)

For scanned PDFs (image-only), OCR is required before text extraction.

| Tool | Characteristics |
|------|-----------------|
| Tesseract + pdf2image | Classic open-source combination, convert to images then OCR |
| PaddleOCR | Excellent Chinese recognition, table structure recognition |
| Surya OCR | Next-gen OCR, 90+ languages, excellent layout detection |
| Azure/AWS/Google OCR | Cloud services, high accuracy, pay-per-use |

**Recommended pipeline**: pdf2image → layout detection → per-region OCR → structured output.

### 2. Two-Stage VLM Approach

A modern paradigm: render PDF pages as images, then use Vision Language Models (VLMs) to directly understand page content and produce structured output. Especially effective for complex layouts, mixed text/image content, and scenarios requiring semantic understanding.

#### 2.1 Core Pipeline

```
PDF Page → Render to Image (Stage 1) → VLM Analysis (Stage 2) → Structured Output (Markdown/JSON)
```

**Stage 1 — Page Rendering**: Convert PDF pages to high-resolution images.

| Tool | Description |
|------|-------------|
| pdf2image | Poppler-based, most common PDF→image tool in Python |
| PyMuPDF | Built-in page.get_pixmap(), no extra dependencies |
| ImageMagick | CLI approach, good for batch processing |

**Stage 2 — VLM Analysis**: Feed images to a Vision Language Model for content understanding and extraction.

| Model/Service | Characteristics |
|---------------|-----------------|
| GPT-4o / GPT-4V | Strong comprehension, handles complex tables and formulas |
| Claude 3.5 Sonnet / Opus | Long context, suitable for multi-page documents |
| Gemini 2.5 Flash / Pro | High throughput, low cost, native multimodal |
| Qwen-VL | Open-source, locally deployable |
| Molmo (Allen AI) | Open-source, optimized for document understanding |
| Docling (IBM) | VLM solution specialized for document conversion |

#### 2.2 Example Implementation

```python
from pdf2image import convert_from_path
import base64, io
from openai import OpenAI

client = OpenAI()

# Stage 1: Render to images
images = convert_from_path("document.pdf", dpi=200)

for i, img in enumerate(images):
    # Encode as PNG base64
    buf = io.BytesIO()
    img.save(buf, format="PNG")
    img_base64 = base64.b64encode(buf.getvalue()).decode()

    # Stage 2: VLM analysis
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{
            "role": "user",
            "content": [
                {"type": "text", "text": "Extract all text content from this page, preserve table structure, output in Markdown format."},
                {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{img_base64}"}}
            ]
        }]
    )
    print(response.choices[0].message.content)
```

#### 2.3 Approach Comparison

| Dimension | Protocol-Based Parsing | Two-Stage VLM |
|-----------|----------------------|---------------|
| Accuracy | High for structured PDFs, moderate for complex layouts | Generally high, good image-text understanding |
| Speed | Very fast (ms/page) | Slower (seconds/page) |
| Cost | Free (open-source libraries) | API: per-token billing; local models: GPU required |
| Table Handling | Depends on tool algorithm, good for bordered tables | Semantic-level understanding, good for complex tables |
| Scanned Documents | Requires additional OCR step | Natively supported (image input) |
| Multilingual Mixed | Depends on OCR engine | VLM-native multilingual |
| Offline Capable | Fully offline | API requires network; open-source models run offline |

**Recommendations**:
- Batch processing structured PDFs → Protocol-based parsing (PyMuPDF + Camelot)
- Complex layouts, scanned documents, semantic extraction needed → Two-stage VLM
- Mixed scenarios → Quick filtering with protocol parsing, VLM for difficult pages

## Common Tools Quick Reference

| Tool | Language | Type | Description |
|------|----------|------|-------------|
| PyMuPDF | Python | Protocol | High-speed text/image extraction |
| pdfplumber | Python | Protocol | Precise coordinate-level control |
| Camelot | Python | Protocol | Table extraction specialist |
| unstructured | Python | Hybrid | Unified document parsing pipeline |
| pdf2image + GPT-4o | Python | VLM Two-Stage | Semantic-level understanding |
| PaddleOCR | Python | OCR | Excellent for Chinese scenarios |
| Surya | Python | OCR/VLM | Next-gen open-source OCR |

## Example Code

See the [examples/](examples/) directory for runnable examples.
