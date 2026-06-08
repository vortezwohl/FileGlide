# fileglide 稳定工作流与踩坑方法论

## 核心心智模型

`fileglide` 是低层、可验证、面向文件系统的精确工具，不是 AST 编辑器，也不会替你理解代码结构。

## 稳定工作流

1. 先 `text read` 读准边界，再决定怎么改。
2. 能整块替换就整块替换，默认优先 `text replace-lines`。
3. 大内容写入时，用 Python 发 UTF-8 字节到 `fileglide text write --content-stdin`。
4. 每次写入后立即回读。
5. 怀疑编码或 BOM 问题时，用 `inspect bytes` 查真实字节。

## `insert-anchor` 的正确理解

- 它是“在唯一 anchor 字符串前或后做字符级插入”。
- `--after` 意味着 anchor 字符串之后，不是“这一行之后”。
- 如果你需要新行，就要自己显式补换行。
- anchor 插入后必须立即重新读文件，不能沿用旧行号。
- 修改完整函数、类或 import 块时，通常应该用 `replace-lines`，不是 `insert-anchor`。

## Python stdin 写入模式

```python
import json
import subprocess

fileglide = r"D:\github-project\fileglide\.venv\Scripts\fileglide.exe"
root = r"D:\github-project\fileglide"
target = "docs/tmp/large.md"
payload = "# 标题

这里是大块正文。
"

command = [
    fileglide,
    "text",
    "write",
    "--root",
    root,
    "--mode",
    "overwrite",
    "--content-stdin",
    target,
]
completed = subprocess.run(
    command,
    input=payload.encode("utf-8"),
    capture_output=True,
    check=False,
)
response = json.loads(completed.stdout.decode("utf-8"))
if completed.returncode != 0:
    raise RuntimeError(response)
```

## 编码和 shell 层排障

- 多行正文、引号和特殊字符很容易在 shell 参数层先出问题，所以长正文默认不要走 `--content`。
- `invalid_text_content_source` 或 `binary_detected` 不一定是 `fileglide` bug，可能是 shell、BOM、编码或终端显示层的问题。
- 终端显示乱码不等于文件已损坏，先看 JSON 返回，再看 `inspect bytes`。

## 方法论总结

- 先读清边界，再改
- 优先整块替换，不要碎片拼接
- 大内容统一走 Python UTF-8 stdin
- 每次写完都立即回读
- 把文件写入成功、结构正确、语义正确分层验证
