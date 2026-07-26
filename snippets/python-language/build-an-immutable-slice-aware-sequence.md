---
title: "Build an Immutable Slice-Aware Sequence"
snippet_type: recipe
use_cases:
  - data-transformation
  - interoperability
tested_python:
  - "3.14"
dependencies: []
related:
  - make-a-defensive-copy-at-a-mutable-input-boundary.md
---

# Build an Immutable Slice-Aware Sequence

## Idea and Problem

Back a custom sequence with a tuple and return the same documented sequence type for slices instead of exposing the storage tuple.

Implementing `__len__` and `__getitem__` through `collections.abc.Sequence`
provides standard iteration, containment, reverse iteration, `index`, and
`count` behavior. Distinguishing integer indexes from slices lets the object
preserve its public abstraction while retaining normal Python slice semantics.

## When to Use

Use this form for a small immutable or snapshot-like collection that needs a
domain-specific type after slicing. The constructor eagerly captures any
iterable, so later mutation of a source list cannot change the sequence. Use a
plain tuple when the custom type adds no meaningful behavior or vocabulary,
and define a different policy for mutable views.

## Implementation

```python
from collections.abc import Iterable, Sequence
from dataclasses import dataclass
from typing import TypeVar, overload


ItemT = TypeVar("ItemT")


@dataclass(frozen=True, slots=True, init=False)
class SnapshotSequence(Sequence[ItemT]):
    _items: tuple[ItemT, ...]

    def __init__(self, items: Iterable[ItemT]) -> None:
        object.__setattr__(self, "_items", tuple(items))

    def __len__(self) -> int:
        return len(self._items)

    @overload
    def __getitem__(self, index: int) -> ItemT: ...

    @overload
    def __getitem__(self, index: slice) -> "SnapshotSequence[ItemT]": ...

    def __getitem__(
        self,
        index: int | slice,
    ) -> ItemT | "SnapshotSequence[ItemT]":
        if isinstance(index, slice):
            return SnapshotSequence(self._items[index])
        return self._items[index]
```

## Example

```python
from dataclasses import FrozenInstanceError


source = [10, 20, 30, 40]
sequence = SnapshotSequence(source)
source[1] = 99

middle = sequence[1:3]
reversed_step = sequence[::-2]
empty = sequence[2:2]

try:
    setattr(sequence, "_items", (99,))
except FrozenInstanceError:
    reassignment_rejected = True
else:
    reassignment_rejected = False

assert (
    sequence[-1],
    tuple(middle),
    isinstance(middle, SnapshotSequence),
    tuple(reversed_step),
    len(empty),
    20 in sequence,
    sequence.count(20),
    reassignment_rejected,
) == (40, (20, 30), True, (40, 20), 0, True, 1, True)
```

## Trade-offs and Limitations

Construction and every slice copy references into a new tuple, so large or
frequent slices allocate memory. The frozen dataclass blocks ordinary field
assignment, but immutability remains shallow: mutable elements can still change,
and `object.__setattr__` is not a security boundary. The implementation
deliberately returns `SnapshotSequence` rather than `type(self)`, because
subclasses may require incompatible constructor arguments; subclasses that
must preserve their type need an explicit factory hook. Sequence mixins may
repeatedly call `__getitem__`, so a backing store with expensive indexed access
can make inherited operations slow.

## Related Snippets

<!-- catalog:related:start -->
- [Make a Defensive Copy at a Mutable Input Boundary](make-a-defensive-copy-at-a-mutable-input-boundary.md)
<!-- catalog:related:end -->
