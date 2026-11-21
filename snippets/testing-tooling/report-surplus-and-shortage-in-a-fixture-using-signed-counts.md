---
title: "Report Surplus and Shortage in a Fixture Using Signed Counts"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - report-missing-and-unknown-keys-from-a-typeddict-declaration.md
  - ../storage-databases/compare-bounded-apparent-sizes-of-two-file-tree-snapshots.md
---

# Report Surplus and Shortage in a Fixture Using Signed Counts

## Idea and Problem

Report extra and missing occurrences separately when the same item can appear more than once.

Subtract expected occurrences from observed ones with `Counter.subtract()`.
It preserves both signs: positive differences are surplus, negative ones
are shortage. Unary `+` keeps the surplus counts and unary `-` makes the
shortage counts positive. Neither report includes zero differences.

## When to Use

Use this in a test for unordered events, labels, or records that you can
reduce to string identifiers. It helps when comparing sets would hide
count differences, but comparing lists would fail on harmless reordering.

## Implementation

```python
from collections import Counter
from collections.abc import Iterable


def count_differences(
    observed: Iterable[str], expected: Iterable[str]
) -> tuple[Counter[str], Counter[str]]:
    delta = Counter(iter(observed))
    delta.subtract(iter(expected))
    return +delta, -delta
```

## Example

```python
observed = ["ready", "render", "render", "render", "store"]
expected = ["render", "store", "store", "store", "ready"]

assert set(observed) == set(expected)
surplus, shortage = count_differences(observed, expected)
assert dict(surplus) == {"render": 2}
assert dict(shortage) == {"store": 2}
assert not any(count_differences(reversed(expected), expected))
```

## Trade-offs and Limitations

The helper consumes each input once and retains counts for distinct
identifiers. It ignores positions and order. Use separate iterables for
the two inputs, not the same one-shot iterator. Iteration counts each key
once if given a mapping; this API does not read counts from mapping values.

`Counter(observed) - Counter(expected)` alone would drop the negative
differences and hide shortages. Use signed subtraction before splitting
the report.

## Related Snippets

<!-- catalog:related:start -->
- [Report Missing and Unknown Keys from a TypedDict Declaration](report-missing-and-unknown-keys-from-a-typeddict-declaration.md)
- [Compare Bounded Apparent Sizes of Two File-Tree Snapshots](../storage-databases/compare-bounded-apparent-sizes-of-two-file-tree-snapshots.md)
<!-- catalog:related:end -->
