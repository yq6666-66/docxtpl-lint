# docxtpl-lint

[中文文档 / Chinese](README.zh.md)

**A static linter for [docxtpl](https://github.com/elapouya/python-docx-template) (python-docx-template) Word templates. It finds broken tags, silently lost content and data mismatches, points each one to the exact paragraph and table cell, and gives CI-friendly exit codes.**

**[▶ Try it in your browser](https://yq6666-66.github.io/docxtpl-lint/)**: drop in a `.docx` to see the report. The file stays on your computer; the linter runs locally via Pyodide.

![docxtpl-lint output](https://raw.githubusercontent.com/yq6666-66/docxtpl-lint/main/docs/demo.svg)

> Not affiliated with python-docx-template. Tested against docxtpl 0.20.2; CI also checks weekly against docxtpl's latest release and master.

## Why

When a template is broken, docxtpl raises one error with no location:

```text
jinja2.exceptions.TemplateSyntaxError: Encountered unknown tag 'endfor'. Jinja was looking for the following tags: 'elif' or 'else' or 'endif'.
```

Worse, many mistakes raise nothing at all, and the generated document is just wrong. docxtpl-lint reports all of them in one run, each with its location (body / header / footer / footnotes, paragraph, and table path such as `T1 R2 C2 > T2 R2 C1`):

1. **Broken structure:** unclosed or stray `if`/`for`, tags that never close, Jinja syntax errors, tags split across paragraphs.
2. **docxtpl-specific traps:** `{%tr %}` / `{%tc %}` / `{%p %}` misuse, such as prefix mismatches that silently drop table rows ([#603](https://github.com/elapouya/python-docx-template/issues/603)), `%` inside `{%p %}` ([#644](https://github.com/elapouya/python-docx-template/issues/644)), or text next to a `{%p %}` tag that silently disappears.
3. **Jinja mistakes:** unknown filters and tests, and loop variables used after the loop.
4. **Word artefacts:** template code inside unaccepted tracked deletions (it leaks into every generated document).
5. **Data mismatches** against sample JSON or a JSON Schema: missing variables, loops over non-lists, plain strings given to `{{r}}`, and `&` / `<` values that docxtpl silently cuts when autoescape is off.

## Install

```sh
pip install docxtpl-lint
pip install "docxtpl-lint[render]"   # adds docxtpl, needed for `verify` and `check_context`
```

Requires Python ≥ 3.10. Available on [PyPI](https://pypi.org/project/docxtpl-lint/).

## Quick start

```sh
docxtpl-lint templates/*.docx                      # lint
docxtpl-lint invoice.docx --data sample.json       # also check against sample data
docxtpl-lint verify invoice.docx                   # render with real docxtpl and find lost output
docxtpl-lint --explain DTL209                      # what a rule means
```

```console
$ docxtpl-lint verify invoice.docx
invoice.docx:body ¶1:7: error DTL401 `{{ total }}` was rendered by docxtpl but its value does not appear in the output document
    {{ total }}
      > ¶1             | Total {{ total }} {%p if x %}
        ¶2             | {%p endif %}
```

| exit code | meaning |
|---|---|
| 0 | no findings at the `--fail-on` level (default `error`) |
| 1 | findings at that level |
| 2 | bad arguments or config, missing file, unreadable `.docx` / JSON |

## Features

### Lint rules

There are 30 rules in five groups. Each has examples, the docxtpl behaviour it guards against, and a fix in **[docs/rules.md](docs/rules.md)**. Run `docxtpl-lint --list-rules` to list them all.

| group | rules | examples |
|---|---|---|
| tags and Word | DTL001–DTL005 | tag never closed, tag spans paragraphs, code in a tracked deletion, unused suppression |
| Jinja | DTL101–DTL106 | unclosed block, stray `endfor`, syntax error, unknown filter/test, loop variable outside its loop |
| docxtpl tags | DTL201–DTL210 | `{% tr` with a space, `tr` outside a table, prefix mismatch, `{%tr %}` across tables, content dropped next to `{%p %}` |
| data (`--data` / `--schema`) | DTL301–DTL308 | missing variable, loop over a non-list, undeclared in schema, list filter on a string, plain value for `{{r}}`, unescaped `&` / `<` |
| render (`verify`) | DTL401–DTL402 | value silently lost in the output, render fails |

Use `--select DTL1,DTL206` and `--ignore DTL2` to choose rules (prefixes allowed). Use `--context N` to print the surrounding paragraphs, `--list-vars` to list the data paths a template uses, and `--lang zh` for messages in Chinese.

### Data contract: `infer`, schema, Pydantic

```sh
docxtpl-lint infer a.docx b.docx -o schema.json --sample sample.json --pydantic models.py
docxtpl-lint contract.docx --schema schema.json    # DTL304/DTL305
```

`infer` writes the JSON Schema (2020-12) of the data the templates need:

1. lists from `for` loops and nested fields;
2. numbers from arithmetic, `round` and comparisons;
3. optional fields from `is defined` / `default`;
4. `RichText` / `Subdoc` objects from `{{r}}` / `{{p}}`.

When several templates are given, their schemas are merged, and each field is tagged with the templates that use it. `--sample` adds sample data that renders (`--items N` list items), and `--pydantic` writes Pydantic v2 models.

### `verify`: render-equivalence check

`docxtpl-lint verify a.docx` renders the template with the real docxtpl and a unique sentinel value per variable. It reports:

1. every `{{ value }}` that does not reach the output (DTL401);
2. a render that fails (DTL402).

Pass `--data` (repeatable) to render your own contexts instead. Pass `--jinja-env MODULE:ATTR` and `--autoescape` to render the way your app does, and `--out DIR` to keep the rendered documents.

`verify` reads the same configuration and supports the same output, baseline and `--fail-on` options as lint. The one exception is config `data`: it is not applied, so the sentinel check always runs.

### `diff`: contract changes between template versions

```sh
docxtpl-lint diff old.docx new.docx --fail-on breaking   # or: change, never
```

These count as breaking:

1. new required fields;
2. list ↔ object changes;
3. optional → required;
4. plain → `RichText` / `Subdoc`.

Removed fields are not breaking, and likely renames are hinted.

### `fix`: suggestions and safe auto-fix

```sh
docxtpl-lint fix contract.docx [--data sample.json]          # unified diffs of the paragraph text
docxtpl-lint fix contract.docx --write contract.fixed.docx    # apply safe fixes to a copy
```

Safe fixes include a missing `}`, `{% tr` → `{%tr`, prefix pairs, and `endfro` → `endfor`. They are applied to a **copy**, inside the existing Word runs, so formatting is kept. Fixes that guess at intent (for example `custmer` → `customer` from `--data`) need `--unsafe`. The original template is never modified.

### Baseline and suppressions

```sh
docxtpl-lint templates/*.docx --write-baseline bl.json   # accept today's findings
docxtpl-lint templates/*.docx --baseline bl.json         # fail only on new ones
```

Baseline entries are keyed by part, table cell and paragraph text, so they survive inserted paragraphs.

You can also silence rules inside the template with a Jinja comment (docxtpl renders it as nothing):

1. `{# docxtpl-lint: disable=DTL301 #}` for the whole template;
2. `disable-line=` for the current paragraph;
3. `{#p docxtpl-lint: disable-next=… #}` for the next paragraph.

A suppression that matches nothing is reported as DTL005.

### Output formats

| format | use |
|---|---|
| `text` | terminal, with context lines and colour |
| `json` | machine readable (`schema_version: 1`), with a stable `fingerprint` per finding |
| `github` | GitHub Actions annotations |
| `sarif` | GitHub code scanning, reviewdog (with `partialFingerprints`) |
| `gitlab` | GitLab Code Quality |
| `markdown` | job summary / PR comment |

Use `--format F`, `-o FILE`, and `--report FORMAT:FILE` (repeatable) to write several reports in one run. Two more outputs:

1. `--statistics` prints counts per rule;
2. `--annotate DIR` writes copies of the templates with one Word comment per finding, for people who fix templates in Word.

### CI: GitHub Action, pre-commit, GitLab

**GitHub Action** (annotations on the run and the PR, plus a Markdown job summary):

```yaml
- uses: actions/checkout@v7
- uses: yq6666-66/docxtpl-lint@v2.0.1
  with:
    files: |
      templates/**/*.docx
    args: --data tests/sample_context.json
    # baseline: docxtpl-lint-baseline.json
    # sarif: docxtpl-lint.sarif     # then upload with github/codeql-action/upload-sarif
    # annotate: true                # upload commented copies as an artifact
    # summary: false
```

**pre-commit:**

```yaml
repos:
  - repo: https://github.com/yq6666-66/docxtpl-lint
    rev: v2.0.1
    hooks:
      - id: docxtpl-lint
        # args: [--data, tests/sample_context.json]
```

**GitLab CI** (Code Quality report):

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

**Other CI:** `docxtpl-lint --format github templates/*.docx`. With reviewdog: `docxtpl-lint --format sarif templates/*.docx | reviewdog -f=sarif -name=docxtpl-lint -reporter=github-pr-review`.

### Python API

```python
from docxtpl_lint import check_context, infer_schema, lint

lint("invoice.docx").raise_for_errors()  # static check, e.g. in tests
res = check_context(tpl_bytes, ctx, jinja_env=env)  # guard right before rendering in your app
if not res.ok:
    log.error(res.to_dict())
schema = infer_schema(["invoice.docx"])
```

Templates can be paths, bytes or binary file objects. A passed `jinja_env` supplies the custom filter and test names and the autoescape setting, and is never modified. The API follows semantic versioning. See **[docs/api.md](docs/api.md)**.

### Playground

**[yq6666-66.github.io/docxtpl-lint](https://yq6666-66.github.io/docxtpl-lint/)** runs the same linter in the browser (Pyodide). It shows findings, fix suggestions and the inferred schema and sample data. Nothing is uploaded.

## Configuration

Defaults go in the nearest `pyproject.toml`. Command-line options take precedence. Unknown keys are an error (exit 2), so a typo never silently disables a check.

```toml
[tool.docxtpl-lint]
ignore = ["DTL002"]
data = ["tests/sample_context.json"]  # relative to this pyproject.toml (not used by verify)
fail-on = "warning"                   # error | warning | never
lang = "en"                           # or "zh"
jinja-env = "myapp.reports:jinja_env" # take custom filters/tests/autoescape from your environment (imports your code)
# also: select, allow-cross-paragraph, report-unused-data, context, baseline, filters, tests, autoescape
```

Use `--config path/to/pyproject.toml` to pick a file, or `--no-config` to ignore it. If you render with a custom Jinja environment but don't want it imported, list its names with `--filters money --tests vat`. If you render with `autoescape=True`, add `--autoescape`.

## How it works

1. **Reads the `.docx` directly.** It reads the OOXML parts (body, headers, footers, footnotes) with lxml, including table structure, field codes, text boxes and tracked changes. Linting does not import docxtpl and never modifies the file.
2. **Mirrors docxtpl's preprocessing.** It applies the same tag-cleaning regexes as docxtpl and the same handling of `{%p/tr/tc/r %}`, so the template text it analyses is what docxtpl's Jinja actually sees. Each docxtpl rule has a test that renders a bad template with the real docxtpl to prove that it really breaks.
3. **Own tag stack plus the Jinja AST.** Block problems are found with the linter's own tag stack, so all of them are reported at once. Jinja syntax and semantics (filters, tests, scopes, variable paths, types) come from parsing with Jinja2 and walking the AST. That same analysis builds the data contract for `infer`, `--data` and `--schema`.
4. **Real rendering where it matters.** `verify` and `check_context` render with docxtpl itself, using sentinel or your own data.
5. **Validated on real templates.** CI lints the 40 templates in docxtpl's own test suite (0 false positives; the deliberately broken one is found), plus 86 adversarial cases adjudicated by real renders ([benchmark](benchmark/expected.json), [adversarial cases](benchmark/adversarial_adjudicated.json)).
6. **Runs in the browser.** The pure-Python wheel loads in Pyodide for the playground.

## Limitations

1. **Static checks assume the data you give them.** Without `--data`/`--schema`, type errors are only found by `verify` or at render time. Custom Jinja extensions, `include`, and macro signatures are not checked.
2. **Parts docxtpl does not render are skipped:** endnotes, comments, charts, embedded objects. Text boxes are checked as part of their paragraph.
3. **`¶N` counts every `<w:p>` of the part**, including empty paragraphs inside table cells. Use the table path and the printed snippet to find the spot in Word.
4. **DTL201 follows docxtpl ≤ 0.20.2.** If upstream relaxes the regex (PR [#645](https://github.com/elapouya/python-docx-template/pull/645)), use `--ignore DTL201`.

## Related

[lintxtpl](https://github.com/splittist/lintxtpl) by splittist was the first linter for docxtpl templates, and the source of the annotated-`.docx` idea. Version history is in [CHANGELOG.md](CHANGELOG.md). See [CONTRIBUTING.md](CONTRIBUTING.md) to contribute.

## License

MIT
