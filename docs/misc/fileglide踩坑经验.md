# fileglide 踩坑经验

## 背景

这次在 `cover_generation` 项目里，需要只通过 `D:\github-project\fileglide\.venv\Scripts\fileglide.exe` 完成代码读取、定点编辑和写回。

实际操作过程中，先后使用了：
- `text read`
- `text insert-anchor`
- `text replace-lines`
- `text write`

其中最主要的踩坑点来自 `insert-anchor` 和后续继续沿用旧行号做 `replace-lines`。

## 结论先说

这次问题的主因不是 `fileglide` 自身已确认的 bug，而是对它的编辑语义理解不够准确，尤其是：
- 把 `insert-anchor --after` 误理解成“插到这一行后面”；
- 实际上它更接近“插到 anchor 字符串后面”；
- 第一次插入后没有立刻重新读取文件，再继续按旧行号做替换；
- 导致新旧片段被拼接，局部代码结构损坏。

一句话总结：`fileglide` 是低层文本编辑工具，不是结构化代码编辑器。

## 这次具体是怎么踩坑的

### 1. `insert-anchor` 是字符串级插入，不是整行级插入

这次在测试文件里新增 import 时，使用了：

```powershell
fileglide text insert-anchor --after --anchor "from src.service.context_resolver import infer_story_mood"
```

原本预期是：
- 在这行 import 的下一行插入新 import。

实际行为是：
- 把新内容直接插到 anchor 字符串末尾；
- 如果插入内容前面没有显式换行，就会直接接在当前行后面；
- 最终把两条 import 拼成一行。

错误结果类似：

```python
from src.service.context_resolver import infer_story_moodfrom src.service.context_resolver import infer_lang
```

这说明：
- `after` 的语义更接近“anchor 最后一个字符之后”；
- 不是“anchor 所在行之后”。

### 2. `insert-anchor` 后继续使用旧行号，会让 `replace-lines` 错位

这次在 `src/service/context_resolver.py` 里先插入 helper，再修改 `infer_lang`。

风险点在于：
- 第一次插入之后，文件行号已经整体变化；
- 如果第二步还按插入前估算的旧行号做 `replace-lines`；
- 替换区间就可能切到错误位置；
- 结果就是新代码覆盖了一半、旧代码残片又留了一半。

最终表象通常是：
- 函数前半段是新的；
- 函数后半段还是旧的；
- 中间还可能混进残缺的 helper 或 docstring。

这类问题不是“只差一两行”的小偏移，而是会直接把代码拼坏。

### 3. `insert-anchor` 不会帮你校验“你插进去的是不是完整代码块”

这次第一次插 helper 时，传进去的内容不完整，缺了函数签名，结果文件里出现了：
- 一个裸 docstring；
- 下面跟着缩进代码体；
- 但没有 `def ...:` 开头。

`fileglide` 仍然执行成功，因为它只负责文本插入，不负责验证你插进去的是不是一个完整函数。

这说明：
- `insert-anchor` 成功，不等于代码结构正确；
- 它只说明文本写入成功。

## 对 `insert-anchor` 的正确理解

建议把 `insert-anchor` 理解成下面这个模型：

1. 先在文件里查找唯一 anchor 字符串。
2. 根据 `--before` 或 `--after` 计算一个文本插入位置。
3. 把给定内容原样插进去。
4. 保持原文件编码和换行策略。
5. 不理解 Python、import、函数、类、缩进层级这些结构语义。

也就是说，它的定位更像：
- 文本补丁工具；
- 字符串锚点插入工具；
- 不是 AST 编辑器；
- 不是“按代码块智能插入”的工具。

## 什么时候适合用 `insert-anchor`

适合场景：
- 在稳定模板里补固定片段；
- 在唯一锚点前后追加几行配置；
- 往文档中插入说明段落；
- 一次性插入不再继续依赖旧行号的内容。

不适合场景：
- 先插一段，再按旧行号继续做多轮替换；
- 修改函数、类、import 块这种结构化代码；
- 需要保证语法块完整的复杂编辑；
- 一次任务里连续做多个相互依赖的定点补丁。

## 这次踩坑后的稳定做法

### 做法 1：插入后立刻重读，不沿用旧行号

推荐顺序：

1. `text read` 读取目标区块；
2. `insert-anchor` 做第一次插入；
3. 再次 `text read` 读取最新文件；
4. 重新计算后续 `replace-lines` 的真实行号；
5. 再执行第二次编辑。

不要这样做：

1. 先估算一个行号；
2. 插入一段；
3. 不重读，直接继续按旧行号替换。

### 做法 2：整块替换优先于碎片拼接

如果目标是修改一个完整函数，优先：
- 直接 `read` 出函数周边；
- 确认完整边界；
- 一次 `replace-lines` 重写整个函数。

不要优先：
- 先插一点；
- 再补一点；
- 再替换一点。

原因很简单：
- 碎片编辑越多，坐标漂移越大；
- 出错后也更难定位。

### 做法 3：对 `--after` 插整行时，默认自己补换行

如果必须用 `insert-anchor --after` 给某一行后面追加整行内容，应该假设工具不会自动帮你换行。

更稳妥的思路是：
- 明确检查 anchor 是否包含行末换行；
- 明确让插入内容自己处理好前后换行；
- 或者直接避免用 `after` 做 import 行插入，改为整段 `replace-lines`。

### 做法 4：编辑成功后必须立刻做只读复核

每次写入后，至少做一次：
- `text read` 回读刚刚修改的区块；
- 检查是否出现拼接、丢行、重复残片；
- 确认编码、换行和结构正常；
- 再决定是否继续下一步。

这一步非常关键，因为：
- `fileglide` 的“写入成功”只代表文件写成功；
- 不代表代码结构正确。

## 对 `replace-lines` 的补充认识

`replace-lines` 本身比 `insert-anchor` 更适合代码修改，因为它的边界更明确。

但它也有前提：
- 你给的起止行必须是最新文件的真实行号；
- 不能拿第一次读取时的旧坐标去替换第二次编辑后的文件。

因此：
- 单次整块替换通常比多次插入更稳；
- 连续编辑时，每一步之后都要重新读取。

## 推荐操作范式

如果以后继续用 `fileglide` 改代码，推荐遵循下面这套顺序：

1. 先 `text read`，确认目标块完整边界。
2. 能整块 `replace-lines` 就不要拆成多次 `insert-anchor`。
3. 如果必须 `insert-anchor`，插入后立刻重新 `read`。
4. 后续所有行号都以最新读取结果为准。
5. 每次写入后都做回读复核。
6. 最后再跑测试，不要把“写入成功”当成“修改完成”。

## 一句话经验总结

`fileglide` 的 `insert-anchor` 本质上是“基于唯一字符串锚点的原样文本插入”。

它的问题不在于“不精确”，而在于“它比直觉里更字面、更底层”。

只要把它误当成“按整行、按代码块、按结构插入”的工具，就很容易把片段拼坏。

## PowerShell 相关踩坑经验

这次除了 `fileglide` 自身的编辑语义，还有几类 PowerShell 配合使用时的坑，实际影响了文档写入、回读和问题定位。

### 1. 多行内容直接传给 `--content` 很容易被 PowerShell 拆参数

看起来像这样：

```powershell
fileglide text write ... --content $content target.md
```

但如果 `$content` 里包含多行、代码块、引号，PowerShell 可能不会把它稳定地当作单个参数传过去，而是把后续内容拆成额外参数。

这次就出现了 `Got unexpected extra arguments`，说明并不是 `fileglide` 不支持写入，而是 shell 参数层先炸了。

经验：
- 短文本可以用 `--content`；
- 长文档不要赌 shell 引号；
- 更稳的是走 `batch run`，把正文放进 JSON 计划里。

### 2. `stdin` 和 `--content-file` 在 PowerShell 下不一定稳定是纯文本

这次试过：
- `--content-stdin`
- `--content-file`

两条路径都被 `fileglide` 判成了 `invalid_text_content_source` 或 `binary_detected`。

原因不是文档本身一定是二进制，而是 PowerShell 到 `fileglide` 之间这一层，可能混入了 BOM、编码不一致或管道字节流和预期文本流不一致的问题。

经验：
- 不要默认 PowerShell 管道里的文本就是 `fileglide` 能稳定识别的 UTF-8 文本；
- 如果 `stdin` / `content-file` 反复异常，优先切换到 `batch run`；
- `batch run` 把内容放进 JSON 字段里，通常比 shell 直接传长文本更稳。

### 3. `Set-Content -Encoding utf8` 不一定等价于 UTF-8 without BOM

这次中转文件一度被 `fileglide` 当成 binary，排查时必须显式把临时文件改成 UTF-8 without BOM。

经验：
- 在 PowerShell 里不要想当然地认为 `utf8` 就一定没有 BOM；
- 如果后续工具对文本编码严格，最好显式使用 `.NET` 的 `UTF8Encoding($false)`；
- 项目要求是 UTF-8 without BOM，就不要依赖 PowerShell 默认行为。

### 4. `Get-Content` 的终端显示乱码，不一定等于文件内容真的坏了

这次文档写入后，PowerShell 直接 `Get-Content` 输出看起来像乱码，但 `fileglide inspect.bytes` 又显示文件头部字节是合理的 UTF-8。

这说明显示层和文件层是两回事：
- 文件字节可能是对的；
- 终端输出编码可能是错的；
- 不能只看终端显示就下结论。

经验：
- 先看字节，再看显示；
- 先确认文件实际编码，再判断是不是乱码；
- `Get-Content` 的显示结果只能作为线索，不能直接当最终事实。

### 5. 排查顺序要先分层，不要把 PowerShell 问题误判成 `fileglide` 问题

这次实际遇到的问题分成三层：
- `fileglide` 的编辑语义问题；
- PowerShell 的参数展开和编码问题；
- 终端显示编码问题。

如果不分层，很容易把：
- shell 拆参；
- BOM 问题；
- 终端乱码；

统统误认为是 `fileglide` 的 bug。

更稳的排查顺序是：
1. 先看 `fileglide` 返回的是参数错误、文本源错误，还是实际写入成功。
2. 再看目标文件的字节是否合理。
3. 最后再看 PowerShell 终端显示是不是只是展示层乱码。

## 补充结论

如果在 Windows + PowerShell 环境下需要稳定地让 `fileglide` 写长文本，当前最稳的方式是：
- 读取用 `fileglide inspect` / `text.read`；
- 写入优先用 `fileglide batch run`；
- 不要优先依赖长文本 `--content`、临时 `stdin` 管道或默认编码的中转文件。