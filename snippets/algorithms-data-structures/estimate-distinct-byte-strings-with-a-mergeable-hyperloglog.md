---
title: "Estimate Distinct Byte Strings with a Mergeable HyperLogLog"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-capacity-sized-bloom-filter.md
  - estimate-stream-frequencies-with-a-count-min-sketch.md
  - map-keys-with-an-immutable-consistent-hash-ring.md
  - ../data-processing/sample-a-stream-with-a-fixed-size-reservoir.md
---

# Estimate Distinct Byte Strings with a Mergeable HyperLogLog

## Idea and Problem

Estimate the number of distinct byte strings with a fixed register array that can be merged across independently processed streams.

A versioned 64-bit digest selects one register and a leading-zero rank. Each
register keeps only its maximum rank; the harmonic-mean estimator combines
them, with linear counting correcting the small range when registers remain
empty.

## When to Use

Use this algorithm when exact distinct values would consume too much memory
and a reproducible approximate count is acceptable. All producers must use the
same precision and exact byte serialization. Pick a precision from 4 through
16 based on the memory/error trade-off; the approximate standard error of the
basic estimator is about `1.04 / sqrt(register_count)` for ordinary
non-adversarial streams.

## Implementation

```python
import hashlib
import math


_HASH_VERSION = b"hll-byte-values-v1\x00"


class HyperLogLog:
    __slots__ = ("_precision", "_register_count", "_registers")

    def __init__(self, *, precision: int = 10) -> None:
        if isinstance(precision, bool) or not isinstance(precision, int):
            raise TypeError("precision must be an integer")
        if not 4 <= precision <= 16:
            raise ValueError("precision must be between 4 and 16")
        self._precision = precision
        self._register_count = 1 << precision
        self._registers = [0] * self._register_count

    @property
    def precision(self) -> int:
        return self._precision

    @property
    def register_count(self) -> int:
        return self._register_count

    @property
    def registers(self) -> tuple[int, ...]:
        return tuple(self._registers)

    def _hash64(self, payload: bytes) -> int:
        digest = hashlib.blake2b(digest_size=8)
        digest.update(_HASH_VERSION)
        digest.update(len(payload).to_bytes(8, "big"))
        digest.update(payload)
        return int.from_bytes(digest.digest(), "big")

    def add(self, payload: bytes) -> None:
        if not isinstance(payload, bytes):
            raise TypeError("payload must be bytes")
        value = self._hash64(payload)
        register = value & (self._register_count - 1)
        remainder = value >> self._precision
        remaining_bits = 64 - self._precision
        rank = (
            remaining_bits - remainder.bit_length() + 1
            if remainder
            else remaining_bits + 1
        )
        self._registers[register] = max(self._registers[register], rank)

    def merge(self, other: "HyperLogLog") -> None:
        if not isinstance(other, HyperLogLog):
            raise TypeError("other must be a HyperLogLog")
        if self._precision != other._precision:
            raise ValueError("precision must match")
        self._registers = [
            max(left, right)
            for left, right in zip(self._registers, other._registers, strict=True)
        ]

    def estimate(self) -> float:
        count = self._register_count
        if count == 16:
            alpha = 0.673
        elif count == 32:
            alpha = 0.697
        elif count == 64:
            alpha = 0.709
        else:
            alpha = 0.7213 / (1 + 1.079 / count)

        denominator = sum(2.0**-register for register in self._registers)
        raw_estimate = alpha * count * count / denominator
        empty_registers = self._registers.count(0)
        if raw_estimate <= 2.5 * count and empty_registers:
            return count * math.log(count / empty_registers)
        return raw_estimate
```

## Example

```python
whole = HyperLogLog(precision=10)
left = HyperLogLog(precision=10)
right = HyperLogLog(precision=10)

for number in range(10_000):
    payload = f"value-{number}".encode()
    whole.add(payload)
    whole.add(payload)
    (left if number % 2 == 0 else right).add(payload)

left.merge(right)
merged_once = left.registers
left.merge(right)

try:
    whole.precision = 8
except AttributeError:
    precision_is_read_only = True
else:
    precision_is_read_only = False

assert (
    9_000 <= whole.estimate() <= 11_000
    and left.registers == whole.registers
    and left.registers == merged_once
    and precision_is_read_only
    and HyperLogLog(precision=10).estimate() == 0.0
)
```

## Trade-offs and Limitations

This is the original-style estimator with small-range linear counting, not
HLL++. It omits empirical bias correction, sparse representation, large-range
correction, serialization, and interoperability with external HLL formats.
The public structural parameters are read-only, but `add()` and `merge()` still
mutate register state. The 64-bit hash also limits the meaningful upper range,
while Python integer registers use much more memory than packed production
representations.
Changing the version prefix, digest, byte framing, or precision changes the
sketch contract. Approximation errors are expected, and attacker-chosen inputs
require a keyed or otherwise threat-modelled hashing strategy.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Capacity-Sized Bloom Filter](build-a-capacity-sized-bloom-filter.md)
- [Estimate Stream Frequencies with a Count-Min Sketch](estimate-stream-frequencies-with-a-count-min-sketch.md)
- [Map Keys with an Immutable Consistent Hash Ring](map-keys-with-an-immutable-consistent-hash-ring.md)
- [Sample a Stream with a Fixed-Size Reservoir](../data-processing/sample-a-stream-with-a-fixed-size-reservoir.md)
<!-- catalog:related:end -->
