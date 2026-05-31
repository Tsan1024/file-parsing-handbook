# Word Document Parsing Guide

English | [中文](README.md)

## Overview

Word documents (.docx) are the core document format of Microsoft Office, built on the Open XML standard. A .docx file is essentially a ZIP archive containing multiple XML files. This guide covers common DOCX parsing approaches and tools.

## Contents

- [Basic Reading](basic-reading.md)
- [Template Processing](template-handling.md)
- [Format Conversion](conversion.md)

## Parsing Approaches

### 1. Protocol-Level Parsing (Direct XML Access)

A DOCX file is a ZIP archive containing structured XML files: `document.xml` (body), `styles.xml` (styles), `relationships` (resource references), and more. Direct XML parsing offers maximum flexibility.

#### 1.1 Library-Based Parsing

The most common approach: libraries encapsulate ZIP extraction and XML traversal, providing object-oriented APIs.

| Tool | Language | Characteristics |
|------|----------|-----------------|
| python-docx | Python | Most popular Python Word library; paragraph/table/style/image read/write |
| Apache POI (XWPF) | Java | Enterprise-grade Office document processing, most comprehensive |
| Open XML SDK | C#/.NET | Official Microsoft SDK, low-level XML operations |
| docx.js | JavaScript | Browser and Node.js Word processing |

**python-docx example**:

```python
from docx import Document

doc = Document("report.docx")

# Extract paragraph text
for para in doc.paragraphs:
    print(para.text)

# Extract tables
for table in doc.tables:
    for row in table.rows:
        for cell in row.cells:
            print(cell.text)
```

#### 1.2 Raw XML Parsing

For special structures (custom properties, complex fields, content controls), direct XML manipulation offers more flexibility.

```python
import zipfile
from lxml import etree

# DOCX is a ZIP
with zipfile.ZipFile("document.docx") as z:
    with z.open("word/document.xml") as f:
        tree = etree.parse(f)

# Word namespace
ns = {"w": "http://schemas.openxmlformats.org/wordprocessingml/2006/main"}

# XPath to extract all text
texts = tree.xpath("//w:t/text()", namespaces=ns)
print("".join(texts))
```

**Use cases**: custom XML attribute extraction, Content Controls, mail merge fields, complex numbered lists.

### 2. Format Conversion Approach

Converting DOCX to an intermediate format is another common strategy.

| Tool | Target Format | Best For |
|------|--------------|----------|
| pandoc | Markdown/HTML/LaTeX etc. | General-purpose document conversion |
| mammoth | HTML/Markdown | Word-to-HTML specialist, excellent style mapping |
| docx2txt | Plain text | Simplest option, plain text extraction |
| libreoffice-headless | PDF/Text | CLI batch conversion |
| unoconv | Multiple formats | LibreOffice-based universal conversion |

**pandoc example**:

```bash
# DOCX to Markdown
pandoc input.docx -t markdown -o output.md

# DOCX to Plain text
pandoc input.docx -t plain -o output.txt
```

**Recommendation**: mammoth or pandoc for preserving formatting; docx2txt for text-only; libreoffice-headless for batch conversion.

### 3. VLM Image-Based Approach

For highly complex Word documents (multi-column layouts, embedded charts, WordArt), you can render DOCX to images and use Vision Language Models (VLMs) for semantic-level parsing.

#### 3.1 Core Pipeline

```
DOCX → Render to Images → VLM Analysis → Structured Output
```

#### 3.2 Example Implementation

```python
import subprocess, base64, io
from pdf2image import convert_from_path
from openai import OpenAI

# Convert DOCX to PDF via libreoffice
subprocess.run([
    "libreoffice", "--headless", "--convert-to", "pdf", "document.docx"
], check=True)

# Render to images
images = convert_from_path("document.pdf", dpi=200)

client = OpenAI()
for img in images:
    buf = io.BytesIO()
    img.save(buf, format="PNG")
    img_b64 = base64.b64encode(buf.getvalue()).decode()

    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{
            "role": "user",
            "content": [
                {"type": "text", "text": "Extract all content from this page. Preserve tables and heading hierarchy. Output in Markdown."},
                {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{img_b64}"}}
            ]
        }]
    )
    print(response.choices[0].message.content)
```

#### 3.3 Use Cases

- Complex layouts: multi-column, text boxes, embedded charts
- Scanned or image-embedded Word documents
- Scenarios requiring semantic understanding beyond plain text extraction

### 4. Approach Comparison

| Dimension | python-docx | Raw XML | pandoc Convert | VLM Image |
|-----------|------------|---------|----------------|-----------|
| Complexity | Low | Medium | Low | High |
| Flexibility | Medium | Highest | Low | N/A |
| Format Preservation | Good | Good | Medium | Model-dependent |
| Table Handling | Good | Custom | Medium | Good |
| Image Handling | Extract embedded | Extract raw files | Limited | Semantic-level |
| Speed | Fast | Fast | Fast | Slow |
| Offline | Yes | Yes | Yes | Open-source models possible |
| Cost | Free | Free | Free | API: paid |

**Recommended Strategy**:

1. **Standard documents**: python-docx (simple and efficient, covers 90% of cases)
2. **Complex fields, custom XML**: lxml + raw XML parsing
3. **Cross-format conversion**: pandoc or mammoth
4. **Highly complex layout, semantic understanding needed**: VLM image approach

## Common Tools Quick Reference

| Tool | Language | Type | Description |
|------|----------|------|-------------|
| python-docx | Python | Protocol | Paragraph/table/style/image read/write |
| lxml + zipfile | Python | Raw XML | Low-level control, custom structures |
| Apache POI | Java | Protocol | Enterprise Office processing |
| pandoc | Haskell/CLI | Format Conversion | Universal document converter |
| mammoth | Python/JS/Java | Format Conversion | Word-to-HTML/Markdown specialist |
| libreoffice-headless | CLI | Rendering/Conversion | Batch conversion and image rendering |
| GPT-4o / Claude | API | VLM | Semantic-level document understanding |

## Example Code

See the [examples/](examples/) directory.
