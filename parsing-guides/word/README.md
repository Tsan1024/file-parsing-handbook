# Word 文档解析指南

[English](README_EN.md) | 中文

## 概述

Word 文档（.docx）是 Microsoft Office 的核心文档格式，底层基于 Open XML 标准，本质是一个 ZIP 压缩包内包含多个 XML 文件。本指南涵盖常见的 DOCX 解析方法与工具。

## 内容

- [基础读取](basic-reading.md)
- [模板处理](template-handling.md)
- [格式转换](conversion.md)

## 解析方法总览

### 一、协议级解析（直接操作 XML）

DOCX 本质是一个 ZIP 文件，包含 `document.xml`（正文）、`styles.xml`（样式）、`relationships`（资源关联）等结构化 XML。直接解析 XML 可以获得最大灵活性。

#### 1.1 库封装解析

最常见的方案，库封装了 ZIP 解压和 XML 遍历细节，提供面向对象的 API。

| 工具 | 语言 | 特点 |
|------|------|------|
| python-docx | Python | 最流行的 Python Word 库，支持段落/表格/样式/图片读写 |
| Apache POI (XWPF) | Java | 企业级 Office 文档处理，功能最全面 |
| Open XML SDK | C#/.NET | 微软官方 SDK，底层 XML 操作 |
| docx.js | JavaScript | 浏览器和 Node.js 端的 Word 处理 |

**python-docx 典型用法**：

```python
from docx import Document

doc = Document("report.docx")

# 提取段落文本
for para in doc.paragraphs:
    print(para.text)

# 提取表格
for table in doc.tables:
    for row in table.rows:
        for cell in row.cells:
            print(cell.text)
```

#### 1.2 原始 XML 解析

当需要处理特殊结构（如自定义属性、复杂字段、内容控件）时，直接操作底层 XML 更灵活。

```python
import zipfile
from lxml import etree

# DOCX 即 ZIP
with zipfile.ZipFile("document.docx") as z:
    with z.open("word/document.xml") as f:
        tree = etree.parse(f)

# 定义 Word 命名空间
ns = {"w": "http://schemas.openxmlformats.org/wordprocessingml/2006/main"}

# XPath 提取所有文本
texts = tree.xpath("//w:t/text()", namespaces=ns)
print("".join(texts))
```

**适用场景**：自定义 XML 属性提取、内容控件（Content Control）、邮件合并域、复杂编号列表。

### 二、格式转换方案

将 DOCX 转换为中间格式再处理，是另一种常见策略。

| 工具 | 转换目标 | 适用场景 |
|------|----------|----------|
| pandoc | Markdown/HTML/LaTeX 等 | 通用文档格式转换 |
| mammoth | HTML/Markdown | 专为 Word→HTML 设计，样式映射优秀 |
| docx2txt | 纯文本 | 最简方案，纯文本提取 |
| libreoffice-headless | PDF/文本 | 命令行批量转换 |
| unoconv | 多种格式 | 基于 LibreOffice 的通用转换 |

**pandoc 示例**：

```bash
# DOCX → Markdown
pandoc input.docx -t markdown -o output.md

# DOCX → 纯文本
pandoc input.docx -t plain -o output.txt
```

**选型建议**：需要保留格式结构选 mammoth 或 pandoc；只需要文字内容选 docx2txt；批量转换选 libreoffice-headless。

### 三、VLM 图像方案

对于排版极其复杂的 Word 文档（如多栏布局、嵌入式图表、艺术字），可以先将 DOCX 转为图像，再用视觉语言模型（VLM）进行语义级解析。

#### 3.1 核心流程

```
DOCX → 渲染为图像 → VLM 分析 → 结构化输出
```

#### 3.2 实现示例

```python
import subprocess, base64, io
from pdf2image import convert_from_path
from openai import OpenAI

# 先通过 libreoffice 将 docx 转为 pdf
subprocess.run([
    "libreoffice", "--headless", "--convert-to", "pdf", "document.docx"
], check=True)

# 渲染为图像
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
                {"type": "text", "text": "提取此页全部内容，保留表格和层级结构，Markdown 输出。"},
                {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{img_b64}"}}
            ]
        }]
    )
    print(response.choices[0].message.content)
```

#### 3.3 适用场景

- 复杂排版：多栏、文本框、嵌入图表
- 扫描版或图片内嵌的 Word 文档
- 需要语义理解而非纯文本提取（如区分标题层级、正文、脚注）

### 四、方案对比

| 维度 | python-docx | 原始 XML | pandoc 转换 | VLM 图像 |
|------|------------|----------|------------|----------|
| 复杂度 | 低 | 中 | 低 | 高 |
| 灵活性 | 中 | 最高 | 低 | — |
| 格式保留 | 好 | 好 | 中 | 依赖模型 |
| 表格处理 | 好 | 自定义 | 中 | 好 |
| 图片处理 | 提取内嵌图 | 提取原始文件 | 有限 | 语义级理解 |
| 速度 | 快 | 快 | 快 | 慢 |
| 离线 | ✅ | ✅ | ✅ | 开源模型可离线 |
| 成本 | 免费 | 免费 | 免费 | API 收费 |

**推荐策略**：

1. **常规文档** → python-docx（简单高效，覆盖 90% 场景）
2. **复杂字段、自定义 XML** → lxml + 原始 XML 解析
3. **跨格式转换** → pandoc 或 mammoth
4. **极复杂排版、需要语义理解** → VLM 图像方案

## 常用工具速查

| 工具 | 语言 | 类型 | 说明 |
|------|------|------|------|
| python-docx | Python | 协议解析 | 段落/表格/样式/图片读写 |
| lxml + zipfile | Python | 原始 XML | 底层控制，处理自定义结构 |
| Apache POI | Java | 协议解析 | 企业级 Office 处理 |
| pandoc | Haskell/CLI | 格式转换 | 通用文档转换器 |
| mammoth | Python/JS/Java | 格式转换 | Word→HTML/Markdown 专业户 |
| libreoffice-headless | CLI | 渲染/转换 | 批量转换和图像渲染 |
| GPT-4o / Claude | API | VLM | 语义级文档理解 |

## 示例代码

查看 [examples/](examples/) 目录。
