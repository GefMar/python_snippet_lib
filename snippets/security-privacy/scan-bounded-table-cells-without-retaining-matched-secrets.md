---
title: "Scan Bounded Table Cells Without Retaining Matched Secrets"
snippet_type: recipe
use_cases:
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/validate-parsed-csv-rows-with-bounded-structured-problems.md
  - separate-executable-and-redacted-views-of-a-command-argument-vector.md
  - redact-a-printable-ascii-secret-with-a-bounded-visible-tail.md
---

# Scan Bounded Table Cells Without Retaining Matched Secrets

## Idea and Problem

Scan a small immutable table with one closed assignment-token heuristic while returning only bounded, value-free finding coordinates.

The function validates the complete rectangular input before scanning. It then
walks rows in tuple order, columns in schema order, and matches from left to
right. Every frozen finding carries only a conservative row identifier, a
column name, the closed rule kind, and a one-based occurrence ordinal within
that cell; it never carries the match, surrounding text, a masked suffix,
entropy, or a secret-derived fingerprint.

## When to Use

Use this recipe after a trusted parser has already materialized a small table
as exact strings and a caller needs passive triage without preserving candidate
values in the result. Row identifiers and column names must be unique and use
the narrow grammars below. The fixed detector recognizes several common
credential-like labels followed by `=`, `:=`, or `:` and an 8-96 character
ASCII token shape. Callers cannot supply regular expressions or callbacks.

Keep parsing, authorization, remediation, and display outside this boundary.
Pass only the required columns, keep the returned coordinates access-controlled,
and inspect any candidate value through a separately reviewed workflow.

## Implementation

```python
import re
from dataclasses import dataclass, field
from typing import Literal

RuleKind = Literal["assignment-token"]

_MAX_ROWS = 256
_MAX_COLUMNS = 32
_MAX_CELLS = 4_096
_MAX_CELL_UTF8_BYTES = 4_096
_MAX_TOTAL_UTF8_BYTES = 512 * 1_024
_MAX_FINDINGS = 256
_ROW_ID = re.compile(r"[A-Za-z0-9][A-Za-z0-9._-]{0,63}", re.ASCII)
_COLUMN_NAME = re.compile(r"[a-z][a-z0-9_]{0,47}", re.ASCII)
_ASSIGNMENT_TOKEN = re.compile(
    r"(?<![A-Za-z0-9_-])"
    r"(?:api[_-]?key|access[_-]?token|auth[_-]?token|client[_-]?secret|"
    r"password|passwd|secret)"
    r"[ \t]*(?::=|=|:)[ \t]*"
    r"(?:'[A-Za-z0-9][A-Za-z0-9._~+/=-]{7,95}'|"
    r'"[A-Za-z0-9][A-Za-z0-9._~+/=-]{7,95}"|'
    r"[A-Za-z0-9][A-Za-z0-9._~+/=-]{7,95})"
    r"(?![A-Za-z0-9._~+/=-])",
    re.ASCII | re.IGNORECASE,
)


@dataclass(frozen=True, slots=True)
class TableRow:
    row_id: str = field(repr=False)
    cells: tuple[str, ...] = field(repr=False)


@dataclass(frozen=True, slots=True)
class CellFinding:
    row_id: str
    column: str
    kind: RuleKind
    occurrence_ordinal: int


def _validated_columns(value: object) -> tuple[str, ...]:
    if type(value) is not tuple:
        raise TypeError("columns must be an exact tuple")
    if not 1 <= len(value) <= _MAX_COLUMNS:
        raise ValueError("column count is outside the supported range")

    seen: set[str] = set()
    for column in value:
        if type(column) is not str:
            raise TypeError("column names must be exact strings")
        if _COLUMN_NAME.fullmatch(column) is None:
            raise ValueError("a column name is malformed")
        if column in seen:
            raise ValueError("column names must be unique")
        seen.add(column)
    return value


def _validated_rows(
    value: object,
    *,
    width: int,
) -> tuple[TableRow, ...]:
    if type(value) is not tuple:
        raise TypeError("rows must be an exact tuple")
    if len(value) > _MAX_ROWS:
        raise ValueError("row count exceeds the supported limit")
    if len(value) * width > _MAX_CELLS:
        raise ValueError("cell count exceeds the supported limit")

    seen_row_ids: set[str] = set()
    total_bytes = 0
    for row in value:
        if type(row) is not TableRow:
            raise TypeError("rows must contain exact TableRow values")
        if type(row.row_id) is not str:
            raise TypeError("row identifiers must be exact strings")
        if _ROW_ID.fullmatch(row.row_id) is None:
            raise ValueError("a row identifier is malformed")
        if row.row_id in seen_row_ids:
            raise ValueError("row identifiers must be unique")
        seen_row_ids.add(row.row_id)

        if type(row.cells) is not tuple:
            raise TypeError("row cells must be an exact tuple")
        if len(row.cells) != width:
            raise ValueError("a row has the wrong cell count")
        for cell in row.cells:
            if type(cell) is not str:
                raise TypeError("cells must be exact strings")
            if len(cell) > _MAX_CELL_UTF8_BYTES:
                raise ValueError("a cell exceeds the UTF-8 byte limit")
            try:
                encoded = cell.encode("utf-8")
            except UnicodeEncodeError:
                raise ValueError("cells must be encodable as UTF-8") from None
            byte_count = len(encoded)
            if byte_count > _MAX_CELL_UTF8_BYTES:
                raise ValueError("a cell exceeds the UTF-8 byte limit")
            if byte_count > _MAX_TOTAL_UTF8_BYTES - total_bytes:
                raise ValueError("aggregate cell bytes exceed the supported limit")
            total_bytes += byte_count
    return value


def scan_table_cells(
    columns: tuple[str, ...],
    rows: tuple[TableRow, ...],
) -> tuple[CellFinding, ...]:
    validated_columns = _validated_columns(columns)
    validated_rows = _validated_rows(rows, width=len(validated_columns))

    findings: list[CellFinding] = []
    for row in validated_rows:
        for column, cell in zip(validated_columns, row.cells, strict=True):
            occurrence_ordinal = 0
            for _match in _ASSIGNMENT_TOKEN.finditer(cell):
                occurrence_ordinal += 1
                if len(findings) == _MAX_FINDINGS:
                    raise ValueError("finding count exceeds the supported limit")
                findings.append(
                    CellFinding(
                        row_id=row.row_id,
                        column=column,
                        kind="assignment-token",
                        occurrence_ordinal=occurrence_ordinal,
                    )
                )
    return tuple(findings)
```

## Example

```python
# Every marker value below is synthetic and is not a credential.
columns = ("summary", "details")
rows = (
    TableRow(
        "row-001",
        (
            "api_key=placeholder",
            "plain note; password: placeholder; auth-token=placeholder",
        ),
    ),
    TableRow(
        "row-002",
        ("ordinary text", "client_secret='placeholder'"),
    ),
)

findings = scan_table_cells(columns, rows)

assert findings == (
    CellFinding("row-001", "summary", "assignment-token", 1),
    CellFinding("row-001", "details", "assignment-token", 1),
    CellFinding("row-001", "details", "assignment-token", 2),
    CellFinding("row-002", "details", "assignment-token", 1),
)
```

## Trade-offs and Limitations

This is a deliberately small heuristic, not a credential verifier. It can flag
documentation, fixtures, or ordinary configuration whose labels and values fit
the grammar. It can miss secrets under unknown labels, values shorter or longer
than the fixed token shape, Unicode or whitespace-heavy values, multiline
assignments, encoded containers, split strings, and credentials without an
assignment. Expanding the rule deserves separate tests and review rather than
caller-provided patterns.

All names, rows, cells, byte totals, and findings are validated or capped before
a result is returned. An overrun raises a static, value-free exception instead
of returning a partial prefix. Input-row representation hides both fields, and
the function never slices a matched value or places cell text in a finding,
exception message, explicit error field, log, digest, or fingerprint. It
performs no table or distributed data access, logging, persistence, mutation,
callback, or I/O.

Python cannot erase the caller's immutable strings from memory. The regular
expression engine also holds temporary references while scanning. Every raised
exception automatically retains traceback frames whose locals can include the
rows, current cell, and match object; do not retain exception objects, and
disable traceback-local capture at this boundary. If even transient traceback
linkage is forbidden, use a separately designed structured over-limit outcome
instead of raising from the scan. Minimize the supplied table, discard it
promptly under the caller's own lifetime policy, and use a lower-level
controlled-memory design when memory erasure is a requirement.

## Related Snippets

<!-- catalog:related:start -->
- [Validate Parsed CSV Rows with Bounded Structured Problems](../data-processing/validate-parsed-csv-rows-with-bounded-structured-problems.md)
- [Separate Executable and Redacted Views of a Command Argument Vector](separate-executable-and-redacted-views-of-a-command-argument-vector.md)
- [Redact a Printable ASCII Secret with a Bounded Visible Tail](redact-a-printable-ascii-secret-with-a-bounded-visible-tail.md)
<!-- catalog:related:end -->
