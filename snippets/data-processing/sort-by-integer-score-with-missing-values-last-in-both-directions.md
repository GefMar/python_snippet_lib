---
title: "Sort by Integer Score with Missing Values Last in Both Directions"
snippet_type: recipe
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/rank-bounded-records-with-stable-ties-and-neighbor-windows.md
  - ../algorithms-data-structures/sort-text-by-literal-text-runs-and-ascii-digit-runs.md
---

# Sort by Integer Score with Missing Values Last in Both Directions

## Idea and Problem

Keep missing scores at the end whether numeric scores are sorted up or down.

Use a key with a missing flag first, then a numeric component. Reverse only the numeric component for descending order. Reversing the entire sort would move the missing group to the front.

## When to Use

Use this for small lists of records with integer scores or None. Score zero remains a numeric value, and records with equal numeric scores or missing scores retain input order.

## Implementation

```python
from collections.abc import Iterable


def sort_scores(
    rows: Iterable[tuple[str, int | None]], *, descending: bool = False
) -> list[tuple[str, int | None]]:
    def sort_key(row: tuple[str, int | None]) -> tuple[bool, int]:
        score = row[1]
        if score is None:
            return True, 0
        return False, -score if descending else score

    return sorted(rows, key=sort_key)
```

## Example

```python
rows = [
    ("missing a", None),
    ("first zero", 0),
    ("below", -2),
    ("second zero", 0),
    ("missing b", None),
]
ascending = sort_scores(rows)
descending = sort_scores(rows, descending=True)
assert ([name for name, _ in ascending], [name for name, _ in descending]) == (
    ["below", "first zero", "second zero", "missing a", "missing b"],
    ["first zero", "second zero", "below", "missing a", "missing b"],
)
```

## Trade-offs and Limitations

Negating the score works for integers, not for arbitrary comparable keys. This does not assign ranks or decide other missing value conventions such as NaN. Sorting materializes the rows and returns references to the original records.

## Related Snippets

<!-- catalog:related:start -->
- [Rank Bounded Records with Stable Ties and Neighbor Windows](../algorithms-data-structures/rank-bounded-records-with-stable-ties-and-neighbor-windows.md)
- [Sort Text by Literal Text Runs and ASCII Digit Runs](../algorithms-data-structures/sort-text-by-literal-text-runs-and-ascii-digit-runs.md)
<!-- catalog:related:end -->
