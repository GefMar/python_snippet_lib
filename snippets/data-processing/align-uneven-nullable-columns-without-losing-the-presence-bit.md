---
title: "Align Uneven Nullable Columns Without Losing the Presence Bit"
snippet_type: recipe
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - transpose-bounded-ragged-exact-string-rows-with-none-for-missing-cells.md
  - build-named-rows-from-columns-with-strict-zip.md
---

# Align Uneven Nullable Columns Without Losing the Presence Bit

## Idea and Problem

Compare uneven columns while distinguishing a missing row from a present null value.

Zip using a fresh private object for padding. Translate that sentinel into an explicit presence flag before returning values, so a real None is still distinguishable.

## When to Use

Use this for positional checks across two finite or incrementally consumed nullable columns. The two flags describe presence, not whether the supplied values meet a domain rule.

## Implementation

```python
from collections.abc import Iterable, Iterator
from itertools import zip_longest


def paired_cells(
    left: Iterable[str | None], right: Iterable[str | None]
) -> Iterator[tuple[bool, str | None, bool, str | None]]:
    missing = object()
    for left_cell, right_cell in zip_longest(left, right, fillvalue=missing):
        left_present = left_cell is not missing
        right_present = right_cell is not missing
        yield (
            left_present,
            left_cell if left_present else None,
            right_present,
            right_cell if right_present else None,
        )
```

## Example

```python
sample = tuple(paired_cells(iter((None, "", "left")), iter(("right", None))))
assert (sample, tuple(paired_cells((), ()))) == (
    ((True, None, True, "right"), (True, "", True, None), (True, "left", False, None)),
    (),
)
```

## Trade-offs and Limitations

Rows are aligned by position rather than key and extra rows are exposed, not rejected. Consumption continues until both inputs finish, so one unbounded input makes the result unbounded. If unequal lengths are invalid, strict zip is the clearer policy.

## Related Snippets

<!-- catalog:related:start -->
- [Transpose Bounded Ragged Exact-String Rows with None for Missing Cells](transpose-bounded-ragged-exact-string-rows-with-none-for-missing-cells.md)
- [Build Named Rows from Columns with Strict Zip](build-named-rows-from-columns-with-strict-zip.md)
<!-- catalog:related:end -->
