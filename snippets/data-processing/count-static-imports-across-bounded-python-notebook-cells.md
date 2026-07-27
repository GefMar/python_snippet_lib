---
title: "Count Static Imports Across Bounded Python Notebook Cells"
snippet_type: algorithm
use_cases:
  - data-transformation
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - collect-expected-parse-failures-without-stopping-a-batch.md
  - ../configuration-serialization/evaluate-a-bounded-boolean-tag-expression-with-an-ast-allowlist.md
  - ../python-language/collect-decorated-methods-in-class-definition-order.md
---

# Count Static Imports Across Bounded Python Notebook Cells

## Idea and Problem

Summarize static top-level module imports from bounded notebook-shaped data without executing or rewriting any cell source.

Each absolute module counts at most once per code cell, so repeated imports do
not inflate cell-level usage. Non-code, empty, extension-syntax, syntax-error,
no-import, and non-Python code cells remain visible as separate counters rather
than disappearing from the report.

## When to Use

Use this algorithm for a conservative dependency inventory over already decoded
notebook JSON. The cell count, source size, AST size, and distinct module
count must fit the hard limits. A declared non-Python notebook is classified
without attempting Python parsing.

Use the maintained notebook-format library when full schema validation or
format conversion matters. Use runtime instrumentation when dynamic imports are
part of the question. Never execute untrusted cells merely to improve a static
inventory.

## Implementation

```python
import ast
from collections import Counter
from collections.abc import Mapping
from dataclasses import dataclass


_MAX_CELLS = 1_024
_MAX_SOURCE_BYTES_PER_CELL = 64 * 1024
_MAX_SOURCE_BYTES_TOTAL = 1024 * 1024
_MAX_AST_NODES_PER_CELL = 20_000
_MAX_MODULES = 256
_MAX_SOURCE_LINES = 8_192


@dataclass(frozen=True, slots=True)
class NotebookImportSummary:
    total_cells: int
    code_cells: int
    non_code_cells: int
    empty_code_cells: int
    extension_code_cells: int
    syntax_error_cells: int
    no_import_code_cells: int
    non_python_code_cells: int
    modules: tuple[tuple[str, int], ...]


def _cell_source(cell: Mapping[str, object]) -> tuple[str, int]:
    source = cell.get("source", "")
    if isinstance(source, str):
        text = source
    elif isinstance(source, list):
        if len(source) > _MAX_SOURCE_LINES or not all(
            isinstance(line, str) for line in source
        ):
            raise ValueError("cell source lines are outside the supported format")
        text = "".join(source)
    else:
        raise TypeError("cell source must be text or a list of text lines")
    encoded_size = len(text.encode("utf-8"))
    if encoded_size > _MAX_SOURCE_BYTES_PER_CELL:
        raise ValueError("a cell source exceeds the supported size")
    return text, encoded_size


def _declared_language(notebook: Mapping[str, object]) -> str | None:
    metadata = notebook.get("metadata", {})
    if not isinstance(metadata, Mapping):
        raise TypeError("notebook metadata must be a mapping")
    language_info = metadata.get("language_info", {})
    if not isinstance(language_info, Mapping):
        raise TypeError("language_info must be a mapping")
    language = language_info.get("name")
    if language is not None and not isinstance(language, str):
        raise TypeError("language_info.name must be text or absent")
    return language.casefold() if language is not None else None


def count_notebook_imports(notebook: Mapping[str, object]) -> NotebookImportSummary:
    if not isinstance(notebook, Mapping):
        raise TypeError("notebook must be a mapping")
    cells = notebook.get("cells")
    if not isinstance(cells, list):
        raise TypeError("notebook cells must be a list")
    if len(cells) > _MAX_CELLS:
        raise ValueError("notebook has too many cells")

    language = _declared_language(notebook)
    modules: Counter[str] = Counter()
    code_cells = 0
    non_code_cells = 0
    empty_code_cells = 0
    extension_code_cells = 0
    syntax_error_cells = 0
    no_import_code_cells = 0
    non_python_code_cells = 0
    source_bytes = 0

    for raw_cell in cells:
        if not isinstance(raw_cell, Mapping):
            raise TypeError("notebook cells must be mappings")
        cell_type = raw_cell.get("cell_type")
        if not isinstance(cell_type, str):
            raise TypeError("cell_type must be text")
        if cell_type != "code":
            non_code_cells += 1
            continue

        code_cells += 1
        source, encoded_size = _cell_source(raw_cell)
        source_bytes += encoded_size
        if source_bytes > _MAX_SOURCE_BYTES_TOTAL:
            raise ValueError("notebook source exceeds the aggregate size limit")
        if language not in (None, "python"):
            non_python_code_cells += 1
            continue
        if not source.strip():
            empty_code_cells += 1
            continue
        if any(
            line.lstrip().startswith(("%", "!"))
            for line in source.splitlines()
        ):
            extension_code_cells += 1
            continue

        try:
            tree = ast.parse(source)
        except SyntaxError:
            syntax_error_cells += 1
            continue
        except (MemoryError, RecursionError) as error:
            raise ValueError(
                "a cell source exceeds the parser complexity limit"
            ) from error
        cell_modules: set[str] = set()
        node_count = 0
        for node in ast.walk(tree):
            node_count += 1
            if node_count > _MAX_AST_NODES_PER_CELL:
                raise ValueError("a cell AST exceeds the supported node limit")
            if isinstance(node, ast.Import):
                cell_modules.update(
                    alias.name.split(".", 1)[0] for alias in node.names
                )
            elif isinstance(node, ast.ImportFrom) and node.level == 0 and node.module:
                cell_modules.add(node.module.split(".", 1)[0])
        if not cell_modules:
            no_import_code_cells += 1
            continue
        modules.update(cell_modules)
        if len(modules) > _MAX_MODULES:
            raise ValueError("notebook contains too many distinct modules")

    return NotebookImportSummary(
        total_cells=len(cells),
        code_cells=code_cells,
        non_code_cells=non_code_cells,
        empty_code_cells=empty_code_cells,
        extension_code_cells=extension_code_cells,
        syntax_error_cells=syntax_error_cells,
        no_import_code_cells=no_import_code_cells,
        non_python_code_cells=non_python_code_cells,
        modules=tuple(sorted(modules.items())),
    )
```

## Example

```python
notebook = {
    "metadata": {"language_info": {"name": "python"}},
    "cells": [
        {"cell_type": "markdown", "source": "Imports used below"},
        {
            "cell_type": "code",
            "source": ["import json\n", "import json.tool\n", "from pathlib import Path\n"],
        },
        {"cell_type": "code", "source": "from json import loads\n"},
        {"cell_type": "code", "source": "%time value\n"},
        {"cell_type": "code", "source": "if True print('broken')\n"},
        {"cell_type": "code", "source": "value = 3\n"},
    ],
}

summary = count_notebook_imports(notebook)

assert (
    summary.total_cells,
    summary.modules,
    summary.extension_code_cells,
    summary.syntax_error_cells,
    summary.no_import_code_cells,
) == (6, (("json", 2), ("pathlib", 1)), 1, 1, 1)
```

## Trade-offs and Limitations

Static AST inspection sees ordinary absolute import statements only. It does
not discover dynamic imports, executed strings, shell package installs,
imports hidden behind unsupported notebook extensions, or modules loaded by
dependencies. Relative imports are deliberately excluded because their top
level depends on package context.

Language metadata can be absent or inaccurate, and classifying any cell with a
line-leading extension marker is conservative. The result measures cells that
mention a module, not import execution frequency or dependency correctness.
The parser can reject a deeply structured source before an AST is available;
that case is reported as a complexity-limit failure rather than a syntax error.
For full notebook compatibility, validate and normalize with the official
format library before applying this smaller analysis boundary.

## Related Snippets

<!-- catalog:related:start -->
- [Collect Expected Parse Failures Without Stopping a Batch](collect-expected-parse-failures-without-stopping-a-batch.md)
- [Evaluate a Bounded Boolean Tag Expression with an AST Allowlist](../configuration-serialization/evaluate-a-bounded-boolean-tag-expression-with-an-ast-allowlist.md)
- [Collect Decorated Methods in Class Definition Order](../python-language/collect-decorated-methods-in-class-definition-order.md)
<!-- catalog:related:end -->
