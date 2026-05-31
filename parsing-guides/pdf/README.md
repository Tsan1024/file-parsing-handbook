# PDF 解析指南

[English](README_EN.md) | 中文

## 概述

PDF（Portable Document Format）是最广泛使用的文档格式之一，但其设计初衷是「打印输出」而非「数据交换」，因此 PDF 解析一直是文件处理领域的难点。本指南涵盖两大类主流解析方案：传统协议解析和基于视觉模型的方案。

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

### 二、基于视觉模型的方案

利用视觉语言模型（VLM）理解 PDF 页面内容，特别适合复杂排版、图文混排、需要语义理解的场景。根据是否显式进行布局检测，分为端到端方案和两阶段方案。

#### 2.1 端到端 VLM 方案

直接将 PDF 页面渲染为高分辨率图像，整页喂给 VLM，让模型自行理解布局并提取结构化内容。这是最直接的 VLM 应用方式。

**核心流程**：

```
PDF 页面 → 渲染为图像 → VLM 整页分析 → 结构化输出 (Markdown/JSON)
```

**页面渲染工具**：

| 工具 | 说明 |
|------|------|
| pdf2image | 基于 poppler，Python 中最常用的 PDF→图像工具 |
| PyMuPDF | 内置 page.get_pixmap()，无需额外依赖 |
| ImageMagick | 命令行方案，适合批量处理 |

**常用 VLM 模型**：

| 模型/服务 | 特点 |
|-----------|------|
| GPT-4o / GPT-4V | 理解能力强，支持复杂表格和公式 |
| Claude 3.5 Sonnet / Opus | 长上下文，适合多页文档 |
| Gemini 2.5 Flash / Pro | 高吞吐，成本低，原生多模态 |
| Qwen-VL | 开源方案，可本地部署 |

**示例代码**：

```python
from pdf2image import convert_from_path
import base64, io
from openai import OpenAI

client = OpenAI()

# 渲染为图像
images = convert_from_path("document.pdf", dpi=200)

for img in images:
    # 编码为 PNG base64
    buf = io.BytesIO()
    img.save(buf, format="PNG")
    img_base64 = base64.b64encode(buf.getvalue()).decode()

    # 端到端 VLM 分析
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

**优缺点**：实现简单，一个 API 调用搞定一页；但面对超大页面或多页文档时受限于上下文窗口，且无法精确控制每个区域的提取策略。

#### 2.2 两阶段方案：布局检测 → 区域提取

这是文档解析领域的经典两阶段范式——将「找到内容在哪」和「理解内容是什么」分离处理，各阶段可以用最合适的模型。

| 阶段 | 目标 | 方法 |
|------|------|------|
| **第一阶段：布局检测** | 识别页面上的各类元素区域（标题、正文段落、表格、图片、页眉页脚、公式等），输出每个区域的边界框和类型 | 传统规则（pdfplumber 坐标分析）、深度学习检测模型（layoutparser、DocTR、Surya）、轻量级目标检测（YOLO-based）、MinerU（doclayout-yolo） |
| **第二阶段：区域提取** | 对每个检测到的区域，选择最合适的模型进行内容提取和结构化 | 文本区域 → VLM 或 OCR 提取；表格区域 → 表格专用 VLM 或 Camelot/Tabula；图片区域 → VLM 描述或 OCR；公式区域 → LaTeX 转换模型 |

**核心流程**：

```
PDF 页面 → 渲染为图像 → 布局检测（Stage 1）→ 按区域提取（Stage 2）→ 结构化输出
```


**代表性工具：MinerU**

MinerU 是两阶段方案的开源标杆实现，由 OpenDataLab（上海人工智能实验室）推出：

- **Stage 1**：基于 doclayout-yolo 进行文档布局检测，识别标题、正文、表格、公式、图片等区域
- **Stage 2**：表格区域用表格识别模型还原为 Markdown/HTML 表格；公式区域通过 UnimeRNet 转为 LaTeX；正文区域 OCR 后保留阅读顺序

MinerU 在学术论文、研报、财报等复杂排版场景下表现突出，且完全开源可离线部署。

**两阶段 vs 端到端**：

| 维度 | 端到端 VLM | 两阶段（布局→提取） |
|------|-----------|---------------------|
| 实现复杂度 | 低，一次调用 | 中，需要组合多个模型 |
| 灵活性 | 依赖 VLM 的判断 | 每区域可定制最佳策略 |
| 表格精度 | 依赖模型，可能不稳定 | 表格区域可专用提取器 |
| 大页面处理 | 受上下文窗口限制 | 分区域处理，无限制 |
| 成本控制 | 每页固定 token 开销 | 可跳过低价值区域 |

**选型建议**：
- 快速原型、简单文档 → 端到端 VLM
- 生产环境、高精度要求、复杂表格 → 两阶段方案
- 混合使用：端到端粗筛 + 两阶段精提取

## 方案总体对比

| 维度 | 普通协议解析 | 端到端 VLM | 两阶段（布局→提取） |
|------|-------------|-----------|---------------------|
| 准确性 | 结构化 PDF 高，复杂排版中等 | 整体较高，图文理解好 | 最高，各区域最优处理 |
| 速度 | 极快（毫秒级/页） | 较慢（秒级/页） | 较慢，但可优化 |
| 成本 | 免费（开源库） | API 按 token 计费 | 可混合开源+API |
| 扫描件 | 需额外 OCR 步骤 | 天然支持 | 天然支持 |
| 离线能力 | 完全离线 | API 需联网 | 可全离线（开源检测+VLM） |

**推荐策略**：
- 批量处理结构化 PDF → 普通协议解析（PyMuPDF + Camelot）
- 复杂排版、需要语义理解 → 两阶段方案（布局检测 + VLM）
- 快速验证、原型开发 → 端到端 VLM

## 常用工具速查

| 工具 | 语言 | 类型 | 说明 |
|------|------|------|------|
| PyMuPDF | Python | 协议解析 | 高速文本/图片提取 |
| pdfplumber | Python | 协议解析 | 精确坐标级控制 |
| Camelot | Python | 协议解析 | 表格提取专家 |
| layoutparser | Python | 布局检测 | 深度学习布局分析 |
| Surya | Python | 布局检测+OCR | 新一代开源方案 |
| MinerU | Python | 两阶段方案 | 开源标杆，布局检测+表格/公式/文本提取 |
| PaddleOCR | Python | OCR | 中文场景优秀 |
| GPT-4o / Claude | API | VLM | 端到端文档理解 |
