---
title: "Find Exact Heavy Hitters with Verified Misra-Gries Candidates"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - estimate-stream-frequencies-with-a-count-min-sketch.md
  - find-a-strict-majority-with-boyer-moore-voting.md
  - estimate-distinct-byte-strings-with-a-mergeable-hyperloglog.md
---

# Find Exact Heavy Hitters with Verified Misra-Gries Candidates

## Idea and Problem

Find every bounded string whose exact frequency is strictly greater than a one-over-k share of a materialized sequence without retaining a full frequency table.

Misra-Gries keeps at most `k - 1` candidates by decrementing all counters when
a new key arrives and the candidate table is full. This first pass cannot give
exact counts, but it cannot discard a true heavy hitter. Replaying the stable
tuple verifies the candidates and removes false positives.

## When to Use

Use this algorithm when the input is bounded and replayable, only keys above a
strict frequency threshold matter, and auxiliary state should depend on `k`
rather than the number of distinct keys. The final entries are exact even
though the candidate counters are not frequency estimates.

Use `Counter` when a complete frequency table is affordable or arbitrary
top-ranked keys are required. Use an approximate sketch for a one-shot or
unbounded stream when exact replay is impossible and estimation error is an
accepted part of the contract.

## Implementation

```python
from dataclasses import dataclass

_MAX_ITEMS = 65_536
_MAX_K = 256
_MAX_KEY_CHARACTERS = 256
_MAX_KEY_BYTES = 1_024
_MAX_TOTAL_BYTES = 4 * 1024 * 1024


@dataclass(frozen=True, slots=True)
class HeavyHitter:
    key: str
    count: int


def find_exact_heavy_hitters(
    values: tuple[str, ...],
    *,
    k: int,
) -> tuple[HeavyHitter, ...]:
    """Return exactly the keys whose counts satisfy count * k > len(values)."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if not 1 <= len(values) <= _MAX_ITEMS:
        raise ValueError("item count is outside the supported range")
    if type(k) is not int:
        raise TypeError("k must be an exact integer")
    if not 2 <= k <= _MAX_K:
        raise ValueError("k is outside the supported range")

    total_bytes = 0
    for index, key in enumerate(values):
        if type(key) is not str:
            raise TypeError(f"values[{index}] must be an exact string")
        if not key:
            raise ValueError(f"values[{index}] must not be empty")
        if len(key) > _MAX_KEY_CHARACTERS:
            raise ValueError(f"values[{index}] exceeds the character limit")
        try:
            encoded = key.encode("utf-8")
        except UnicodeEncodeError:
            raise ValueError(f"values[{index}] must be valid UTF-8 text") from None
        if len(encoded) > _MAX_KEY_BYTES:
            raise ValueError(f"values[{index}] exceeds the UTF-8 byte limit")
        if len(encoded) > _MAX_TOTAL_BYTES - total_bytes:
            raise ValueError("values exceed the aggregate UTF-8 byte limit")
        total_bytes += len(encoded)

    counters: dict[str, int] = {}
    for key in values:
        if key in counters:
            counters[key] += 1
        elif len(counters) < k - 1:
            counters[key] = 1
        else:
            expired: list[str] = []
            for candidate in counters:
                counters[candidate] -= 1
                if counters[candidate] == 0:
                    expired.append(candidate)
            for candidate in expired:
                del counters[candidate]

    exact_counts = dict.fromkeys(counters, 0)
    for key in values:
        if key in exact_counts:
            exact_counts[key] += 1

    hitters = [
        HeavyHitter(key, count) for key, count in exact_counts.items() if count * k > len(values)
    ]
    hitters.sort(key=lambda item: (-item.count, item.key))
    return tuple(hitters)
```

## Example

```python
observations = (
    "red",
    "blue",
    "red",
    "green",
    "red",
    "blue",
    "yellow",
    "blue",
    "red",
    "green",
    "blue",
    "green",
)

hitters = find_exact_heavy_hitters(observations, k=4)
reordered = find_exact_heavy_hitters(tuple(reversed(observations)), k=4)

assert (
    hitters
    == reordered
    == (
        HeavyHitter("blue", 4),
        HeavyHitter("red", 4),
    )
)
```

## Trade-offs and Limitations

Let `B` be the aggregate UTF-8 input size and `n` the item count. With ordinary
expected dictionary hashing, validation, candidate selection, and verification
take expected `O(B + n)` work. A full-table decrement costs `O(k)`, but it also
cancels `k` represented stream items, so all such decrements cost `O(n)` in
aggregate. Sorting at most `k - 1` verified entries adds `O(k log k)` work.
Candidate dictionaries and a temporary expiration list use `O(k)` auxiliary
state, excluding the input and immutable output references.

The tuple must remain stable for both passes. This function does not return a
generic top-k ranking, a full frequency table, or estimates for non-candidates.
It excludes non-replayable and unbounded streams, weighted updates, sliding
windows, distributed summary merging, and guarantees under adversarial hash
behavior. A key occurring exactly `n / k` times is deliberately excluded.

## Related Snippets

<!-- catalog:related:start -->
- [Estimate Stream Frequencies with a Count-Min Sketch](estimate-stream-frequencies-with-a-count-min-sketch.md)
- [Find a Strict Majority with Boyer-Moore Voting](find-a-strict-majority-with-boyer-moore-voting.md)
- [Estimate Distinct Byte Strings with a Mergeable HyperLogLog](estimate-distinct-byte-strings-with-a-mergeable-hyperloglog.md)
<!-- catalog:related:end -->
