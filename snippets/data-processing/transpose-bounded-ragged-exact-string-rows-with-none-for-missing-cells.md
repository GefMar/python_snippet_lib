---
title: "Transpose Bounded Ragged Exact-String Rows with None for Missing Cells"
snippet_type: recipe
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - normalize-optional-csv-columns-in-a-single-pass.md
  - read-bounded-csv-text-into-pandas-under-an-explicit-schema.md
  - reshape-repeated-pandas-column-families-with-wide-to-long.md
---

# Transpose Bounded Ragged Exact-String Rows with None for Missing Cells

## Idea and Problem

Transpose bounded ragged string rows while preserving the difference between an empty string and a structurally missing cell.

Validate the complete source before using `itertools.zip_longest` to form
columns. Because source cells must be exact strings, `None` is an unambiguous
marker for a row that ended before a later column.

## When to Use

Use this recipe for a finite in-memory table whose rows may have different
lengths but whose existing cells are all text. It fits import normalization,
small report transformations, and other boundaries where callers must retain
empty strings while aligning columns.

Choose the output shape and missing-cell marker as part of the public contract.
Use a schema-aware tabular library when columns have types, names, nullability,
or vectorized operations, and use a streaming design when the complete input
cannot be validated in memory first.

## Implementation

```python
from itertools import zip_longest

_MAX_ROWS = 256
_MAX_WIDTH = 256
_MAX_SOURCE_CELLS = 16_384
_MAX_PADDED_CELLS = 32_768
_MAX_CELL_BYTES = 4_096
_MAX_TOTAL_BYTES = 1_048_576


def transpose_ragged_string_rows(
    rows: tuple[tuple[str, ...], ...],
) -> tuple[tuple[str | None, ...], ...]:
    """Return columns padded with None where a source row has no cell."""
    if type(rows) is not tuple:
        raise TypeError("rows must be an exact tuple")
    if len(rows) > _MAX_ROWS:
        raise ValueError(f"rows must contain at most {_MAX_ROWS} items")

    source_cells = 0
    total_bytes = 0
    max_width = 0

    for row_index, row in enumerate(rows):
        if type(row) is not tuple:
            raise TypeError(f"rows[{row_index}] must be an exact tuple")
        if len(row) > _MAX_WIDTH:
            raise ValueError(f"rows[{row_index}] exceeds the supported width")

        source_cells += len(row)
        if source_cells > _MAX_SOURCE_CELLS:
            raise ValueError("source cell count exceeds the supported limit")
        max_width = max(max_width, len(row))

        for column_index, cell in enumerate(row):
            if type(cell) is not str:
                raise TypeError(f"rows[{row_index}][{column_index}] must be an exact string")
            if len(cell) > _MAX_CELL_BYTES:
                raise ValueError(f"rows[{row_index}][{column_index}] exceeds the byte limit")
            try:
                encoded = cell.encode("utf-8")
            except UnicodeEncodeError:
                raise ValueError(
                    f"rows[{row_index}][{column_index}] is not valid UTF-8 text"
                ) from None

            cell_bytes = len(encoded)
            if cell_bytes > _MAX_CELL_BYTES:
                raise ValueError(f"rows[{row_index}][{column_index}] exceeds the byte limit")
            if cell_bytes > _MAX_TOTAL_BYTES - total_bytes:
                raise ValueError("source text exceeds the total byte limit")
            total_bytes += cell_bytes

    padded_cells = len(rows) * max_width
    if padded_cells > _MAX_PADDED_CELLS:
        raise ValueError("padded cell count exceeds the supported limit")

    return tuple(tuple(column) for column in zip_longest(*rows, fillvalue=None))
```

## Example

```python
rows = (
    ("alpha", "", "tail"),
    ("beta",),
    (),
    ("gamma", "value"),
)
columns = transpose_ragged_string_rows(rows)

try:
    transpose_ragged_string_rows((("\ud800",),))
except ValueError:
    surrogate_rejected = True
else:
    surrogate_rejected = False

assert (columns, surrogate_rejected) == (
    (
        ("alpha", "beta", None, "gamma"),
        ("", None, None, "value"),
        ("tail", None, None, None),
    ),
    True,
)
```

## Trade-offs and Limitations

Validation inspects every source cell and its UTF-8 encoding. A character-count
precheck rejects definitely oversized cells before encoding; the encoded-byte
check remains authoritative for multibyte text. Construction takes
`O(rows * max_width)` time and output slots, which can be much larger than the
source for sparse rows; the padded-cell limit bounds that expansion. Output
tuples reuse the original strings, while temporary encoded byte strings are not
retained.

`None` always means that a row had no cell at that position, while `""` remains
a real value. Empty outer input and any positive number of entirely empty rows
both produce `()`, so the result does not preserve row count when there are no
columns and is not a general round-trip format. Inputs must be exact tuples of
exact strings; subclasses, non-text cells, and strings containing lone Unicode
surrogates are rejected. The recipe performs no Unicode normalization, parsing,
type inference, streaming, or mutation of the source rows.

## Related Snippets

<!-- catalog:related:start -->
- [Normalize Optional CSV Columns in a Single Pass](normalize-optional-csv-columns-in-a-single-pass.md)
- [Read Bounded CSV Text into pandas Under an Explicit Schema](read-bounded-csv-text-into-pandas-under-an-explicit-schema.md)
- [Reshape Repeated pandas Column Families with Wide-to-Long](reshape-repeated-pandas-column-families-with-wide-to-long.md)
<!-- catalog:related:end -->
