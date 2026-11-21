---
title: "Stream the Last Record of Each Consecutive Key Run"
snippet_type: recipe
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - select-one-record-per-key-with-an-explicit-ranking-rule.md
  - co-group-bounded-sorted-integer-keys-with-full-outer-coverage.md
---

# Stream the Last Record of Each Consecutive Key Run

## Idea and Problem

Reduce contiguous equal-key records to their final record without keeping a dictionary of all keys.

Consume each group immediately because consecutive groups share the underlying iterator. Keep just the last value from each group. The same key may produce another result if it appears in a later, separate run.

## When to Use

Use this for ordered updates that are already grouped by key when the last input record wins. Unlike a full sort or grouping into lists, this can produce the first result as soon as its run ends.

## Implementation

```python
from collections import deque
from collections.abc import Callable, Iterable, Iterator
from itertools import groupby


def last_of_each_run[T, K](rows: Iterable[T], key: Callable[[T], K]) -> Iterator[T]:
    for _, group in groupby(rows, key):
        yield deque(group, maxlen=1)[0]
```

## Example

```python
from operator import itemgetter

rows = iter((("a", 2), ("a", 8), ("b", 7), ("a", 4), ("a", 3)))
assert (tuple(last_of_each_run(rows, itemgetter(0))), tuple(last_of_each_run([], str))) == (
    (("a", 8), ("b", 7), ("a", 3)),
    (),
)
```

## Trade-offs and Limitations

Group boundaries use consecutive key equality, not global deduplication. The iterator may read one item into the next group to detect a boundary, and it cannot yield a result from a run that never ends. Records are returned by reference.

## Related Snippets

<!-- catalog:related:start -->
- [Select One Record per Key with an Explicit Ranking Rule](select-one-record-per-key-with-an-explicit-ranking-rule.md)
- [Co-Group Bounded Sorted Integer Keys with Full-Outer Coverage](co-group-bounded-sorted-integer-keys-with-full-outer-coverage.md)
<!-- catalog:related:end -->
