---
title: "Generate Binary64 Boundary Probes with math.nextafter"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - generate-integer-boundary-probes-around-closed-cut-points.md
  - ../algorithms-data-structures/evaluate-a-bounded-float-polynomial-with-fused-horner-steps.md
  - ../machine-learning-statistics/compute-a-bounded-numerically-stable-log-sum-exp-for-finite-floats.md
---

# Generate Binary64 Boundary Probes with math.nextafter

## Idea and Problem

Generate exact representable floating-point neighbors on both sides of bounded finite centers instead of guessing decimal epsilon values.

The distance between adjacent binary floating-point numbers changes with
magnitude. `math.nextafter` moves through that representable lattice directly,
and its `steps` argument makes a small radius explicit. Keeping the original
center and two ordered tuples preserves duplicates and the sign bit of zero.

## When to Use

Use these probes in tests for numeric thresholds, interval endpoints,
serialization boundaries, and rounding behavior on a known IEEE-754 binary64
runtime. The result immediately exposes the first through `radius`-th values
toward negative and positive infinity.

These are representation-level probes, not application tolerances. Do not use
them as an approximate-equality policy or assume that one representable step is
a meaningful measurement error. Decimal formats, arbitrary-precision numbers,
and platforms that fail the binary64 guard need their own probe generator.

## Implementation

```python
import math
import struct
import sys
from dataclasses import dataclass

_MAX_CENTERS = 256
_MAX_RADIUS = 32


@dataclass(frozen=True, slots=True)
class Binary64BoundaryProbes:
    center: float
    below: tuple[float, ...]
    above: tuple[float, ...]


def _require_binary64_runtime() -> None:
    characteristics = (
        sys.float_info.radix,
        sys.float_info.mant_dig,
        sys.float_info.min_exp,
        sys.float_info.max_exp,
        struct.calcsize("d"),
    )
    ieee_formats = {"IEEE, little-endian", "IEEE, big-endian"}
    if (
        characteristics != (2, 53, -1021, 1024, 8)
        or float.__getformat__("double") not in ieee_formats
    ):
        raise RuntimeError("this helper requires IEEE-754 binary64 floats")


def generate_binary64_boundary_probes(
    centers: tuple[float, ...],
    radius: int,
) -> tuple[Binary64BoundaryProbes, ...]:
    if type(centers) is not tuple:
        raise TypeError("centers must be an exact tuple")
    if len(centers) > _MAX_CENTERS:
        raise ValueError("centers exceed the supported count")
    if type(radius) is not int:
        raise TypeError("radius must be an exact integer")
    if not 1 <= radius <= _MAX_RADIUS:
        raise ValueError("radius is outside the supported range")
    _require_binary64_runtime()

    results: list[Binary64BoundaryProbes] = []
    for center in centers:
        if type(center) is not float:
            raise TypeError("centers must contain exact floats")
        if not math.isfinite(center):
            raise ValueError("centers must be finite")
        below = tuple(
            math.nextafter(center, -math.inf, steps=step)
            for step in range(1, radius + 1)
        )
        above = tuple(
            math.nextafter(center, math.inf, steps=step)
            for step in range(1, radius + 1)
        )
        results.append(Binary64BoundaryProbes(center, below, above))
    return tuple(results)
```

## Example

```python
def binary64_hex(value: float) -> str:
    return struct.pack(">d", value).hex()


one, negative_zero = generate_binary64_boundary_probes((1.0, -0.0), radius=2)

assert tuple(map(binary64_hex, one.below)) == (
    "3fefffffffffffff",
    "3feffffffffffffe",
)
assert tuple(map(binary64_hex, one.above)) == (
    "3ff0000000000001",
    "3ff0000000000002",
)
assert binary64_hex(negative_zero.center) == "8000000000000000"
assert tuple(map(binary64_hex, negative_zero.below)) == (
    "8000000000000001",
    "8000000000000002",
)
assert tuple(map(binary64_hex, negative_zero.above)) == (
    "0000000000000001",
    "0000000000000002",
)

largest_finite = math.nextafter(math.inf, 0.0)
first, duplicate, extreme = generate_binary64_boundary_probes(
    (1.0, 1.0, largest_finite),
    radius=3,
)
assert first == duplicate

below = first.center
above = first.center
for index in range(3):
    below = math.nextafter(below, -math.inf)
    above = math.nextafter(above, math.inf)
    assert first.below[index] == below
    assert first.above[index] == above
assert all(math.isinf(value) for value in extreme.above)
```

## Trade-offs and Limitations

For `C` centers and radius `R`, generation uses `Theta(C * R)` time and
returned state. The input and output limits make the exact number of
`nextafter` calls explicit.

A direction can reach infinity from a finite extreme. Infinity is a valid
output here, and larger requested steps may repeat it after saturation. Inputs
must remain finite so the two directions always begin at an ordinary binary64
value. Duplicate centers are retained, and signed zeros must not be
deduplicated through equality or a set because `0.0 == -0.0` is true.

The runtime guard checks Python's reported radix, precision, exponent range,
and native double size. It does not establish the floating-point behavior of
external hardware, file formats, databases, or non-Python code. The generated
values are useful test cases, but callers still own expected domain behavior
at each boundary.

## Related Snippets

<!-- catalog:related:start -->
- [Generate Integer Boundary Probes Around Closed Cut Points](generate-integer-boundary-probes-around-closed-cut-points.md)
- [Evaluate a Bounded Float Polynomial with Fused Horner Steps](../algorithms-data-structures/evaluate-a-bounded-float-polynomial-with-fused-horner-steps.md)
- [Compute a Bounded Numerically Stable Log-Sum-Exp for Finite Floats](../machine-learning-statistics/compute-a-bounded-numerically-stable-log-sum-exp-for-finite-floats.md)
<!-- catalog:related:end -->
