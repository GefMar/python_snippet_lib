---
title: "Estimate Stream Frequencies with a Count-Min Sketch"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-capacity-sized-bloom-filter.md
  - ../data-processing/sample-a-stream-with-a-fixed-size-reservoir.md
---

# Estimate Stream Frequencies with a Count-Min Sketch

## Idea and Problem

Estimate per-key frequencies with a fixed-size counter table whose hash collisions can only increase an answer.

Each update increments one counter in every independently addressed row. A
query returns the smallest of those counters, which limits collision noise
while preserving the invariant that accepted non-negative updates are never
underestimated.

## When to Use

Use a Count-Min Sketch for a high-cardinality stream when exact per-key counts
would use too much memory and a bounded overestimate is acceptable. Keys need a
stable byte representation, dimensions must be selected for the expected error
budget, and all producers that merge sketches must use identical dimensions
and hash namespaces. Keep exact counters when under- and over-counting are both
unacceptable.

## Implementation

```python
from hashlib import blake2b


_HASH_DOMAIN = b"py-cms-v1"


class CountMinSketch:
    def __init__(
        self,
        *,
        width: int,
        depth: int,
        namespace: bytes = b"",
    ) -> None:
        for name, value in (("width", width), ("depth", depth)):
            if isinstance(value, bool) or not isinstance(value, int):
                raise TypeError(f"{name} must be an integer")
            if value <= 0:
                raise ValueError(f"{name} must be positive")
        if not isinstance(namespace, bytes):
            raise TypeError("namespace must be bytes")
        if len(namespace) > 64:
            raise ValueError("namespace must contain at most 64 bytes")

        self.width = width
        self.depth = depth
        self.namespace = namespace
        self._rows = [[0] * width for _ in range(depth)]

    def _column(self, key: bytes, row: int) -> int:
        if not isinstance(key, bytes):
            raise TypeError("keys must be bytes")
        digest = blake2b(
            row.to_bytes(8, byteorder="big") + key,
            digest_size=8,
            key=self.namespace,
            person=_HASH_DOMAIN,
        ).digest()
        return int.from_bytes(digest, byteorder="big") % self.width

    def add(self, key: bytes, count: int = 1) -> None:
        if isinstance(count, bool) or not isinstance(count, int):
            raise TypeError("count must be an integer")
        if count < 0:
            raise ValueError("count must be non-negative")

        for row in range(self.depth):
            column = self._column(key, row)
            self._rows[row][column] += count

    def estimate(self, key: bytes) -> int:
        return min(
            self._rows[row][self._column(key, row)]
            for row in range(self.depth)
        )

    def merge(self, other: "CountMinSketch") -> None:
        if not isinstance(other, CountMinSketch):
            raise TypeError("other must be a CountMinSketch")
        if (
            self.width,
            self.depth,
            self.namespace,
        ) != (
            other.width,
            other.depth,
            other.namespace,
        ):
            raise ValueError("sketch dimensions and namespace must match")

        for row in range(self.depth):
            for column in range(self.width):
                self._rows[row][column] += other._rows[row][column]
```

## Example

```python
first = CountMinSketch(width=128, depth=4, namespace=b"events-v1")
second = CountMinSketch(width=128, depth=4, namespace=b"events-v1")

first.add(b"alpha", 3)
first.add(b"beta", 2)
second.add(b"alpha", 2)
first.merge(second)

fully_colliding = CountMinSketch(width=1, depth=2)
fully_colliding.add(b"left", 4)
fully_colliding.add(b"right", 7)

try:
    first.merge(CountMinSketch(width=128, depth=4, namespace=b"other"))
except ValueError:
    incompatible_merge_rejected = True
else:
    incompatible_merge_rejected = False

assert (
    first.estimate(b"alpha") >= 5,
    first.estimate(b"beta") >= 2,
    first.estimate(b"missing") >= 0,
    fully_colliding.estimate(b"left"),
    fully_colliding.estimate(b"right"),
    incompatible_merge_rejected,
) == (True, True, True, 11, 11, True)
```

## Trade-offs and Limitations

Estimates may be larger than the true count, and insufficient width or an
adversarial key distribution can make that error large. This implementation
does not derive dimensions from an error target, discover heavy-hitter
candidates, delete keys, accept negative updates, compress counters, or protect
concurrent mutation. Memory is `O(width * depth)`, and callers must cap
dimensions before allocating from untrusted configuration. Treat the fixed
hash domain and namespace as compatibility data when sketches are persisted or
exchanged.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Capacity-Sized Bloom Filter](build-a-capacity-sized-bloom-filter.md)
- [Sample a Stream with a Fixed-Size Reservoir](../data-processing/sample-a-stream-with-a-fixed-size-reservoir.md)
<!-- catalog:related:end -->
