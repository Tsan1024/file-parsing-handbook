# 通用最佳实践

## 1. 错误处理

始终使用 try-catch 处理文件操作：

```python
def parse_file(file_path):
    try:
        with open(file_path, 'rb') as f:
            # 解析逻辑
            pass
    except FileNotFoundError:
        logger.error(f"文件不存在: {file_path}")
    except PermissionError:
        logger.error(f"无权限访问: {file_path}")
    except Exception as e:
        logger.error(f"解析失败: {e}")
```

## 2. 内存优化

处理大文件时，使用流式处理：

```python
# 不好：一次性读取整个文件
with open('large_file.txt', 'r') as f:
    content = f.read()  # 可能导致内存溢出

# 好：逐行读取
with open('large_file.txt', 'r') as f:
    for line in f:
        process_line(line)
```

## 3. 编码处理

明确指定文件编码：

```python
with open('file.txt', 'r', encoding='utf-8') as f:
    content = f.read()
```

## 4. 验证输入

处理前验证文件类型和大小：

```python
import os

def validate_file(file_path, max_size=10*1024*1024):
    if not os.path.exists(file_path):
        raise FileNotFoundError

    file_size = os.path.getsize(file_path)
    if file_size > max_size:
        raise ValueError(f"文件过大: {file_size} bytes")

    return True
```

## 5. 日志记录

记录关键操作和错误：

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

logger.info(f"开始解析文件: {file_path}")
logger.debug(f"文件大小: {file_size} bytes")
```

## 6. 并行处理

批量处理时使用多进程：

```python
from concurrent.futures import ProcessPoolExecutor

def process_files(file_paths):
    with ProcessPoolExecutor(max_workers=4) as executor:
        executor.map(parse_file, file_paths)
```

更多模式详见：[通用模式](../common-patterns/)
