---
title: "Collect Validated Metric Specifications from Explicit Classes"
snippet_type: pattern
use_cases:
  - observability
  - configuration
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - group-metric-samples-by-their-exact-label-key-shape.md
  - count-values-in-fixed-upper-bound-bins.md
  - ../python-language/collect-decorated-methods-in-class-definition-order.md
---

# Collect Validated Metric Specifications from Explicit Classes

## Idea and Problem

Turn a small explicit tuple of metric declaration classes into immutable, validated specifications.

The caller supplies every class, so collection has no hidden discovery step.
Reading metadata along each class's ordinary inheritance chain permits shared
defaults while retaining a deterministic input order. Validation closes the
kind vocabulary and bucket rules before any specification tuple is returned.

## When to Use

Use this pattern when class declarations make a compact metric inventory easy
to review and the complete inventory is already known at the call site. It is
appropriate for a fixed set of counters and histograms that needs a neutral,
backend-independent description before later wiring.

Prefer plain frozen values when inheritance adds no clarity. Use a separate
registration mechanism when declarations must be selected dynamically.

## Implementation

```python
import math
import re
from dataclasses import dataclass
from typing import ClassVar


_MAX_DECLARATIONS = 32
_MAX_HISTOGRAM_BOUNDS = 32
_METRIC_NAME = re.compile(r"[a-z][a-z0-9_]{0,63}", re.ASCII)
_METRIC_KINDS = frozenset({"counter", "histogram"})
_UNSET = object()


class MetricDeclaration:
    metric_name: ClassVar[object] = _UNSET
    metric_kind: ClassVar[object] = _UNSET
    bucket_bounds: ClassVar[object] = ()


@dataclass(frozen=True, slots=True)
class MetricSpecification:
    name: str
    kind: str
    bucket_bounds: tuple[float, ...]


def _inherited_metadata(declaration: type, attribute: str) -> object:
    for owner in declaration.__mro__:
        if attribute in owner.__dict__:
            return owner.__dict__[attribute]
    return _UNSET


def _validated_bounds(value: object, *, kind: str) -> tuple[float, ...]:
    if type(value) is not tuple:
        raise TypeError("bucket_bounds must be an exact tuple")
    if kind == "counter":
        if value:
            raise ValueError("a counter cannot declare bucket bounds")
        return ()
    if not 1 <= len(value) <= _MAX_HISTOGRAM_BOUNDS:
        raise ValueError(
            "a histogram must declare between 1 and 32 bucket bounds"
        )

    bounds: list[float] = []
    for index, bound in enumerate(value):
        if type(bound) not in (int, float):
            raise TypeError(f"bucket_bounds[{index}] must be an integer or float")
        try:
            normalized = float(bound)
        except OverflowError as error:
            raise ValueError(f"bucket_bounds[{index}] must be finite") from error
        if not math.isfinite(normalized) or normalized <= 0.0:
            raise ValueError(
                f"bucket_bounds[{index}] must be finite and positive"
            )
        if bounds and normalized <= bounds[-1]:
            raise ValueError("histogram bucket bounds must be strictly increasing")
        bounds.append(normalized)
    return tuple(bounds)


def collect_metric_specifications(
    declarations: tuple[type[MetricDeclaration], ...],
) -> tuple[MetricSpecification, ...]:
    if type(declarations) is not tuple:
        raise TypeError("declarations must be an exact tuple")
    if not 1 <= len(declarations) <= _MAX_DECLARATIONS:
        raise ValueError("declarations must contain between 1 and 32 classes")

    specifications: list[MetricSpecification] = []
    for index, declaration in enumerate(declarations):
        if not isinstance(declaration, type):
            raise TypeError(f"declarations[{index}] must be a class")
        if declaration is MetricDeclaration or not issubclass(
            declaration,
            MetricDeclaration,
        ):
            raise TypeError(
                f"declarations[{index}] must be a MetricDeclaration subclass"
            )

        name = _inherited_metadata(declaration, "metric_name")
        if type(name) is not str or _METRIC_NAME.fullmatch(name) is None:
            raise ValueError(
                f"declarations[{index}].metric_name is not a conservative name"
            )
        kind = _inherited_metadata(declaration, "metric_kind")
        if type(kind) is not str or kind not in _METRIC_KINDS:
            raise ValueError(
                f"declarations[{index}].metric_kind must be counter or histogram"
            )
        bounds = _validated_bounds(
            _inherited_metadata(declaration, "bucket_bounds"),
            kind=kind,
        )
        specifications.append(MetricSpecification(name, kind, bounds))

    seen_names: set[str] = set()
    for specification in specifications:
        if specification.name in seen_names:
            raise ValueError(f"duplicate metric name: {specification.name!r}")
        seen_names.add(specification.name)
    return tuple(specifications)
```

## Example

```python
class AcceptedItems(MetricDeclaration):
    metric_name = "items_accepted_total"
    metric_kind = "counter"


class TimedMetric(MetricDeclaration):
    metric_kind = "histogram"


class EvaluationDuration(TimedMetric):
    metric_name = "evaluation_duration_seconds"
    bucket_bounds = (0.02, 0.2, 2.0)


specifications = collect_metric_specifications(
    (AcceptedItems, EvaluationDuration),
)


class ReusedName(MetricDeclaration):
    metric_name = "items_accepted_total"
    metric_kind = "histogram"
    bucket_bounds = (0.5,)


try:
    collect_metric_specifications((AcceptedItems, ReusedName))
except ValueError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

assert (specifications, duplicate_rejected) == (
    (
        MetricSpecification("items_accepted_total", "counter", ()),
        MetricSpecification(
            "evaluation_duration_seconds",
            "histogram",
            (0.02, 0.2, 2.0),
        ),
    ),
    True,
)
```

## Trade-offs and Limitations

Validation is linear in at most 32 declarations and 32 bounds per histogram.
Only plain inherited class attributes are read; descriptors and computed
metadata are rejected by the value checks. The function never inspects module
namespaces, global names, or the runtime subclass graph, so the result cannot
vary with import order.

The returned records describe only names, kinds, and bounds. They do not create
backend objects, accept measurements, define labels, or perform registration.
Extending the closed kind set or its validation rules is an explicit contract
change.

## Related Snippets

<!-- catalog:related:start -->
- [Group Metric Samples by Their Exact Label-Key Shape](group-metric-samples-by-their-exact-label-key-shape.md)
- [Count Values in Fixed Upper-Bound Bins](count-values-in-fixed-upper-bound-bins.md)
- [Collect Decorated Methods in Class Definition Order](../python-language/collect-decorated-methods-in-class-definition-order.md)
<!-- catalog:related:end -->
