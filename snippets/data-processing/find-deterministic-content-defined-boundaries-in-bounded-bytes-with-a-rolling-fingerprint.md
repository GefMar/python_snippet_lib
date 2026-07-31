---
title: "Find Deterministic Content-Defined Boundaries in Bounded Bytes with a Rolling Fingerprint"
snippet_type: algorithm
use_cases:
  - caching
  - data-transformation
  - performance-optimization
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - ../storage-databases/split-a-binary-stream-into-exclusively-created-numbered-parts.md
  - ../storage-databases/validate-a-bounded-chunk-manifest-before-a-conditional-version-switch.md
  - ../testing-tooling/enumerate-every-contiguous-chunking-of-bounded-bytes-for-stream-tests.md
---

# Find Deterministic Content-Defined Boundaries in Bounded Bytes with a Rolling Fingerprint

## Idea and Problem

Divide bounded bytes at repeatable content-selected positions so a local insertion does not necessarily shift every later boundary.

The fixed `polynomial64-v1` profile hashes the latest 64 bytes with a rolling
base-257 polynomial modulo `2**64`. After a 2,048-byte minimum, a low-bit
pattern selects a natural boundary with probability near one in 8,192 for
ordinary varied data. A 32,768-byte maximum forces progress.

Resetting the window after every cut makes each span independently reproducible
under this exact profile. Returning half-open offsets avoids copying the input
and exposes whether each boundary came from the fingerprint, the size cap, or
the final suffix.

## When to Use

Use this profile for small local experiments, deterministic cache partitioning,
or reference tests where both peers explicitly agree on `polynomial64-v1`.
Content-defined spans can retain some later boundaries after nearby insertions,
unlike fixed-size blocks.

Use a standardized or production chunker when compatibility, streaming state,
measured distribution quality, adversarial inputs, or stable long-term storage
formats matter. Use a cryptographic digest after chunking when identity or
integrity must be established.

## Implementation

```python
from collections import deque
from dataclasses import dataclass
from itertools import pairwise
from random import Random

_POLYNOMIAL64_MASK = 2**64 - 1
_POLYNOMIAL64_BASE = 257
_POLYNOMIAL64_WINDOW = 64
_POLYNOMIAL64_MIN_SIZE = 2_048
_POLYNOMIAL64_TARGET_SIZE = 8_192
_POLYNOMIAL64_MAX_SIZE = 32_768
_MAX_CONTENT_DEFINED_BYTES = 1_048_576
_OUTGOING_FACTOR = pow(
    _POLYNOMIAL64_BASE,
    _POLYNOMIAL64_WINDOW - 1,
    2**64,
)


@dataclass(frozen=True, slots=True)
class ContentDefinedSpan:
    start: int
    stop: int
    reason: str


def polynomial64_v1_spans(data: bytes) -> tuple[ContentDefinedSpan, ...]:
    """Return spans under the fixed reset-per-chunk polynomial64-v1 profile."""
    if type(data) is not bytes:
        raise TypeError("data must be exact bytes")
    if len(data) > _MAX_CONTENT_DEFINED_BYTES:
        raise ValueError("data exceeds 1048576 bytes")
    if not data:
        return ()

    spans: list[ContentDefinedSpan] = []
    chunk_start = 0
    fingerprint = 0
    window: deque[int] = deque()
    for index, byte in enumerate(data):
        value = byte + 1
        if len(window) == _POLYNOMIAL64_WINDOW:
            outgoing = window.popleft()
            fingerprint = (
                (fingerprint - outgoing * _OUTGOING_FACTOR) * _POLYNOMIAL64_BASE + value
            ) & _POLYNOMIAL64_MASK
        else:
            fingerprint = (fingerprint * _POLYNOMIAL64_BASE + value) & _POLYNOMIAL64_MASK
        window.append(value)

        chunk_size = index + 1 - chunk_start
        reason: str | None = None
        if chunk_size == _POLYNOMIAL64_MAX_SIZE:
            reason = "maximum"
        elif (
            chunk_size >= _POLYNOMIAL64_MIN_SIZE
            and fingerprint & (_POLYNOMIAL64_TARGET_SIZE - 1) == 0
        ):
            reason = "fingerprint"
        if reason is not None:
            spans.append(ContentDefinedSpan(chunk_start, index + 1, reason))
            chunk_start = index + 1
            fingerprint = 0
            window.clear()

    if chunk_start < len(data):
        spans.append(ContentDefinedSpan(chunk_start, len(data), "final"))
    return tuple(spans)
```

## Example

```python
def recomputed_polynomial64_v1_spans(
    data: bytes,
) -> tuple[ContentDefinedSpan, ...]:
    spans: list[ContentDefinedSpan] = []
    chunk_start = 0
    for index in range(len(data)):
        chunk_size = index + 1 - chunk_start
        reason: str | None = None
        if chunk_size == _POLYNOMIAL64_MAX_SIZE:
            reason = "maximum"
        elif chunk_size >= _POLYNOMIAL64_MIN_SIZE:
            window_start = index + 1 - _POLYNOMIAL64_WINDOW
            fingerprint = 0
            for byte in data[window_start : index + 1]:
                fingerprint = (fingerprint * _POLYNOMIAL64_BASE + byte + 1) & _POLYNOMIAL64_MASK
            if fingerprint & (_POLYNOMIAL64_TARGET_SIZE - 1) == 0:
                reason = "fingerprint"
        if reason is not None:
            spans.append(ContentDefinedSpan(chunk_start, index + 1, reason))
            chunk_start = index + 1
    if chunk_start < len(data):
        spans.append(ContentDefinedSpan(chunk_start, len(data), "final"))
    return tuple(spans)


def assert_span_invariants(data: bytes, spans: tuple[ContentDefinedSpan, ...]) -> None:
    assert tuple(data[span.start : span.stop] for span in spans)
    assert b"".join(data[span.start : span.stop] for span in spans) == data
    assert all(span.start < span.stop for span in spans)
    assert all(left.stop == right.start for left, right in pairwise(spans))
    assert all(span.stop - span.start <= _POLYNOMIAL64_MAX_SIZE for span in spans)
    assert all(span.stop - span.start >= _POLYNOMIAL64_MIN_SIZE for span in spans[:-1])


assert polynomial64_v1_spans(b"") == ()
forced = polynomial64_v1_spans(b"\x00" * 70_000)
assert forced == recomputed_polynomial64_v1_spans(b"\x00" * 70_000)
assert any(span.reason == "maximum" for span in forced)
assert_span_invariants(b"\x00" * 70_000, forced)

rng = Random(0xC0_17)
checked = 0
for _ in range(50):
    data = rng.randbytes(rng.randrange(1, 6_001))
    actual = polynomial64_v1_spans(data)
    assert actual == recomputed_polynomial64_v1_spans(data)
    assert_span_invariants(data, actual)
    assert polynomial64_v1_spans(data) == actual
    checked += 1

assert checked == 50
```

## Trade-offs and Limitations

The rolling implementation takes `O(N)` time and retains the 64-value window
plus `O(chunks)` result spans. The Example's independent oracle deliberately
recomputes each eligible window in `O(N × window)` time. Python integer masking
makes the modulo-`2**64` behavior explicit and repeatable.

`polynomial64-v1` is intentionally non-parameterized: changing its constants,
cut precedence, update timing, or reset policy defines a different profile.
The rolling value is not cryptographic and provides no identity, integrity, or
adversarial collision guarantee. The low-bit condition suggests rather than
guarantees an average size, while the maximum can dominate low-entropy data.
No streaming API or compatibility with another chunker is promised.

## Related Snippets

<!-- catalog:related:start -->
- [Split a Binary Stream into Exclusively Created Numbered Parts](../storage-databases/split-a-binary-stream-into-exclusively-created-numbered-parts.md)
- [Validate a Bounded Chunk Manifest Before a Conditional Version Switch](../storage-databases/validate-a-bounded-chunk-manifest-before-a-conditional-version-switch.md)
- [Enumerate Every Contiguous Chunking of Bounded Bytes for Stream Tests](../testing-tooling/enumerate-every-contiguous-chunking-of-bounded-bytes-for-stream-tests.md)
<!-- catalog:related:end -->
