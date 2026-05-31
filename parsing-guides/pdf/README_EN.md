# PDF Parsing Guide

English | [中文](README.md)

## Overview

PDF (Portable Document Format) is one of the most widely used document formats, but it was designed for "print output" rather than "data exchange," making PDF parsing a persistent challenge. This guide covers two major approaches: traditional protocol-based parsing and vision-model-based approaches.

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

### 2. Vision-Model-Based Approaches

Leverage Vision Language Models (VLMs) to understand PDF page content. Especially effective for complex layouts, mixed text/image content, and scenarios requiring semantic understanding. Approaches are divided by whether layout detection is performed explicitly.

#### 2.1 End-to-End VLM

The simplest VLM approach: render PDF pages as high-resolution images, feed the entire page to a VLM, and let the model understand the layout and extract content in one pass.

**Core pipeline**:

```
PDF Page → Render to Image → VLM Full-Page Analysis → Structured Output (Markdown/JSON)
```

**Page Rendering Tools**:

| Tool | Description |
|------|-------------|
| pdf2image | Poppler-based, most common PDF-to-image tool in Python |
| PyMuPDF | Built-in page.get_pixmap(), no extra dependencies |
| ImageMagick | CLI approach, good for batch processing |

**Common VLM Models**:

| Model/Service | Characteristics |
|---------------|-----------------|
| GPT-4o / GPT-4V | Strong comprehension, handles complex tables and formulas |
| Claude 3.5 Sonnet / Opus | Long context, suitable for multi-page documents |
| Gemini 2.5 Flash / Pro | High throughput, low cost, native multimodal |
| Qwen-VL | Open-source, locally deployable |

**Example**:

```python
from pdf2image import convert_from_path
import base64, io
from openai import OpenAI

client = OpenAI()

# Render to images
images = convert_from_path("document.pdf", dpi=200)

for img in images:
    # Encode as PNG base64
    buf = io.BytesIO()
    img.save(buf, format="PNG")
    img_base64 = base64.b64encode(buf.getvalue()).decode()

    # End-to-end VLM analysis
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

**Pros & cons**: Simple to implement, one API call per page. Limited by context window for very large pages or multi-page documents, and no fine-grained control per region.

#### 2.2 Two-Stage: Layout Detection → Regional Extraction

This is the classic two-stage paradigm in document parsing — separating "where is the content?" from "what is the content?", allowing each stage to use the most suitable model.

| Stage | Goal | Methods |
|-------|------|---------|
| **Stage 1: Layout Detection** | Identify all element regions on the page (headings, paragraphs, tables, figures, headers, footers, formulas, etc.) with bounding boxes and types | Rule-based (pdfplumber coordinate analysis), deep learning detection models (layoutparser, DocTR, Surya), lightweight object detection (YOLO-based), MinerU (doclayout-yolo) |
| **Stage 2: Regional Extraction** | For each detected region, select the best model for content extraction and structuring | Text regions → VLM or OCR; Table regions → table-specific VLM or Camelot/Tabula; Figure regions → VLM description or OCR; Formula regions → LaTeX conversion models (MinerU uses UnimeRNet) |

**Core pipeline**:

```
PDF Page → Render to Image → Layout Detection (Stage 1) → Regional Extraction (Stage 2) → Structured Output
```


**Representative Tool: MinerU**

MinerU is the flagship open-source implementation of the two-stage paradigm, developed by OpenDataLab (Shanghai AI Laboratory):

- **Stage 1**: Uses doclayout-yolo for document layout detection, identifying headings, body text, tables, formulas, and figures
- **Stage 2**: Table regions → Markdown/HTML tables via table recognition models; Formula regions → LaTeX via UnimeRNet; Text regions → OCR with reading order preservation

MinerU excels on academic papers, research reports, and financial statements, and is fully open-source with offline deployment support.

**Two-Stage vs End-to-End**:

| Dimension | End-to-End VLM | Two-Stage (Layout→Extraction) |
|-----------|---------------|-------------------------------|
| Complexity | Low, single call | Medium, requires model composition |
| Flexibility | Relies on VLM judgment | Best strategy per region |
| Table Accuracy | Model-dependent, may vary | Dedicated table extractors |
| Large Pages | Limited by context window | Region-level, no limit |
| Cost Control | Fixed token cost per page | Can skip low-value regions |

**Recommendations**:
- Quick prototypes, simple documents → End-to-end VLM
- Production, high precision, complex tables → Two-stage approach
- Hybrid: end-to-end for rough pass + two-stage for refinement

## Overall Comparison

| Dimension | Protocol-Based | End-to-End VLM | Two-Stage (Layout→Extraction) |
|-----------|---------------|----------------|-------------------------------|
| Accuracy | High for structured PDFs, moderate otherwise | High overall, good image-text | Highest, optimal per region |
| Speed | Very fast (ms/page) | Slower (seconds/page) | Slower, but optimizable |
| Cost | Free (open-source) | API per-token billing | Mix open-source + API |
| Scanned Docs | Requires OCR step | Native support | Native support |
| Offline | Fully offline | API requires network | Fully offline possible (open-source detection + VLM) |

**Recommended Strategy**:
- Batch structured PDFs → Protocol-based (PyMuPDF + Camelot)
- Complex layouts, semantic understanding needed → Two-stage (layout detection + VLM)
- Quick validation, prototyping → End-to-end VLM

## Common Tools Quick Reference

| Tool | Language | Type | Description |
|------|----------|------|-------------|
| PyMuPDF | Python | Protocol | High-speed text/image extraction |
| pdfplumber | Python | Protocol | Precise coordinate-level control |
| Camelot | Python | Protocol | Table extraction specialist |
| layoutparser | Python | Layout Detection | Deep learning layout analysis |
| Surya | Python | Layout+OCR | Next-gen open-source solution |
| MinerU | Python | Two-Stage | Open-source benchmark, layout detection + table/formula/text extraction |
| PaddleOCR | Python | OCR | Excellent for Chinese scenarios |
| GPT-4o / Claude | API | VLM | End-to-end document understanding |
