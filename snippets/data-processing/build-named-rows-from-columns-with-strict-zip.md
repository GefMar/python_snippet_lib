---
title: "Build Named Rows from Columns with Strict Zip"
snippet_type: recipe
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - transpose-bounded-ragged-exact-string-rows-with-none-for-missing-cells.md
  - assemble-fixed-width-feature-slots-from-bounded-named-sources.md
---

# Build Named Rows from Columns with Strict Zip

## Idea and Problem

Combine independent feature columns into named rows without silently discarding the tail of a longer column.

Zip the columns with strict length checking and materialize the result before returning it. This makes it impossible for the caller to receive the first rows of an invalid result before a length mismatch is detected.

## When to Use

Use this for finite, small columns in a named mapping. Preserve input column order in each row, and let a mismatch fail the conversion rather than supplying padding.

## Implementation

```python
from collections.abc import Iterable, Mapping


def named_rows[T](columns: Mapping[str, Iterable[T]]) -> tuple[dict[str, T], ...]:
    names = tuple(columns)
    if not names:
        return ()
    aligned = zip(*(columns[name] for name in names), strict=True)
    return tuple(dict(zip(names, values, strict=True)) for values in aligned)
```

## Example

```python
rows = named_rows({"label": iter(("small", "large")), "count": iter((2, 7))})
try:
    named_rows({"label": ["small", "extra"], "count": [2]})
except ValueError:
    mismatched_columns = True
else:
    mismatched_columns = False

assert (rows, tuple(rows[0]), named_rows({}), mismatched_columns) == (
    ({"label": "small", "count": 2}, {"label": "large", "count": 7}),
    ("label", "count"),
    (),
    True,
)
```

## Trade-offs and Limitations

Materialization costs memory proportional to the output. A mismatch still consumes input iterators and does not undo their side effects. Do not pass the same single-pass iterator as two columns when the columns are meant to be independent. The rows copy references to values rather than deep-copying the values.

## Related Snippets

<!-- catalog:related:start -->
- [Transpose Bounded Ragged Exact-String Rows with None for Missing Cells](transpose-bounded-ragged-exact-string-rows-with-none-for-missing-cells.md)
- [Assemble Fixed-Width Feature Slots from Bounded Named Sources](assemble-fixed-width-feature-slots-from-bounded-named-sources.md)
<!-- catalog:related:end -->
