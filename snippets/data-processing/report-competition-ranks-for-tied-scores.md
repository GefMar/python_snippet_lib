---
title: "Report Competition Ranks for Tied Scores"
snippet_type: recipe
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/rank-bounded-records-with-stable-ties-and-neighbor-windows.md
  - select-one-record-per-key-with-an-explicit-ranking-rule.md
---

# Report Competition Ranks for Tied Scores

## Idea and Problem

Give equal scores a shared place while leaving a gap before the next lower score.

Sort once in descending score order and keep the sort stable. Each score group gets the position of its first item. The next group starts after every item in the previous group, so two first-place entries are followed by third place.

## When to Use

Use this for small leaderboards where ties should occupy the same rank and still count toward the position of the next group. Records with equal scores stay in input order.

## Implementation

```python
from collections.abc import Iterable
from operator import itemgetter


def competition_ranks(rows: Iterable[tuple[str, int]]) -> tuple[tuple[str, int], ...]:
    ordered = sorted(rows, key=itemgetter(1), reverse=True)
    result = []
    previous_score = None
    rank = 0

    for position, (label, score) in enumerate(ordered, start=1):
        if score != previous_score:
            rank = position
            previous_score = score
        result.append((label, rank))
    return tuple(result)
```

## Example

```python
ranked = competition_ranks([("small", -1), ("top b", 8), ("low", -1), ("top a", 8), ("mid", 3)])
assert (ranked, competition_ranks([])) == (
    (("top b", 1), ("top a", 1), ("mid", 3), ("small", 4), ("low", 4)),
    (),
)
```

## Trade-offs and Limitations

This is competition ranking, not dense ranking or tie-breaking into unique positions. Sorting materializes the input and takes O(n log n) comparisons. Labels are not used to break ties and need not be unique; give each row an independent identifier if repeated labels are ambiguous downstream.

## Related Snippets

<!-- catalog:related:start -->
- [Rank Bounded Records with Stable Ties and Neighbor Windows](../algorithms-data-structures/rank-bounded-records-with-stable-ties-and-neighbor-windows.md)
- [Select One Record per Key with an Explicit Ranking Rule](select-one-record-per-key-with-an-explicit-ranking-rule.md)
<!-- catalog:related:end -->
