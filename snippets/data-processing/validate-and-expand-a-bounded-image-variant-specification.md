---
title: "Validate and Expand a Bounded Image Variant Specification"
snippet_type: algorithm
use_cases:
  - data-transformation
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/map-points-between-rectangular-coordinate-spaces.md
  - ../configuration-serialization/expand-a-bounded-plan-matrix-with-explicit-target-overrides.md
  - project-bounded-records-into-multiple-closed-output-schemas.md
---

# Validate and Expand a Bounded Image Variant Specification

## Idea and Problem

Turn an exact source pixel geometry and a declaration-ordered tuple of closed variant specifications into immutable geometry plans while preserving the source's reduced aspect ratio without upscaling.

The function preflights every specification, pixel budget, name, output
geometry, and rendered key before it returns anything. Consequently, two
declarations cannot silently identify the same output or produce the same
geometry under different names.

## When to Use

Use this algorithm at a trusted in-memory boundary where callers already have
the exact decoded source width and height and need a small, deterministic set
of resize geometries. It is particularly useful when floating-point rounding
or implicit enlargement would make cache or job planning ambiguous.

The specification deliberately requires `UpscalePolicy.FORBID`. A caller that
wants enlargement, cropping, padding, or approximate aspect ratios needs a
different explicit contract.

## Implementation

```python
import math
import re
from dataclasses import dataclass
from enum import StrEnum
from fractions import Fraction

_NAME = re.compile(r"[a-z][a-z0-9_-]{0,31}", re.ASCII).fullmatch
_MAX_DIMENSION = 32_768
_MAX_SOURCE_PIXELS = 100_000_000
_MAX_VARIANTS = 16
_MAX_BOX_PIXELS = 100_000_000
_MAX_OUTPUT_PIXELS = 240_000_000


class UpscalePolicy(StrEnum):
    FORBID = "forbid"


class VariantKind(StrEnum):
    ORIGINAL = "original"
    FIT_WITHIN = "fit-within"


@dataclass(frozen=True, slots=True)
class PixelGeometry:
    width: int
    height: int


@dataclass(frozen=True, slots=True)
class Original:
    name: str


@dataclass(frozen=True, slots=True)
class FitWithin:
    name: str
    max_width: int
    max_height: int
    upscale_policy: UpscalePolicy


type VariantSpec = Original | FitWithin


@dataclass(frozen=True, slots=True)
class GeometryPlan:
    name: str
    kind: VariantKind
    geometry: PixelGeometry
    scale: Fraction
    rendered_key: str


def _dimension(value: object, *, label: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{label} must be an exact non-boolean integer")
    if not 1 <= value <= _MAX_DIMENSION:
        raise ValueError(f"{label} is outside the supported range")
    return value


def _variant_name(value: object) -> str:
    if type(value) is not str or _NAME(value) is None:
        raise ValueError("variant names must use conservative ASCII syntax")
    return value


def _fit_geometry(
    source: PixelGeometry,
    *,
    max_width: int,
    max_height: int,
) -> tuple[PixelGeometry, Fraction]:
    divisor = math.gcd(source.width, source.height)
    ratio_width = source.width // divisor
    ratio_height = source.height // divisor
    exact_units = min(
        Fraction(max_width, ratio_width),
        Fraction(max_height, ratio_height),
        Fraction(divisor, 1),
    )
    units = exact_units.numerator // exact_units.denominator
    if units < 1:
        raise ValueError("fit box cannot hold the exact source aspect ratio")
    return (
        PixelGeometry(ratio_width * units, ratio_height * units),
        Fraction(units, divisor),
    )


def expand_image_variants(
    source: PixelGeometry,
    specs: tuple[VariantSpec, ...],
) -> tuple[GeometryPlan, ...]:
    if type(source) is not PixelGeometry:
        raise TypeError("source must be an exact PixelGeometry")
    source_width = _dimension(source.width, label="source width")
    source_height = _dimension(source.height, label="source height")
    if source_width * source_height > _MAX_SOURCE_PIXELS:
        raise ValueError("source geometry exceeds the pixel budget")

    if type(specs) is not tuple:
        raise TypeError("specs must be a tuple")
    if not 1 <= len(specs) <= _MAX_VARIANTS:
        raise ValueError("variant count is outside the supported range")

    checked: list[tuple[str, VariantKind, PixelGeometry, Fraction, str]] = []
    names: set[str] = set()
    geometries: set[tuple[int, int]] = set()
    rendered_keys: set[str] = set()
    aggregate_pixels = 0

    for spec in specs:
        if type(spec) is Original:
            name = _variant_name(spec.name)
            kind = VariantKind.ORIGINAL
            geometry = PixelGeometry(source_width, source_height)
            scale = Fraction(1, 1)
        elif type(spec) is FitWithin:
            name = _variant_name(spec.name)
            max_width = _dimension(spec.max_width, label="fit width")
            max_height = _dimension(spec.max_height, label="fit height")
            if max_width * max_height > _MAX_BOX_PIXELS:
                raise ValueError("fit box exceeds the pixel budget")
            if (
                type(spec.upscale_policy) is not UpscalePolicy
                or spec.upscale_policy is not UpscalePolicy.FORBID
            ):
                raise ValueError("fit variants must explicitly forbid upscaling")
            kind = VariantKind.FIT_WITHIN
            geometry, scale = _fit_geometry(
                source,
                max_width=max_width,
                max_height=max_height,
            )
        else:
            raise TypeError("specs must contain exact Original or FitWithin values")

        if name in names:
            raise ValueError("variant names must be unique")
        geometry_key = (geometry.width, geometry.height)
        if geometry_key in geometries:
            raise ValueError("variant output geometries must be unique")
        rendered_key = f"{name}.{geometry.width}x{geometry.height}"
        if rendered_key in rendered_keys:
            raise ValueError("variant rendered keys must be unique")

        pixels = geometry.width * geometry.height
        if pixels > _MAX_SOURCE_PIXELS:
            raise ValueError("variant output exceeds the per-output pixel budget")
        aggregate_pixels += pixels
        if aggregate_pixels > _MAX_OUTPUT_PIXELS:
            raise ValueError("variant outputs exceed the aggregate pixel budget")

        names.add(name)
        geometries.add(geometry_key)
        rendered_keys.add(rendered_key)
        checked.append((name, kind, geometry, scale, rendered_key))

    return tuple(GeometryPlan(*fields) for fields in checked)
```

The two exact specification classes make dimensions exclusive to `FitWithin`;
exact-class checks also reject subclasses or look-alike objects. Reduction by
the source dimensions' greatest common divisor means every accepted fit has
exactly the same integer aspect ratio as the source.

## Example

```python
source = PixelGeometry(width=3_000, height=2_000)
specs = (
    Original(name="source"),
    FitWithin(
        name="preview",
        max_width=900,
        max_height=900,
        upscale_policy=UpscalePolicy.FORBID,
    ),
    FitWithin(
        name="thumbnail",
        max_width=240,
        max_height=180,
        upscale_policy=UpscalePolicy.FORBID,
    ),
)

plans = expand_image_variants(source, specs)

assert [(plan.name, plan.geometry, plan.scale, plan.rendered_key) for plan in plans] == [
    ("source", PixelGeometry(3_000, 2_000), Fraction(1, 1), "source.3000x2000"),
    ("preview", PixelGeometry(900, 600), Fraction(3, 10), "preview.900x600"),
    ("thumbnail", PixelGeometry(240, 160), Fraction(2, 25), "thumbnail.240x160"),
]
```

## Trade-offs and Limitations

- Exact ratio preservation can reject a very small box when the source's reduced ratio has a large width or height; it never rounds one axis independently.
- The function materializes every plan and intentionally rejects two names that resolve to the same geometry.
- Pixel counts bound logical work, not encoded byte size, memory consumption, or processing time.
- Inputs are already-known pixel dimensions. This code does not parse JSON or URLs, infer schemes, access files or networks, inspect image data, or decode, resize, crop, pad, encode, or assess quality or format availability.

## Related Snippets

<!-- catalog:related:start -->
- [Map Points Between Rectangular Coordinate Spaces](../algorithms-data-structures/map-points-between-rectangular-coordinate-spaces.md)
- [Expand a Bounded Plan Matrix with Explicit Target Overrides](../configuration-serialization/expand-a-bounded-plan-matrix-with-explicit-target-overrides.md)
- [Project Bounded Records into Multiple Closed Output Schemas](project-bounded-records-into-multiple-closed-output-schemas.md)
<!-- catalog:related:end -->
