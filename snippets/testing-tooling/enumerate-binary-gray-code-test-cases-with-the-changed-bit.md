---
title: "Enumerate Binary Gray-Code Test Cases with the Changed Bit"
snippet_type: testing-technique
use_cases:
  - data-transformation
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - audit-bounded-boolean-test-rows-for-unique-cause-mc-dc-coverage.md
  - enumerate-every-contiguous-chunking-of-bounded-bytes-for-stream-tests.md
---

# Enumerate Binary Gray-Code Test Cases with the Changed Bit

## Idea and Problem

Enumerate every assignment of a small set of switches while changing one switch between consecutive test cases.

A reflected binary Gray code encodes the counter `n` as `n ^ (n >> 1)`.
XOR consecutive masks to isolate the bit that changed. Its index lets a
test fixture update just that switch while using the mask as the complete
assignment.

## When to Use

Use this for a fixture with independent binary inputs and a supported update
operation for one input at a time. Bit indexes start at zero for the least
significant bit. The first case has all switches off and reports `None` for
the changed bit. Width zero yields one empty assignment.

## Implementation

```python
from collections.abc import Iterator


_MAX_WIDTH = 20


def gray_cases(width: int) -> Iterator[tuple[int, int | None]]:
    """Yield (mask, changed_bit) in reflected binary Gray order."""
    if type(width) is not int:
        raise TypeError("width must be an integer")
    if not 0 <= width <= _MAX_WIDTH:
        raise ValueError(f"width must be between 0 and {_MAX_WIDTH}")

    previous = 0
    yield previous, None
    for counter in range(1, 1 << width):
        mask = counter ^ (counter >> 1)
        changed = (previous ^ mask).bit_length() - 1
        yield mask, changed
        previous = mask
```

## Example

```python
assert (tuple(gray_cases(3)), tuple(gray_cases(0))) == (
    ((0, None), (1, 0), (3, 1), (2, 0), (6, 2), (7, 0), (5, 1), (4, 0)),
    ((0, None),),
)
```

## Trade-offs and Limitations

The iterator keeps only the current counter and masks. It generates `2**width`
cases, so even the width cap of 20 can produce more than a million cases.
Choose width based on the cost of each test and consume cases lazily.

This order changes one bit per transition in the emitted sequence. It leaves
fixture reset and independence to the test. Applying the changed-bit hint
works only when the fixture still holds the previous assignment.

## Related Snippets

<!-- catalog:related:start -->
- [Audit Bounded Boolean Test Rows for Unique-Cause MC/DC Coverage](audit-bounded-boolean-test-rows-for-unique-cause-mc-dc-coverage.md)
- [Enumerate Every Contiguous Chunking of Bounded Bytes for Stream Tests](enumerate-every-contiguous-chunking-of-bounded-bytes-for-stream-tests.md)
<!-- catalog:related:end -->
