# Changelog

All notable changes to this project are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project uses
[Semantic Versioning](https://semver.org/).

## [Unreleased]

### Changed

Root `README.md` and `README.zh.md`: reader-facing Markdown unordered lists were converted to ordered lists. Code blocks and tables are unchanged. No functional change.

## [2.0.1] - 2026-10-16

First release of 2.0. The `v2.0.0` tag was pushed but never released: CI's ruff import-order check failed on test files, so 2.0.0 is not on PyPI. 2.0.1 = 2.0.0 + sorted test imports.

Migration guide: [docs/MIGRATION-2.0.md](docs/MIGRATION-2.0.md).

### Changed (breaking)
- `verify` reads `[tool.docxtpl-lint]` (`jinja-env`, `autoescape`, `select`, `ignore`, `fail-on`, `baseline`, `lang`, …, but not `data`), so it renders like lint checks. Before, a project with `jinja-env` in its config got false DTL402 from verify. `verify --no-config` restores 1.x behaviour.
- `verify` supports every lint output option: `--format text|json|github|sarif|gitlab|markdown`, `--output`, `--report`, `--select`/`--ignore`, `--fail-on`, `--baseline`/`--write-baseline`, `--annotate`.

### Removed
- `--warnings-as-errors` and config `warnings-as-errors` (deprecated in 1.9). Use `--fail-on warning` / `fail-on = "warning"`.

### Added
- `fingerprint` on every JSON issue and in `Result.to_dict()` (same id as SARIF `partialFingerprints` and GitLab).
- `diff --fail-on breaking|change|never`.

### Deprecated (removal in 3.0)
- `docxtpl_lint.lint_path`: use `docxtpl_lint.lint`.
- `diff --fail-on-breaking`: use `--fail-on breaking`.

## [1.9.1] - 2026-10-16

First release of the 1.9 line. The `v1.9.0` tag was pushed but never released: CI's `ruff format --check` (which also formats code blocks in README) failed on it, so 1.9.0 is not on PyPI. 1.9.1 = 1.9.0 + README formatting.

### Added
- Stable Python API `docxtpl_lint.api` (also top-level): `lint(template, data, ...)`, `check_context(template, context)` (lint + in-memory docxtpl render, a pre-render guard), `infer_schema(paths)`, `Result` (`issues`, `errors`, `warnings`, `ok`, `variables`, `rendered`, `raise_for_errors()`, `to_dict()`), `TemplateLintError`. Templates can be paths, bytes or binary file objects. Stability policy: docs/api.md.
- `--fail-on error|warning|never` and config `fail-on`.

### Fixed
- `--jinja-env` with an autoescaping environment no longer reports DTL308; a callable autoescape (`select_autoescape`) is evaluated like docxtpl does.
- `verify --autoescape --jinja-env` no longer changes your environment object (docxtpl sets `env.autoescape` in place; we render via an overlay).

### Deprecated
- `--warnings-as-errors` / `warnings-as-errors`: use `--fail-on warning`. Removed in 2.0.

## [1.8.0] - 2026-10-16

### Added
- `--format markdown` (CI job summary / PR comment) and `--format gitlab` (GitLab Code Quality report).
- Stable finding fingerprints (part + table cell + paragraph text + occurrence): GitLab `fingerprint` and SARIF `partialFingerprints`, unchanged when paragraphs are inserted elsewhere.
- `--output FILE` and repeatable `--report FORMAT:FILE` (several reports from one run).
- GitHub Action inputs: `summary` (Markdown job summary, on by default), `baseline`, `sarif` (path for `upload-sarif`), `annotate` (uploads copies of failing templates with Word comments as the `docxtpl-lint-annotated` artifact).

## [1.7.0] - 2026-10-16

### Added
- `fix --write OUT`: apply fix suggestions to a copy of the template (`OUT` is a `.docx` path, or a directory for several templates). Edits are made inside the existing runs, so formatting is kept. Passes repeat until no fix applies, and a pass is kept only if the number of findings drops. The template itself is never modified.
- Fix safety: a missing `}`, a space before `tr`, a prefix that must match its pair and a misspelt Jinja keyword are `safe`. Renaming a path after `--data`, removing a stray tag or dropping a `tr`/`tc` prefix are `unsafe` and need `--unsafe`. JSON output has a `safe` field.

## [1.6.0] - 2026-10-16

### Added
- DTL308 (error, with `--data`): a string with `&` or `<` printed without `|e`. docxtpl's default `autoescape=False` silently cuts it: `Smith & Co` → `Smith  Co`, and `<b>` drops the rest of the paragraph. Silence with `--autoescape` / `autoescape = true` if you render with `autoescape=True`.
- `verify`: `{{r}}` / `{{p}}` outputs are checked with RichText / Subdoc sentinels.
- `verify`: repeatable `--data`, plus `--autoescape`, `--jinja-env MODULE:ATTR` and `--out DIR` (save the rendered documents).

## [1.5.0] - 2026-10-16

### Added
- DTL306 (error, with `--data`): a list filter (`join`, `sum`, `map`, `selectattr`, …) applied to a non-list value. A number raises, a string is split into characters, and a mapping yields its keys.
- DTL307 (warning, with `--data`): `{{r x}}` / `{{p x}}` given a plain value, which docxtpl renders as nothing.
- `infer` / Pydantic / `diff`: `{{r}}` and `{{p}}` paths are marked `x-docxtpl-object: RichText|Subdoc`; plain → RichText is a breaking change.

### Changed
- The contract understands list and attribute filters: `xs|join` → `xs` is an array; `items|sum(attribute='price')` → `items[].price` is a number; `for i in items|selectattr('ok')|sort(attribute='n')` keeps `i` bound to `items[]`.

## [1.4.0] - 2026-10-16

### Added
- DTL003 (error): template code inside an unaccepted tracked deletion. docxtpl still renders it, and the value ends up in the generated document's revision history.
- Field codes are linted: `{{ … }}` inside `HYPERLINK "…"` / other field instructions (`w:instrText`, `w:fldSimple`) is rendered by docxtpl, so it is now syntax-checked and part of `--list-vars` / `infer` / `--data` checks.

### Changed
- Paragraphs with tracked deletions or field codes are read the way docxtpl's `patch_xml` sees them: a deletion inside a tag is dropped, and deleted text outside tags is kept. Other paragraphs are unchanged.
- The weekly docxtpl-upstream CI job also runs the 1.3/1.4 render tests.

## [1.3.0] - 2026-10-16

### Added
- DTL104 unknown Jinja filter, DTL105 unknown Jinja test (docxtpl raises `TemplateAssertionError` / `TemplateRuntimeError`).
- DTL106 loop variable, `loop` or a `set` made inside a loop used after the loop (renders empty or raises `UndefinedError`).
- `--filters`, `--tests`, `--jinja-env MODULE:ATTR` and the `filters` / `tests` / `jinja-env` config keys for templates rendered with a custom `jinja_env`.

### Changed
- Loop-local names used outside their loop are no longer listed as top-level data by `--list-vars` / `infer`.

## [1.2.0] - 2026-10-16

Ideas adopted from a survey of other open-source linters and template engines
([docs/research/oss-survey.md](docs/research/oss-survey.md), [docs/PLAN-v1.2.md](docs/PLAN-v1.2.md)).

### Added
- Paragraph-level suppression: `{# docxtpl-lint: disable-line=RULES #}` and `{#p docxtpl-lint: disable-next=RULES #}`
  (ids or prefixes), after ruff `noqa` / djLint.
- DTL005: a paragraph suppression that matches nothing, or names an unknown rule (after ruff RUF100).
- `--write-baseline FILE` / `--baseline FILE` (also `baseline` in `[tool.docxtpl-lint]`): accept existing findings and
  fail only on new ones. Keys are part + table cell + paragraph-text hash, with counts (after ESLint bulk suppressions).
- `--statistics`: count per rule (after ruff).
- `--annotate DIR`: write `<name>.annotated.docx` copies with one Word comment per paragraph with findings (after
  lintxtpl). Originals are never modified.
- `infer --pydantic FILE`: Pydantic v2 models for the inferred contract (after datamodel-code-generator, no new
  dependency).
- README: reviewdog recipe (`--format sarif | reviewdog -f=sarif`).
- `action.yml`: `author`.

### Fixed
- `--lang zh`: the location of the opening tag in DTL204 and the unit name in DTL209 (`table row` → `表格行`) were left
  in English.

## [1.1.0] - 2026-10-16

Found by an adversarial run (86 cases designed by DeepSeek, disputes settled by rendering with docxtpl) and a
self-evaluation from a fresh install (docs/eval/).

### Added
- DTL210: `{%p %}` block opened and closed in the same paragraph (docxtpl raises).
- `--explain RULE`.
- `fix` suggests the missing `}` of a `{{ … }` tag.
- `scripts/adversarial.py` + `benchmark/adversarial_adjudicated.json`: the 86 cases run in the test-suite.
- `scripts/realworld.py`: invoice / Chinese contract / broken / 3000-paragraph templates used by the self-evaluation.

### Fixed
- `}}` or `%}` inside a string literal no longer ends the tag (false DTL103).
- A tag missing its closing brace no longer swallows the following paragraphs (it was only a DTL002 warning and
  hid later errors); it is reported as DTL001 and scanning continues at the next tag.
- Unclosed `{% raw %}` is reported (DTL101).
- DTL209: comments are not counted as lost text; text in other cells of a `{%tr %}` row and in other paragraphs of a
  `{%tc %}` cell is now checked; no DTL209 next to DTL201/202/203 or a same-paragraph `{%p %}` pair.
- `--data` / `--schema` JSON with a UTF-8 BOM (Windows Notepad) is accepted.
- `infer` / `verify` / `diff` / `fix` exit with code 2 and a one-line message on unreadable input instead of a traceback.
- Playground: the 1.0.0 page failed to load (escaped newlines in the embedded Python); the Pyodide smoke test now runs
  the page's own code. A hint appears if loading takes over 30 s.
- `fix` diff headers: no `a//tmp/...` double slash.

### Changed
- `infer` writes `x-docxtpl-templates` only when several templates are merged.
- Clearer `--data` help text.

## [1.0.0] - 2026-10-16

### Added
- `docxtpl-lint infer`: data contract of one or more templates as JSON Schema (2020-12) plus sample data; lists from
  `for` loops, numbers from `round`/`int`/`float`/`abs`, arithmetic and numeric comparisons, objects from `.items()`,
  optional fields from `is defined`/`default`; merged schemas list the templates using each field
  (`x-docxtpl-templates`).
- `docxtpl-lint verify` (optional extra `[render]`): render with the real docxtpl and sentinel data; DTL401 = rendered
  value silently missing from the output, DTL402 = render fails.
- `docxtpl-lint diff old.docx new.docx [--fail-on-breaking]`: data contract changes between template versions,
  with rename hints.
- `--schema schema.json`: DTL304 (variable not declared), DTL305 (loop over a non-iterable type).
- `docxtpl-lint fix`: fix suggestions as unified diffs of the paragraph text for DTL202, DTL203, DTL204, DTL102 (stray tag
  or misspelt opening tag), DTL103 (unknown tag) and DTL301 (with `--data`); conservative matching (≤ 2 edits, no ties).
- DTL209: text next to a `{%p %}`/`{%tr %}`/`{%tc %}` tag is silently dropped by docxtpl (found with `verify`).
- Playground: fix suggestions, inferred JSON Schema and sample data.
- Benchmark runs `verify` on docxtpl's 40 templates (37 render with inferred data, 0 false DTL401).

### Changed
- Pages deploy runs on pushes to `main` (the `release` trigger ran on the tag ref, which the `github-pages` environment
  rejects).
- Development status: Production/Stable.

## [0.2.0] - 2026-10-16

### Added
- Browser playground (Pyodide, runs fully client-side), deployed to GitHub Pages by `pages.yml`.
- `[tool.docxtpl-lint]` configuration in `pyproject.toml` (`--config`, `--no-config`); unknown keys are errors.
- `--context N`: show neighbouring paragraphs next to each issue (default 1 on a terminal, 0 otherwise).
- DTL208: `cellbg`/`colspan`/`vm`/`hm` tag outside a table cell.
- `docs/rules.md` with one anchor per rule; `help_url` in JSON output and `helpUri` in SARIF.
- Benchmark against docxtpl's own 40 test templates (`scripts/benchmark.py`, CI job); stale allow-list entries fail it.
- Manual-only PyPI Trusted Publishing workflow (`publish-pypi.yml`).

### Fixed
- False positives on docxtpl cell tags (`{% cellbg %}`, `{% colspan %}`, `{% vm %}`, `{% hm %}`) and on Word smart quotes inside tags: 7 → 0 on the benchmark.

### Changed
- GitHub Actions updated to Node 24 versions (`checkout@v7`, `setup-python@v7`, ...); the composite action uses `setup-python@v7`.
- New dependency on Python < 3.11: `tomli`.

## [0.1.0] - 2026-10-16

### Added
- First release.
- Locations down to the table cell (`T1 R2 C2 > T2 R2 C1`, nested tables) for body, headers, footers and footnotes.
- Tag and block rules: DTL001 (tag never closed), DTL002 (tag spans paragraphs, warning), DTL101 (unclosed block),
  DTL102 (stray end/else), DTL103 (other Jinja2 syntax errors, using docxtpl's default environment).
- docxtpl tag rules: DTL201 (`%`/`}` in prefixed tags, #644), DTL202 (space before prefix), DTL203 (tr/tc outside
  a table), DTL204 (prefix mismatch, #603), DTL205 (tr block spans tables), DTL206 (tr open/close in the same row),
  DTL207 (tc block spans rows).
- Data rules with `--data` (repeatable): DTL301 (missing path, nested and loop-aware, respects `is defined` /
  `| default`), DTL302 (unused key, with `--report-unused-data`), DTL303 (loop over a non-iterable).
- `--format text|json|github|sarif`, `--lang en|zh`, `--select`, `--ignore`, `--warnings-as-errors`, `--list-vars`,
  `--list-rules`, `--allow-cross-paragraph`; exit codes 0/1/2.
- File-level suppression comment `{# docxtpl-lint: disable=… #}`.
- GitHub Action, pre-commit hook, and test cases checked against real docxtpl 0.20.2 rendering.

[Unreleased]: https://github.com/yq6666-66/docxtpl-lint/compare/v2.0.1...HEAD
[2.0.1]: https://github.com/yq6666-66/docxtpl-lint/compare/v1.9.1...v2.0.1
[1.9.1]: https://github.com/yq6666-66/docxtpl-lint/compare/v1.8.0...v1.9.1
[1.8.0]: https://github.com/yq6666-66/docxtpl-lint/compare/v1.7.0...v1.8.0
[1.7.0]: https://github.com/yq6666-66/docxtpl-lint/compare/v1.6.0...v1.7.0
[1.6.0]: https://github.com/yq6666-66/docxtpl-lint/compare/v1.5.0...v1.6.0
[1.5.0]: https://github.com/yq6666-66/docxtpl-lint/compare/v1.4.0...v1.5.0
[1.4.0]: https://github.com/yq6666-66/docxtpl-lint/compare/v1.3.0...v1.4.0
[1.3.0]: https://github.com/yq6666-66/docxtpl-lint/compare/v1.2.0...v1.3.0
[1.2.0]: https://github.com/yq6666-66/docxtpl-lint/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/yq6666-66/docxtpl-lint/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/yq6666-66/docxtpl-lint/compare/v0.2.0...v1.0.0
[0.2.0]: https://github.com/yq6666-66/docxtpl-lint/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/yq6666-66/docxtpl-lint/releases/tag/v0.1.0
