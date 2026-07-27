---
title: "Report Exact Capacity Deficits for Bounded Resource Profiles"
snippet_type: algorithm
use_cases:
  - resource-management
  - validation
  - automation
tested_python:
  - "3.14"
dependencies: []
related:
  - apportion-a-non-negative-integer-total-without-rounding-drift.md
  - ../data-processing/route-estimated-work-by-ordered-source-and-size-rules.md
  - ../data-processing/group-items-by-an-exact-compatibility-signature-and-report-unmatched-inputs.md
---

# Report Exact Capacity Deficits for Bounded Resource Profiles

## Idea and Problem

Compare several fixed resource profiles with one capacity snapshot and report the exact missing amount for every dimension that does not fit.

Reducing multidimensional capacity to a single boolean hides why a profile was
rejected. A frozen report can preserve dimension and profile order, expose the
remaining capacity, and attach positive deficits to each rejected profile.
Exact integers and explicit unit labels keep arithmetic predictable and stop
silent conversion between incompatible measurements.

## When to Use

Use this algorithm when limits and current usage are already available as one
consistent snapshot, every profile uses the same ordered dimensions, and the
decision is simply whether each fixed profile fits. It works well for local
admission previews, capacity reports, and validation before a separate
scheduler or reservation system runs.

Normalize quantities before calling the function. A unit label documents the
contract but does not convert values. Use exact non-negative integers so that
the caller decides whether a quantity represents bytes, blocks, slots, or
another indivisible unit.

## Implementation

```python
import re
from dataclasses import dataclass


_MAX_DIMENSIONS = 32
_MAX_PROFILES = 256
_MAX_CELLS = 8_192
_MAX_AMOUNT = (1 << 63) - 1
_LABEL_PATTERN = re.compile(r"[A-Za-z0-9][A-Za-z0-9_.-]{0,63}")


@dataclass(frozen=True, slots=True)
class CapacityDimension:
    name: str
    unit: str
    limit: int
    used: int


@dataclass(frozen=True, slots=True)
class ResourceProfile:
    name: str
    requirements: tuple[int, ...]


@dataclass(frozen=True, slots=True)
class AvailableDimension:
    name: str
    unit: str
    amount: int


@dataclass(frozen=True, slots=True)
class CapacityDeficit:
    name: str
    unit: str
    missing: int


@dataclass(frozen=True, slots=True)
class ProfileAssessment:
    name: str
    fits: bool
    deficits: tuple[CapacityDeficit, ...]


@dataclass(frozen=True, slots=True)
class CapacityReport:
    available: tuple[AvailableDimension, ...]
    profiles: tuple[ProfileAssessment, ...]


def _require_label(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact str")
    if _LABEL_PATTERN.fullmatch(value) is None:
        raise ValueError(f"{field} has an invalid format")
    return value


def _require_amount(value: object, *, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact int")
    if not 0 <= value <= _MAX_AMOUNT:
        raise ValueError(f"{field} is outside the supported range")
    return value


def report_capacity(
    dimensions: tuple[CapacityDimension, ...],
    profiles: tuple[ResourceProfile, ...],
) -> CapacityReport:
    if type(dimensions) is not tuple:
        raise TypeError("dimensions must be an exact tuple")
    if not 1 <= len(dimensions) <= _MAX_DIMENSIONS:
        raise ValueError("dimension count is outside the supported range")
    if type(profiles) is not tuple:
        raise TypeError("profiles must be an exact tuple")
    if not 1 <= len(profiles) <= _MAX_PROFILES:
        raise ValueError("profile count is outside the supported range")
    if len(dimensions) * len(profiles) > _MAX_CELLS:
        raise ValueError("dimension-by-profile work exceeds the limit")

    dimension_names: set[str] = set()
    available_amounts: list[int] = []
    available_dimensions: list[AvailableDimension] = []

    for dimension_index, dimension in enumerate(dimensions):
        if type(dimension) is not CapacityDimension:
            raise TypeError("every dimension must be CapacityDimension")
        name = _require_label(
            dimension.name,
            field=f"dimensions[{dimension_index}].name",
        )
        unit = _require_label(
            dimension.unit,
            field=f"dimensions[{dimension_index}].unit",
        )
        if name in dimension_names:
            raise ValueError("dimension names must be unique")
        dimension_names.add(name)

        limit = _require_amount(
            dimension.limit,
            field=f"dimensions[{dimension_index}].limit",
        )
        used = _require_amount(
            dimension.used,
            field=f"dimensions[{dimension_index}].used",
        )
        if used > limit:
            raise ValueError("used amount cannot exceed its limit")

        available = limit - used
        available_amounts.append(available)
        available_dimensions.append(
            AvailableDimension(name=name, unit=unit, amount=available)
        )

    profile_names: set[str] = set()
    assessments: list[ProfileAssessment] = []

    for profile_index, profile in enumerate(profiles):
        if type(profile) is not ResourceProfile:
            raise TypeError("every profile must be ResourceProfile")
        profile_name = _require_label(
            profile.name,
            field=f"profiles[{profile_index}].name",
        )
        if profile_name in profile_names:
            raise ValueError("profile names must be unique")
        profile_names.add(profile_name)

        requirements = profile.requirements
        if type(requirements) is not tuple:
            raise TypeError("profile requirements must be an exact tuple")
        if len(requirements) != len(dimensions):
            raise ValueError("profile requirements must align with dimensions")

        deficits: list[CapacityDeficit] = []
        for dimension_index, (dimension, available, required) in enumerate(
            zip(dimensions, available_amounts, requirements, strict=True)
        ):
            required_amount = _require_amount(
                required,
                field=(
                    f"profiles[{profile_index}].requirements"
                    f"[{dimension_index}]"
                ),
            )
            if required_amount > available:
                deficits.append(
                    CapacityDeficit(
                        name=dimension.name,
                        unit=dimension.unit,
                        missing=required_amount - available,
                    )
                )

        frozen_deficits = tuple(deficits)
        assessments.append(
            ProfileAssessment(
                name=profile_name,
                fits=not frozen_deficits,
                deficits=frozen_deficits,
            )
        )

    return CapacityReport(
        available=tuple(available_dimensions),
        profiles=tuple(assessments),
    )
```

## Example

```python
dimensions = (
    CapacityDimension("workers", "count", limit=12, used=5),
    CapacityDimension("buffer", "mib", limit=4_096, used=1_024),
    CapacityDimension("archive", "gib", limit=500, used=120),
)
profiles = (
    ResourceProfile("compact", requirements=(2, 512, 40)),
    ResourceProfile("burst", requirements=(8, 3_500, 400)),
)

report = report_capacity(dimensions, profiles)

assert tuple(item.amount for item in report.available) == (7, 3_072, 380)
assert report.profiles[0] == ProfileAssessment(
    name="compact",
    fits=True,
    deficits=(),
)
assert report.profiles[1].deficits == (
    CapacityDeficit("workers", "count", 1),
    CapacityDeficit("buffer", "mib", 428),
    CapacityDeficit("archive", "gib", 20),
)
assert report.profiles[1].fits is False
```

## Trade-offs and Limitations

The function evaluates one immutable snapshot. It does not reserve capacity,
coordinate concurrent consumers, choose a preferred profile, or guarantee
that capacity remains available after the report is returned. A real
admission system needs an atomic reservation boundary around its final
decision.

Dimensions align by tuple position after their names and units are validated;
this keeps work bounded and ordering explicit, but callers must build every
profile against the same dimension contract. Unit labels are descriptive,
not dimensional-analysis types. Convert values before constructing the input,
and use a domain-specific quantity type when accidental unit mixing remains a
material risk.

## Related Snippets

<!-- catalog:related:start -->
- [Apportion a Non-Negative Integer Total Without Rounding Drift](apportion-a-non-negative-integer-total-without-rounding-drift.md)
- [Route Estimated Work by Ordered Source and Size Rules](../data-processing/route-estimated-work-by-ordered-source-and-size-rules.md)
- [Group Items by an Exact Compatibility Signature and Report Unmatched Inputs](../data-processing/group-items-by-an-exact-compatibility-signature-and-report-unmatched-inputs.md)
<!-- catalog:related:end -->
