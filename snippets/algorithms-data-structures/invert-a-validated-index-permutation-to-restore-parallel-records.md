---
title: "Invert a Validated Index Permutation to Restore Parallel Records"
snippet_type: recipe
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - rank-and-unrank-index-permutations-in-itertools-permutations-order.md
  - rank-bounded-records-with-stable-ties-and-neighbor-windows.md
---

# Invert a Validated Index Permutation to Restore Parallel Records

## Idea and Problem

Recover the original position of every moved record so several aligned columns can return to their original order.

Let `order[new]` give the old index of the record now at position `new`.
Store `new` at `inverse[old]` to reverse that relationship. Reordering with
`moved[new] = original[order[new]]` can then be undone with
`original[old] = moved[inverse[old]]`.

## When to Use

Use this when a sort or external operation changed record order and retained
the exact zero-based index permutation. Each index from zero through
`len(order) - 1` must occur once. The same permutation can restore any
parallel column that moved in lockstep and has the corresponding length.

## Implementation

```python
from collections.abc import Iterable


def invert_index_permutation(order: Iterable[int]) -> tuple[int, ...]:
    """Validate new-to-old indices and return the old-to-new inverse."""
    indices = tuple(order)
    inverse = [-1] * len(indices)
    for new, old in enumerate(indices):
        if type(old) is not int:
            raise TypeError("permutation indices must be integers")
        if not 0 <= old < len(indices):
            raise ValueError("permutation index lies outside the input length")
        if inverse[old] != -1:
            raise ValueError("permutation indices must be unique")
        inverse[old] = new
    return tuple(inverse)
```

## Example

```python
names = ("oak", "pine", "ash")
counts = (12, 4, 9)
order = tuple(sorted(range(len(names)), key=names.__getitem__))
moved_names = tuple(names[index] for index in order)
moved_counts = tuple(counts[index] for index in order)
inverse = invert_index_permutation(iter(order))

assert (
    order,
    inverse,
    tuple(moved_names[index] for index in inverse),
    tuple(moved_counts[index] for index in inverse),
) == ((2, 0, 1), (1, 2, 0), names, counts)
```

## Trade-offs and Limitations

The function materializes the input indices and takes linear time and space.
Length, range and uniqueness together guarantee that every inverse position
is set. The empty permutation has an empty inverse.

The caller must preserve the permutation alongside the moved records. Value
equality alone may be ambiguous when records have duplicates. Index inversion
requires a complete reordering, so filtering or duplicating records calls for
a different mapping.

## Related Snippets

<!-- catalog:related:start -->
- [Rank and Unrank Index Permutations in itertools.permutations Order](rank-and-unrank-index-permutations-in-itertools-permutations-order.md)
- [Rank Bounded Records with Stable Ties and Neighbor Windows](rank-bounded-records-with-stable-ties-and-neighbor-windows.md)
<!-- catalog:related:end -->
