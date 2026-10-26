# docxtpl-lint

[English](README.md)

**[docxtpl](https://github.com/elapouya/python-docx-template)（python-docx-template）Word 模板的静态检查工具。它能发现写坏的标签、被静默丢掉的内容和数据不匹配，把每个问题定位到具体段落和表格单元格，并返回适合 CI 使用的退出码。**

**[▶ 在浏览器里试用](https://yq6666-66.github.io/docxtpl-lint/)**：拖入一个 `.docx` 就能看到报告。文件不会离开你的电脑，检查通过 Pyodide 在本地运行。

![docxtpl-lint 输出](https://raw.githubusercontent.com/yq6666-66/docxtpl-lint/main/docs/demo-zh.svg)

> 与 python-docx-template 官方无关。基于 docxtpl 0.20.2 测试，CI 每周还会对照 docxtpl 最新发布版和 master 检查。

## 为什么需要它

模板写坏时，docxtpl 只抛出一条没有位置的错误：

```text
jinja2.exceptions.TemplateSyntaxError: Encountered unknown tag 'endfor'. Jinja was looking for the following tags: 'elif' or 'else' or 'endif'.
```

更糟的是，很多错误根本不报错，只是生成的文档不对。docxtpl-lint 一次报告所有问题，每个都带位置（正文/页眉/页脚/脚注、段落，以及 `T1 R2 C2 > T2 R2 C1` 这样的表格路径）：

1. **结构错误：** 未闭合或多余的 `if`/`for`，标签没有闭合，Jinja 语法错误，标签跨段落。
2. **docxtpl 特有的坑：** `{%tr %}`、`{%tc %}`、`{%p %}` 用法错误。例如前缀不匹配会静默丢失表格行（[#603](https://github.com/elapouya/python-docx-template/issues/603)），`{%p %}` 里出现 `%` 会出错（[#644](https://github.com/elapouya/python-docx-template/issues/644)），`{%p %}` 旁边的文字会静默消失。
3. **Jinja 错误：** 未知的过滤器和测试，循环变量在循环外使用。
4. **Word 遗留问题：** 未接受的修订删除里残留模板代码（会泄漏到每份生成的文档里）。
5. **数据不匹配**（对照示例 JSON 或 JSON Schema）：缺少变量，对非列表做循环，给 `{{r}}` 传普通字符串，以及关闭 autoescape 时被 docxtpl 静默截断的 `&`、`<`。

## 安装

```sh
pip install docxtpl-lint
pip install "docxtpl-lint[render]"   # 加装 docxtpl，verify 和 check_context 需要
```

需要 Python ≥ 3.10。已发布到 [PyPI](https://pypi.org/project/docxtpl-lint/)。

## 快速开始

```sh
docxtpl-lint templates/*.docx                      # 检查
docxtpl-lint invoice.docx --data sample.json       # 同时对照示例数据
docxtpl-lint verify invoice.docx                   # 用真实 docxtpl 渲染，找出丢失的输出
docxtpl-lint --explain DTL209 --lang zh            # 查看规则说明
```

```console
$ docxtpl-lint verify invoice.docx
invoice.docx:body ¶1:7: error DTL401 `{{ total }}` was rendered by docxtpl but its value does not appear in the output document
    {{ total }}
      > ¶1             | Total {{ total }} {%p if x %}
        ¶2             | {%p endif %}
```

| 退出码 | 含义 |
|---|---|
| 0 | 没有达到 `--fail-on` 级别的问题（默认 `error`） |
| 1 | 有达到该级别的问题 |
| 2 | 参数或配置错误、文件不存在、`.docx`/JSON 无法读取 |

## 功能

### 检查规则

共 30 条规则，分五组。每条规则的示例、对应的 docxtpl 行为和修复方法见 **[docs/rules.md](docs/rules.md)**；`docxtpl-lint --list-rules` 可列出全部规则。

| 分组 | 规则 | 示例 |
|---|---|---|
| 标签与 Word | DTL001–DTL005 | 标签未闭合、标签跨段落、修订删除里有代码、无效的抑制注释 |
| Jinja | DTL101–DTL106 | 块未闭合、多余的 `endfor`、语法错误、未知过滤器/测试、循环变量在循环外使用 |
| docxtpl 标签 | DTL201–DTL210 | `{% tr` 多了空格、`tr` 在表格外、前缀不匹配、`{%tr %}` 跨表格、`{%p %}` 旁的内容被丢弃 |
| 数据（`--data`/`--schema`） | DTL301–DTL308 | 缺少变量、对非列表循环、未在 schema 中声明、对字符串用列表过滤器、`{{r}}` 收到普通值、未转义的 `&`/`<` |
| 渲染（`verify`） | DTL401–DTL402 | 值在输出中被静默丢失、渲染失败 |

用 `--select DTL1,DTL206` 和 `--ignore DTL2` 选择规则（支持前缀）。`--context N` 显示上下文段落，`--list-vars` 列出模板用到的数据路径，`--lang zh` 输出中文提示。

### 数据契约：`infer`、schema、Pydantic

```sh
docxtpl-lint infer a.docx b.docx -o schema.json --sample sample.json --pydantic models.py
docxtpl-lint contract.docx --schema schema.json    # DTL304/DTL305
```

`infer` 生成模板所需数据的 JSON Schema（2020-12）：

1. `for` 循环推断为列表，并包含嵌套字段；
2. 算术、`round` 和比较推断为数字；
3. `is defined`/`default` 推断为可选字段；
4. `{{r}}`/`{{p}}` 推断为 `RichText`/`Subdoc` 对象。

传入多个模板时会合并它们的 schema，并标注每个字段被哪些模板使用。`--sample` 同时生成可渲染的示例数据（`--items N` 设列表条数），`--pydantic` 生成 Pydantic v2 模型。

### `verify`：渲染等价检查

`docxtpl-lint verify a.docx` 用真实的 docxtpl 渲染模板，每个变量填一个唯一的哨兵值，然后报告：

1. 没有出现在输出里的 `{{ value }}`（DTL401）；
2. 渲染失败（DTL402）。

用 `--data`（可重复）可以改用你自己的数据渲染。`--jinja-env MODULE:ATTR` 和 `--autoescape` 让它按你的应用方式渲染，`--out DIR` 保留渲染结果。

`verify` 读取与 lint 相同的配置，支持相同的输出、基线和 `--fail-on` 选项。唯一的例外是配置里的 `data` 不生效，这样哨兵检查始终会运行。

### `diff`：模板版本间的契约变化

```sh
docxtpl-lint diff old.docx new.docx --fail-on breaking   # 也可用 change、never
```

以下变化算破坏性：

1. 新增必填字段；
2. 列表与对象互换；
3. 可选变必填；
4. 普通值变 `RichText`/`Subdoc`。

删除字段不算破坏性，疑似改名会给出提示。

### `fix`：修复建议与安全自动修复

```sh
docxtpl-lint fix contract.docx [--data sample.json]          # 输出段落文本的统一 diff
docxtpl-lint fix contract.docx --write contract.fixed.docx    # 把安全修复应用到副本
```

安全修复包括补全缺少的 `}`、`{% tr` → `{%tr`、前缀配对、`endfro` → `endfor`。修复写入**副本**，并在原有 Word run 内修改，因此保留格式。需要猜测意图的修复（例如根据 `--data` 把 `custmer` 改成 `customer`）要加 `--unsafe`。原模板不会被修改。

### 基线与抑制

```sh
docxtpl-lint templates/*.docx --write-baseline bl.json   # 接受现有问题
docxtpl-lint templates/*.docx --baseline bl.json         # 只有新问题才失败
```

基线条目按文档部件、表格单元格和段落文本定位，插入段落后依然有效。

也可以在模板里用 Jinja 注释屏蔽规则（docxtpl 渲染时不输出任何内容）：

1. `{# docxtpl-lint: disable=DTL301 #}` 作用于整个模板；
2. `disable-line=` 作用于当前段落；
3. `{#p docxtpl-lint: disable-next=… #}` 作用于下一段。

没有屏蔽到任何问题的注释会报 DTL005。

### 输出格式

| 格式 | 用途 |
|---|---|
| `text` | 终端，带上下文和颜色 |
| `json` | 机器可读（`schema_version: 1`），每个问题带稳定的 `fingerprint` |
| `github` | GitHub Actions 注解 |
| `sarif` | GitHub 代码扫描、reviewdog（带 `partialFingerprints`） |
| `gitlab` | GitLab Code Quality |
| `markdown` | job summary / PR 评论 |

用 `--format F`、`-o FILE` 和 `--report FORMAT:FILE`（可重复）可以一次运行生成多份报告。另外两种输出：

1. `--statistics` 按规则统计数量；
2. `--annotate DIR` 生成每个问题带一条 Word 批注的模板副本，方便在 Word 里改模板的人使用。

### CI：GitHub Action、pre-commit、GitLab

**GitHub Action**（在运行记录和 PR 中显示注解，并输出 Markdown job summary）：

```yaml
- uses: actions/checkout@v7
- uses: yq6666-66/docxtpl-lint@v2.0.1
  with:
    files: |
      templates/**/*.docx
    args: --data tests/sample_context.json
    # baseline: docxtpl-lint-baseline.json
    # sarif: docxtpl-lint.sarif     # 再用 github/codeql-action/upload-sarif 上传
    # annotate: true                # 把带批注的副本上传为 artifact
    # summary: false
```

**pre-commit：**

```yaml
repos:
  - repo: https://github.com/yq6666-66/docxtpl-lint
    rev: v2.0.1
    hooks:
      - id: docxtpl-lint
        # args: [--data, tests/sample_context.json]
```

**GitLab CI**（Code Quality 报告）：

```yaml
docxtpl-lint:
  image: python:3.12
  script:
    - pip install docxtpl-lint
    - docxtpl-lint --format gitlab -o gl-code-quality.json templates/*.docx
  artifacts:
    when: always
    reports:
      codequality: gl-code-quality.json
```

**其他 CI：** `docxtpl-lint --format github templates/*.docx`。配合 reviewdog：`docxtpl-lint --format sarif templates/*.docx | reviewdog -f=sarif -name=docxtpl-lint -reporter=github-pr-review`。

### Python API

```python
from docxtpl_lint import check_context, infer_schema, lint

lint("invoice.docx").raise_for_errors()  # 静态检查，例如放在测试里
res = check_context(tpl_bytes, ctx, jinja_env=env)  # 应用渲染前的守卫
if not res.ok:
    log.error(res.to_dict())
schema = infer_schema(["invoice.docx"])
```

模板可以传路径、bytes 或二进制文件对象。传入的 `jinja_env` 会提供自定义过滤器、测试名和 autoescape 设置，且不会被修改。API 遵循语义化版本，详见 **[docs/api.md](docs/api.md)**。

### 在线试用

**[yq6666-66.github.io/docxtpl-lint](https://yq6666-66.github.io/docxtpl-lint/)** 在浏览器里（Pyodide）运行同一个检查器，可以看到问题列表、修复建议、推断出的 schema 和示例数据，文件不会上传。

## 配置

默认值写在最近的 `pyproject.toml` 里，命令行参数优先。未知的键会报错（退出码 2），这样拼写错误不会悄悄关掉检查。

```toml
[tool.docxtpl-lint]
ignore = ["DTL002"]
data = ["tests/sample_context.json"]  # 相对于该 pyproject.toml（verify 不使用）
fail-on = "warning"                   # error | warning | never
lang = "zh"                           # 或 "en"
jinja-env = "myapp.reports:jinja_env" # 从你的环境读取自定义过滤器/测试/autoescape（会导入你的代码）
# 其他键：select、allow-cross-paragraph、report-unused-data、context、baseline、filters、tests、autoescape
```

用 `--config path/to/pyproject.toml` 指定配置文件，`--no-config` 忽略配置。如果用了自定义 Jinja 环境又不想导入它，可以用 `--filters money --tests vat` 列出名字；渲染时开了 `autoescape=True` 就加 `--autoescape`。

## 工作原理

1. **直接读取 `.docx`。** 用 lxml 读取 OOXML 各部件（正文、页眉、页脚、脚注），包括表格结构、域代码、文本框和修订。检查时不导入 docxtpl，也不修改文件。
2. **复现 docxtpl 的预处理。** 使用与 docxtpl 相同的标签清理正则，以及相同的 `{%p/tr/tc/r %}` 处理，所以分析的文本就是 docxtpl 的 Jinja 实际看到的内容。每条 docxtpl 规则都有测试，用真实 docxtpl 渲染错误模板，证明它确实会出问题。
3. **自有标签栈加 Jinja AST。** 块结构问题由自己的标签栈检查，因此能一次报出全部问题。Jinja 的语法和语义（过滤器、测试、作用域、变量路径、类型）来自 Jinja2 解析后的 AST 分析，同一份分析也生成 `infer`、`--data` 和 `--schema` 用的数据契约。
4. **关键处真实渲染。** `verify` 和 `check_context` 用 docxtpl 本身渲染，数据为哨兵值或你提供的数据。
5. **在真实模板上验证。** CI 检查 docxtpl 自带测试集中的 40 个模板（0 误报，故意写坏的那个能被发现），以及 86 个经真实渲染裁决的对抗用例（[基准](benchmark/expected.json)、[对抗用例](benchmark/adversarial_adjudicated.json)）。
6. **能在浏览器运行。** 纯 Python wheel 可以在 Pyodide 中加载，用于在线试用页面。

## 局限

1. **静态检查依赖你提供的数据。** 没有 `--data`/`--schema` 时，类型错误只能靠 `verify` 或渲染时发现。不检查自定义 Jinja 扩展、`include` 和宏签名。
2. **跳过 docxtpl 不渲染的部件：** 尾注、批注、图表、嵌入对象。文本框作为所在段落的一部分检查。
3. **`¶N` 按部件里所有 `<w:p>` 计数**，包括表格单元格里的空段落。请结合表格路径和打印出的片段在 Word 中定位。
4. **DTL201 遵循 docxtpl ≤ 0.20.2 的正则。** 如果上游放宽（PR [#645](https://github.com/elapouya/python-docx-template/pull/645)），请用 `--ignore DTL201`。

## 相关项目

splittist 的 [lintxtpl](https://github.com/splittist/lintxtpl) 是第一个 docxtpl 模板检查工具，"生成带批注的 `.docx`"的想法也来自它。版本历史见 [CHANGELOG.md](CHANGELOG.md)，参与贡献见 [CONTRIBUTING.md](CONTRIBUTING.md)。

## 许可证

MIT
