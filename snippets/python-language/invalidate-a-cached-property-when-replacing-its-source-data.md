---
title: "Invalidate a Cached Property When Replacing Its Source Data"
snippet_type: recipe
use_cases:
  - caching
  - performance-optimization
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - make-a-defensive-copy-at-a-mutable-input-boundary.md
  - cache-one-zero-argument-method-result-per-weakly-referenced-instance.md
---

# Invalidate a Cached Property When Replacing Its Source Data

## Idea and Problem

Clear a cached property when its source data changes so the next read recomputes it.

`cached_property` stores its result on the instance. Keep data replacement
behind one method and remove the cached value from that instance after
installing the new data. The method does not recompute the property.

## When to Use

Use this when an object has an expensive derived value, repeated reads are
common, and the source changes through a small, explicit update API. Use
an ordinary property if computing the value is cheap.

## Implementation

```python
from collections.abc import Iterable
from functools import cached_property


class Samples:
    def __init__(self, values: Iterable[int]) -> None:
        self._values = tuple(values)

    def replace(self, values: Iterable[int]) -> None:
        self._values = tuple(values)
        self.__dict__.pop("summary", None)

    @cached_property
    def summary(self) -> tuple[int, int]:
        return len(self._values), sum(self._values)
```

## Example

```python
initial = [2, 4]
left = Samples(initial)
right = Samples([])
right.replace([3, 5, 7])

initial.append(99)
assert left.summary == (2, 6)
right_summary = right.summary
assert right_summary == (3, 15)

replacement = [7, 9, 11]
left.replace(replacement)
replacement.clear()
assert "summary" not in vars(left)
assert right.summary is right_summary
updated = left.summary
assert updated == (3, 27)
assert left.summary is updated
```

## Trade-offs and Limitations

The tuple keeps later changes to the caller's input sequence from affecting
the stored values. This is sufficient for integers, not for nested mutable
objects.
Changes that bypass `replace()` also bypass invalidation.

`cached_property` needs a writable instance dictionary. This example is
for sequential access: concurrent reads and updates can race, including
repopulating the cache with an old result. Coordinate both reads and
updates if instances are shared.

## Related Snippets

<!-- catalog:related:start -->
- [Make a Defensive Copy at a Mutable Input Boundary](make-a-defensive-copy-at-a-mutable-input-boundary.md)
- [Cache One Zero-Argument Method Result per Weakly Referenced Instance](cache-one-zero-argument-method-result-per-weakly-referenced-instance.md)
<!-- catalog:related:end -->
