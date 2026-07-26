---
title: "Parse Pipe-Delimited Tables with Continuation Rows"
snippet_type: algorithm
use_cases:
  - parsing
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - split-quoted-and-bracketed-log-fields.md
  - normalize-optional-csv-columns-in-a-single-pass.md
---

# Parse Pipe-Delimited Tables with Continuation Rows

## Idea and Problem

Parse one deliberately small pipe-delimited table while preserving non-table text and malformed post-header rows for inspection.

The first ordinary pipe row defines unique, non-empty column names. Later rows
must have the same width; a row with an empty first cell appends its non-empty
cells to the preceding record. An invalid header aborts parsing; after a valid
header, unsupported or malformed nonblank lines are kept separately instead of
being guessed into corrupted data.

## When to Use

Use this parser for a controlled command-line or report format that guarantees
outer pipes, has no escaping, and uses empty-first-cell rows for multiline
continuations. Confirm the grammar with representative fixtures. Prefer JSON,
CSV, or a format-specific library whenever the producer offers a stable
machine-readable alternative.

## Implementation

```python
from collections.abc import Iterable
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class ParsedPipeTable:
    rows: tuple[dict[str, str], ...]
    extra_lines: tuple[str, ...]


def _pipe_cells(line: str) -> tuple[str, ...] | None:
    stripped = line.strip()
    if not stripped.startswith("|") or not stripped.endswith("|"):
        return None
    return tuple(cell.strip() for cell in stripped[1:-1].split("|"))


def _is_separator_row(cells: tuple[str, ...]) -> bool:
    return bool(cells) and all(
        len(cell) >= 3 and "-" in cell and set(cell) <= {"-", ":", "="}
        for cell in cells
    )


def parse_pipe_table(lines: Iterable[str]) -> ParsedPipeTable:
    if isinstance(lines, str):
        raise TypeError("lines must be an iterable of complete text lines")
    header: tuple[str, ...] | None = None
    rows: list[dict[str, str]] = []
    extra_lines: list[str] = []

    for raw_line in lines:
        if not isinstance(raw_line, str):
            raise TypeError("lines must contain text")
        line = raw_line.rstrip("\r\n")
        cells = _pipe_cells(line)

        if cells is None:
            if line.strip():
                extra_lines.append(line)
            continue

        if header is None:
            if _is_separator_row(cells):
                extra_lines.append(line)
                continue
            if not cells or any(not name for name in cells):
                raise ValueError("table headers must be non-empty")
            if len(set(cells)) != len(cells):
                raise ValueError("table headers must be unique")
            header = cells
            continue

        if len(cells) != len(header):
            extra_lines.append(line)
            continue
        if _is_separator_row(cells):
            continue

        if cells[0] == "":
            if not rows or not any(cells[1:]):
                extra_lines.append(line)
                continue
            previous = rows[-1]
            for name, value in zip(header[1:], cells[1:]):
                if value:
                    existing = previous[name]
                    previous[name] = f"{existing}\n{value}" if existing else value
            continue

        rows.append(dict(zip(header, cells)))

    return ParsedPipeTable(rows=tuple(rows), extra_lines=tuple(extra_lines))
```

## Example

```python
parsed = parse_pipe_table(
    [
        "warning: cached output\r\n",
        "| id | status |\n",
        "| --- | :---: |\n",
        "| 1 | queued |\n",
        "|   | retried |\n",
        "| 2 | ready |\n",
        "| too | many | cells |\n",
        "summary: 2 rows\n",
    ]
)

try:
    parse_pipe_table(["| name | name |"])
except ValueError:
    duplicate_header_rejected = True
else:
    duplicate_header_rejected = False

assert (
    parsed,
    duplicate_header_rejected,
) == (
    ParsedPipeTable(
        rows=(
            {"id": "1", "status": "queued\nretried"},
            {"id": "2", "status": "ready"},
        ),
        extra_lines=(
            "warning: cached output",
            "| too | many | cells |",
            "summary: 2 rows",
        ),
    ),
    True,
)
```

## Trade-offs and Limitations

This is not a general Markdown, terminal-table, or CSV parser. Pipes inside
cells, quoting, escaping, column spans, wrapped first-column values, multiple
table blocks, and display-width rules are unsupported. Blank non-table lines
are ignored, while malformed nonblank lines are preserved in `extra_lines`.
Continuation rows mutate the most recent result dictionary during parsing, and
the returned dictionaries remain mutable. The function consumes a finite
iterable and materializes all rows and extra lines in memory.

## Related Snippets

<!-- catalog:related:start -->
- [Split Quoted and Bracketed Log Fields](split-quoted-and-bracketed-log-fields.md)
- [Normalize Optional CSV Columns in a Single Pass](normalize-optional-csv-columns-in-a-single-pass.md)
<!-- catalog:related:end -->
