# PDF 解析指南

## 概述

PDF（Portable Document Format）是一种广泛使用的文档格式，本指南涵盖 PDF 文档解析的最佳实践。

## 内容

- [基础文本提取](basic-extraction.md)
- [表格提取](table-extraction.md)
- [布局分析](layout-analysis.md)
- [扫描 PDF/OCR 处理](ocr-handling.md)

## 常用工具

| 工具 | 语言 | 说明 |
|------|------|------|
| PyPDF2 | Python | 基础 PDF 读取和文本提取 |
| pdfplumber | Python | 表格提取、布局分析 |
| Camelot | Python | 高级表格提取 |
| pdf.js | JavaScript | 浏览器端 PDF 解析 |

## 示例代码

查看 [examples/](examples/) 目录获取可运行示例。
