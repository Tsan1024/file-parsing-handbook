# PDF 解析指南

[English](README_EN.md) | 中文

## 概述

PDF（Portable Document Format）是最广泛使用的文档格式之一，但其设计初衷是「打印输出」而非「数据交换」，因此 PDF 解析一直是文件处理领域的难点。本指南涵盖两大类主流解析方案：传统协议解析和两阶段 VLM 方案。

## 解析方案总览

### 一、普通协议解析

传统方案依赖 PDF 协议规范，通过编程方式直接读取 PDF 内部结构来提取内容。适合结构化良好、非扫描版的 PDF。

#### 1.1 文本提取

直接从 PDF 的文本流（content stream）中提取字符及位置信息。

| 工具 | 语言 | 特点 |
|------|------|------|
| PyPDF2 / pypdf | Python | 轻量级，基础文本提取，支持合并/拆分 |
| pdfplumber | Python | 精确提取字符坐标、字体信息，适合布局分析 |
| PyMuPDF (fitz) | Python | 基于 MuPDF，速度极快，支持文本/图片/注释提取 |
| pdf.js | JavaScript | 浏览器端解析，Mozilla 维护 |
| PDFBox | Java | Apache 项目，功能全面 |

**选型建议**：追求速度选 PyMuPDF，追求精度和控制力选 pdfplumber。

#### 1.2 表格提取

从 PDF 中识别并抽取表格是公认的难点，因为 PDF 内部并不存储「表格」语义，只有零散的线段和文本。

| 工具 | 语言 | 方法 | 适用场景 |
|------|------|------|----------|
| Camelot | Python | 基于线条检测 | 有线框的表格 |
| Tabula | Java/Python | 基于文本位置聚类 | 无线框但有对齐的表格 |
| pdfplumber | Python | 内置表格检测 | 中等复杂度表格 |
| unstructured | Python | 规则 + 启发式 | 通用文档解析管道 |

**关键思路**：有线表格检测线条围成的区域；无线表格通过文本列对齐和空白间距推断列边界。

#### 1.3 布局分析

理解 PDF 页面的物理布局（标题、段落、分栏、图片位置等）。

| 工具 | 说明 |
|------|------|
| pdfplumber | 提供字符级坐标，可自行构建布局树 |
| PyMuPDF | 提供 block/line/span 层级结构 |
| layoutparser | 基于深度学习的布局分析，支持多模型后端 |
| unstructured | 内置 partition_pdf，自动识别文档元素类型 |

#### 1.4 OCR（扫描件处理）

对于扫描版 PDF（图片嵌入），需要先 OCR 识别文字。

| 工具 | 特点 |
|------|------|
| Tesseract + pdf2image | 开源经典方案，先转图片再 OCR |
| PaddleOCR | 中文识别优秀，支持表格结构识别 |
| Surya OCR | 新一代 OCR，支持 90+ 语言，布局检测优秀 |
| Azure/AWS/Google OCR | 云服务方案，准确率高，按量付费 |

**推荐流程**：pdf2image 转图片 → 布局检测 → 按区域 OCR → 结构化输出。

### 二、两阶段 VLM 方案

这是近年来兴起的新范式：将 PDF 页面渲染为图像，再使用视觉语言模型（VLM）直接理解页面内容并输出结构化结果。特别适合复杂排版、图文混排、需要语义理解的场景。

#### 2.1 核心流程

```
PDF 页面 → 渲染为图像 (Stage 1) → VLM 分析 (Stage 2) → 结构化输出 (Markdown/JSON)
```

**Stage 1 — 页面渲染**：将 PDF 页面转换为高分辨率图像。

| 工具 | 说明 |
|------|------|
| pdf2image | 基于 poppler，Python 中最常用的 PDF→图像 工具 |
| PyMuPDF | 内置 page.get_pixmap()，无需额外依赖 |
| ImageMagick | 命令行方案，适合批量处理 |

**Stage 2 — VLM 分析**：将图像送入视觉语言模型进行内容理解与提取。

| 模型/服务 | 特点 |
|-----------|------|
| GPT-4o / GPT-4V | 理解能力强，支持复杂表格和公式 |
| Claude 3.5 Sonnet / Opus | 长上下文，适合多页文档 |
| Gemini 2.5 Flash / Pro | 高吞吐，成本低，原生多模态 |
| Qwen-VL | 开源方案，可本地部署 |
| Molmo (Allen AI) | 开源，专为文档理解优化 |
| Docling (IBM) | 专门针对文档转换的 VLM 方案 |

#### 2.2 典型实现

```python
from pdf2image import convert_from_path
import base64, io
from openai import OpenAI

client = OpenAI()

# Stage 1: 渲染为图像
images = convert_from_path("document.pdf", dpi=200)

for i, img in enumerate(images):
    # 图像编码为 PNG base64
    buf = io.BytesIO()
    img.save(buf, format="PNG")
    img_base64 = base64.b64encode(buf.getvalue()).decode()

    # Stage 2: VLM 分析
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{
            "role": "user",
            "content": [
                {"type": "text", "text": "提取此页面的全部文字内容，保留表格结构，使用 Markdown 格式输出。"},
                {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{img_base64}"}}
            ]
        }]
    )
    print(response.choices[0].message.content)
```

#### 2.3 方案对比

| 维度 | 普通协议解析 | 两阶段 VLM |
|------|-------------|-----------|
| 准确性 | 结构化 PDF 高，复杂排版中等 | 整体较高，图文理解好 |
| 速度 | 极快（毫秒级/页） | 较慢（秒级/页） |
| 成本 | 免费（开源库） | API 按 token 计费，本地模型需 GPU |
| 表格处理 | 依赖工具算法，有线表格好 | 语义级理解，复杂表格表现好 |
| 扫描件 | 需额外 OCR 步骤 | 天然支持（图像输入） |
| 多语言混合 | 依赖 OCR 引擎 | VLM 原生多语言 |
| 可离线 | 完全离线 | API 方案需联网，开源模型可离线 |

**选型建议**：
- 批量处理结构化 PDF → 普通协议解析（PyMuPDF + Camelot）
- 复杂排版、扫描件、需要语义提取 → 两阶段 VLM
- 混合场景 → 先用协议解析快速筛选，疑难页面交给 VLM

## 常用工具速查

| 工具 | 语言 | 类型 | 说明 |
|------|------|------|------|
| PyMuPDF | Python | 协议解析 | 高速文本/图片提取 |
| pdfplumber | Python | 协议解析 | 精确坐标级控制 |
| Camelot | Python | 协议解析 | 表格提取专家 |
| unstructured | Python | 混合 | 统一的文档解析管道 |
| pdf2image + GPT-4o | Python | VLM 两阶段 | 语义级理解 |
| PaddleOCR | Python | OCR | 中文场景优秀 |
| Surya | Python | OCR/VLM | 新一代开源 OCR |

## 示例代码

查看 [examples/](examples/) 目录获取可运行示例。
