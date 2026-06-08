<div align="center">
  <h1>FileGlide</h1>
  <p><strong>Agent-oriented filesystem CLI for precise, JSON-first file and path operations.</strong></p>
  <p>Traverse trees, inspect sizes, search names and content, edit exact line ranges, handle binary payloads, and execute explicit batch plans from one consistent command tree.</p>
  <p>
    <a href="https://www.python.org/"><img alt="Python 3.10+" src="https://img.shields.io/badge/Python-3.10%2B-2F5D8C"></a>
    <a href="https://pypi.org/project/click/"><img alt="CLI framework Click" src="https://img.shields.io/badge/CLI-Click-2563EB"></a>
    <a href="https://pypi.org/project/vortezwohl/"><img alt="Fuzzy matching vortezwohl" src="https://img.shields.io/badge/Fuzzy-vortezwohl-0F766E"></a>
    <a href="docs/fileglide-architecture.html"><img alt="Architecture HTML" src="https://img.shields.io/badge/Docs-Architecture-9A3412"></a>
  </p>
  <p>
    <a href="#installation">Installation</a> |
    <a href="#quick-start">Quick Start</a> |
    <a href="#command-tree">Command Tree</a> |
    <a href="#output-and-encoding">Output and Encoding</a> |
    <a href="#search-and-editing">Search and Editing</a> |
    <a href="#batch-execution">Batch Execution</a> |
    <a href="#more-docs">More Docs</a>
  </p>
</div>

<p align="center"><strong>English | <a href="i18n/README.zh-CN.md">简体中文</a> | <a href="i18n/README.zh-TW.md">繁體中文</a></strong></p>

## Installation

This repository currently targets local project execution. Use the project virtual environment command directly:

```powershell
& 'D:\github-project\fileglide\.venv\Scripts\fileglide.exe' --help
```

Python `3.10+` is required. The CLI is implemented with `click` and exposes JSON as the default output contract.

## Quick Start

### 1. Create a scoped workspace path

```powershell
& 'D:\github-project\fileglide\.venv\Scripts\fileglide.exe' path create 'tests/tmp/demo/docs' --root 'D:\github-project\fileglide' --parents --exist-ok
```

Use `--root` to keep all relative paths constrained to a known workspace root.

### 2. Create and write a text file

```powershell
& 'D:\github-project\fileglide\.venv\Scripts\fileglide.exe' file create 'tests/tmp/demo/docs/notes.txt' --root 'D:\github-project\fileglide' --parents --exist-ok
& 'D:\github-project\fileglide\.venv\Scripts\fileglide.exe' text write 'tests/tmp/demo/docs/notes.txt' --root 'D:\github-project\fileglide' --mode overwrite --content "alpha`nbravo`ncharlie"
```

Use `overwrite`, `append`, or `insert` modes to control how text mutations are applied.

### 3. Read exact line ranges

```powershell
& 'D:\github-project\fileglide\.venv\Scripts\fileglide.exe' text read 'tests/tmp/demo/docs/notes.txt' --root 'D:\github-project\fileglide' --start-line 2 --end-line 3
```

This is useful when an agent needs precise context instead of loading an entire file.

### 4. Search file content with regex

```powershell
& 'D:\github-project\fileglide\.venv\Scripts\fileglide.exe' text grep 'bravo|charlie' 'tests/tmp/demo' --root 'D:\github-project\fileglide' --include '*.txt'
```

Content search supports regular expressions and scoped traversal.

### 5. Fuzzy search file names

```powershell
& 'D:\github-project\fileglide\.venv\Scripts\fileglide.exe' file search 'note' 'tests/tmp/demo' --root 'D:\github-project\fileglide' --mode fuzzy
```

Fuzzy filename and pathname matching is powered by `vortezwohl` Levenshtein distance utilities.

## Command Tree

| Command | Role | Typical use |
|---|---|---|
| `file` | File lifecycle and file-only traversal | Create, delete, list, test existence, and search files |
| `path` | Directory lifecycle and path-only traversal | Create, delete, list, test existence, and search directories |
| `tree` | Mixed tree traversal | Return both files and directories from a scoped start path |
| `text` | Text and binary content operations | Read, write, grep, replace line ranges, insert by anchor, and copy binary payloads |
| `inspect` | Size and byte inspection | Measure file or directory size and read binary-safe byte slices |
| `batch` | Explicit batch execution | Validate and execute JSON plans step by step |

## Output and Encoding

Current public-facing defaults:

- `--format json`
- `--pretty`
- scoped path resolution through `--root`
- UTF-8 without BOM works directly
- common encodings such as UTF-8 BOM, UTF-16 LE, GBK, GB18030, and Shift-JIS are detected and handled in the text workflow
- binary commands keep payloads binary-safe instead of forcing text decoding

Practical notes:

- Use `--format text` when a human-readable console summary is preferable.
- Keep `json` for scripts, automation, and agent tool invocation.
- Text commands can force an encoding explicitly with `--encoding`.
- Binary write and byte-slice inspection avoid accidental corruption of non-text data.

## Search and Editing

Representative editing and search operations:

```powershell
& 'D:\github-project\fileglide\.venv\Scripts\fileglide.exe' text write 'tests/tmp/demo/docs/notes.txt' --root 'D:\github-project\fileglide' --mode append --content "`ndelta"
& 'D:\github-project\fileglide\.venv\Scripts\fileglide.exe' text write 'tests/tmp/demo/docs/notes.txt' --root 'D:\github-project\fileglide' --mode insert --position 6 --content '[INSERT]'
& 'D:\github-project\fileglide\.venv\Scripts\fileglide.exe' text replace-lines 'tests/tmp/demo/docs/notes.txt' --root 'D:\github-project\fileglide' --start-line 2 --end-line 2 --content 'BRAVO'
& 'D:\github-project\fileglide\.venv\Scripts\fileglide.exe' text insert-anchor 'tests/tmp/demo/docs/notes.txt' --root 'D:\github-project\fileglide' --after --anchor 'BRAVO' --content "`nANCHOR-INSERT"
```

```powershell
& 'D:\github-project\fileglide\.venv\Scripts\fileglide.exe' path search 'doc' 'tests/tmp/demo' --root 'D:\github-project\fileglide' --mode fuzzy --kind directory
& 'D:\github-project\fileglide\.venv\Scripts\fileglide.exe' tree list 'tests/tmp/demo' --root 'D:\github-project\fileglide' --kind all
& 'D:\github-project\fileglide\.venv\Scripts\fileglide.exe' inspect size 'tests/tmp/demo' --root 'D:\github-project\fileglide'
```

These commands are designed for AI agents that need exact, local, and verifiable filesystem actions instead of ad hoc shell parsing.

## Batch Execution

`fileglide` supports explicit JSON plans for grouped operations.

Example plan shape:

```json
{
  "steps": [
    {
      "action": "path.create",
      "params": {
        "target": "tests/tmp/batch-demo",
        "root": "D:\\github-project\\fileglide",
        "parents": true,
        "exist_ok": true
      }
    },
    {
      "action": "text.write",
      "params": {
        "target": "tests/tmp/batch-demo/readme.txt",
        "root": "D:\\github-project\\fileglide",
        "mode": "overwrite",
        "content": "batch-ready"
      }
    }
  ]
}
```

Preview first, then apply when the plan looks correct:

```powershell
& 'D:\github-project\fileglide\.venv\Scripts\fileglide.exe' batch run 'D:\github-project\fileglide\tests\fixtures\batch\sample-plan.json' --dry-run
& 'D:\github-project\fileglide\.venv\Scripts\fileglide.exe' batch run 'D:\github-project\fileglide\tests\fixtures\batch\sample-plan.json' --apply
```

## More Docs

- [Architecture and project design](docs/fileglide-architecture.html)
- [简体中文 README](i18n/README.zh-CN.md)
- [繁體中文 README](i18n/README.zh-TW.md)

## License

MIT License.