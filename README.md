# File Parsing Handbook

A comprehensive handbook collecting popular GitHub open-source solutions for all types of file parsing, covering PDF, Word, Excel, scanned files, images, TXT, etc., with layout analysis, text & table extraction practices.

## 目录结构

### 核心文档
- [项目概览](docs/overview.md)
- [快速开始](docs/getting-started.md)
- [通用最佳实践](docs/best-practices.md)

### 文件类型解析指南

| 类型 | 说明 | 指南 |
|------|------|------|
| PDF | PDF 文档解析、表格提取、OCR处理 | [指南 →](parsing-guides/pdf/) |
| Word | Word 文档读取、模板处理 | [指南 →](parsing-guides/word/) |
| Excel | Excel 表格数据读取、公式处理 | [指南 →](parsing-guides/excel/) |
| PPT | PowerPoint 幻灯片内容提取 | [指南 →](parsing-guides/ppt/) |
| Images | 图片文字识别 (OCR)、扫描件处理 | [指南 →](parsing-guides/images/) |
| Markdown | Markdown 文档解析、格式转换 | [指南 →](parsing-guides/markdown/) |
| HTML | HTML 网页内容提取、解析 | [指南 →](parsing-guides/html/) |
| XML | XML 文档解析、数据提取 | [指南 →](parsing-guides/xml/) |
| YAML | YAML 配置文件解析 | [指南 →](parsing-guides/yaml/) |
| RTF | 富文本文档解析 | [指南 →](parsing-guides/rtf/) |
| Email | 邮件 (EML/MSG) 内容提取 | [指南 →](parsing-guides/email/) |
| Archive | 压缩包 (ZIP/TAR) 解析 | [指南 →](parsing-guides/archive/) |
| Plain Text | CSV、JSON、TXT 等纯文本处理 | [指南 →](parsing-guides/plain-text/) |

### 其他资源
- [工具评测](tool-reviews/) - 各类解析工具和库的对比评测
- [通用模式](common-patterns/) - 错误处理、内存优化等可复用模式
- [综合示例](examples/) - 发票解析、简历解析等实际场景示例

## 快速开始

```bash
# 克隆仓库
git clone https://github.com/your-username/file-parsing-handbook.git

# 安装依赖
pip install -r requirements.txt

# 查看文档
cd docs
```

详见 [快速开始指南](docs/getting-started.md)。

## 贡献

欢迎贡献！详见 [贡献指南](.github/CONTRIBUTING.md)。

## 许可证

[许可证文件](LICENSE)
