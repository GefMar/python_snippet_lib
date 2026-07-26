---
title: "Build a Capacity-Sized Bloom Filter"
snippet_type: algorithm
use_cases:
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related: []
---

# Build a Capacity-Sized Bloom Filter

## Idea and Problem

Represent approximate membership in a compact bit array sized from an expected item count and acceptable false-positive rate.

A Bloom filter hashes every inserted byte string to several bit positions.
Queries can prove that a value is absent when any corresponding bit is clear;
when all bits are set, the value is only possibly present. The standard sizing
formulas choose a bit count and a near-optimal number of hashes for the stated
capacity and target rate.

## When to Use

Use this data structure as a memory-efficient prefilter before a more expensive
authoritative lookup when false positives cause only extra work. Inputs must
have stable byte encodings, expected capacity must be credible, and inserted
values must never need deletion. Keep a normal set when exact membership or
enumeration is required.

## Implementation

```python
import math
from hashlib import blake2b


_HASH_DOMAIN = b"snippet-bloom-v1"


class BloomFilter:
    def __init__(self, *, bit_count: int, hash_count: int) -> None:
        if isinstance(bit_count, bool) or not isinstance(bit_count, int):
            raise TypeError("bit_count must be an integer")
        if isinstance(hash_count, bool) or not isinstance(hash_count, int):
            raise TypeError("hash_count must be an integer")
        if bit_count <= 0 or hash_count <= 0:
            raise ValueError("bit_count and hash_count must be positive")

        self.bit_count = bit_count
        self.hash_count = hash_count
        self._bits = bytearray((bit_count + 7) // 8)

    @classmethod
    def for_capacity(
        cls,
        *,
        expected_items: int,
        false_positive_rate: float,
    ) -> "BloomFilter":
        if isinstance(expected_items, bool) or not isinstance(expected_items, int):
            raise TypeError("expected_items must be an integer")
        if expected_items <= 0:
            raise ValueError("expected_items must be positive")
        if isinstance(false_positive_rate, bool) or not isinstance(
            false_positive_rate,
            (int, float),
        ):
            raise TypeError("false_positive_rate must be numeric")

        rate = float(false_positive_rate)
        if not math.isfinite(rate) or not 0.0 < rate < 1.0:
            raise ValueError("false_positive_rate must be finite and between 0 and 1")

        bit_count = math.ceil(
            -(expected_items * math.log(rate)) / (math.log(2.0) ** 2)
        )
        hash_count = max(1, round((bit_count / expected_items) * math.log(2.0)))
        return cls(bit_count=bit_count, hash_count=hash_count)

    def _positions(self, value: bytes) -> tuple[int, ...]:
        if not isinstance(value, bytes):
            raise TypeError("BloomFilter values must be bytes")

        positions = []
        for index in range(self.hash_count):
            digest = blake2b(
                index.to_bytes(8, byteorder="big") + value,
                digest_size=8,
                person=_HASH_DOMAIN,
            ).digest()
            positions.append(int.from_bytes(digest, byteorder="big") % self.bit_count)
        return tuple(positions)

    def add(self, value: bytes) -> None:
        for position in self._positions(value):
            self._bits[position // 8] |= 1 << (position % 8)

    def __contains__(self, value: object) -> bool:
        if not isinstance(value, bytes):
            raise TypeError("BloomFilter values must be bytes")
        return all(
            self._bits[position // 8] & (1 << (position % 8))
            for position in self._positions(value)
        )
```

## Example

```python
seen = BloomFilter.for_capacity(
    expected_items=50,
    false_positive_rate=0.01,
)
inserted = (b"alpha", b"beta", b"gamma")
for value in inserted:
    seen.add(value)

try:
    BloomFilter.for_capacity(expected_items=0, false_positive_rate=0.01)
except ValueError:
    zero_capacity_rejected = True
else:
    zero_capacity_rejected = False

try:
    b"text" in BloomFilter(bit_count=32, hash_count=0)
except ValueError:
    zero_hashes_rejected = True
else:
    zero_hashes_rejected = False

assert (
    seen.bit_count,
    seen.hash_count,
    all(value in seen for value in inserted),
    b"definitely-absent" in seen,
    zero_capacity_rejected,
    zero_hashes_rejected,
) == (480, 7, True, False, True, True)
```

## Trade-offs and Limitations

False positives are expected, and exceeding the planned capacity increases
their rate; the requested rate is a sizing target, not a per-query guarantee.
This implementation performs one digest per hash position and favors a clear,
stable contract over peak throughput. It has no deletion, count, persistence,
merge, or concurrency policy. The unkeyed hashes do not make the filter safe
for authorization decisions or adversarial denial-of-service boundaries.

## Related Snippets

<!-- catalog:related:start -->
No related snippets.
<!-- catalog:related:end -->
