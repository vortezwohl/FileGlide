# fileglide 命令替代映射

## 总原则

- 任务本质上是文件、目录、文本区间、字节、树遍历或批处理文件计划时，`fileglide` 应该是第一工具。
- 如果你已经想到 `apply_patch`、`cat > file`、`Set-Content`、`Path.write_text()`或 `fs.writeFile()`，说明工具选型偏了。

## 常见映射

| 旧工具 / 习惯 | 用 `fileglide` 替代 | 说明 |
|---|---|---|
| `apply_patch` 整文件重写 | `file create` + `text write --mode overwrite` | 适合新文件或整文件覆盖 |
| `apply_patch` 局部替换 | `text read` + `text replace-lines` | 先读边界，再替换 |
| `cat <<EOF > file`、`tee`、`echo >`、`printf >` | `file create` + `text write` | 避免 shell 引号风险 |
| `sed -i`、`perl -pi` | `text replace-lines` | 默认优先整块替换 |
| `touch`、`New-Item -ItemType File` | `file create` | 可搭配 `--parents` `--exist-ok` |
| `mkdir`、`mkdir -p`、`New-Item -ItemType Directory` | `path create` | 目录创建 |
| `rm file`、`del`、`Remove-Item file` | `file delete` | 高风险操作先 `--dry-run` |
| `rm -r dir`、`Remove-Item -Recurse` | `path delete --recursive` | 高风险操作先 `--dry-run` |
| `mv` 文件或 `Move-Item` 文件 | `file move` | 跨 root 时用 `--to-root` |
| `mv` 目录或 `Move-Item` 目录 | `path move` | 目录移动明确化 |
| `cat`、`type`、`Get-Content` | `text read` | 可按行号读取 |
| `grep`、`rg`、`Select-String` | `text grep` | 内容检索 |
| `find`、`fd`、`Get-ChildItem` 找文件 | `file search` 或 `file list` | 名称搜索或遍历 |
| `find`、`Get-ChildItem` 找目录 | `path search` 或 `path list` | 目录遍历 |
| `tree`、`ls -R`、`dir /s` | `tree list` | 混合树遍历 |
| `stat`、`du` | `inspect size` | 看文件或目录大小 |
| 字节或十六进制检查 | `inspect bytes` | BOM 和编码排障 |
| 二进制写入 | `text binary-write` | 搭配 `--data-stdin` 或 `--data-file` |
| 一组文件系统计划 | `batch run` | 先 `--dry-run` 再执行 |

## 大内容写入规则

- 不要优先用长 `--content`、shell heredoc、shell 管道、`Set-Content`、`Path.write_text()`、`fs.writeFile()`。
- 必须优先用 Python `subprocess.run(..., input=payload.encode("utf-8"))` 配合 `fileglide text write --content-stdin`。
- 如果是二进制，改用 `fileglide text binary-write --data-stdin`。

## batch 结构提醒

真实的 batch 计划结构是：根对象包含 `steps`，每个 step 包含 `action` 和 `arguments`。
