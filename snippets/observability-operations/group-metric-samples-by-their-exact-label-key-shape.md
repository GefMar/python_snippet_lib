---
title: "Group Metric Samples by Their Exact Label-Key Shape"
snippet_type: algorithm
use_cases:
  - observability
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/group-items-by-an-exact-compatibility-signature-and-report-unmatched-inputs.md
  - ../algorithms-data-structures/group-a-one-dimensional-numpy-structured-array-into-stable-index-sets.md
  - format-log-records-as-json-with-explicit-extra-fields.md
---

# Group Metric Samples by Their Exact Label-Key Shape

## Idea and Problem

Partition immutable metric samples into table plans whose columns are defined by exactly the same label names.

Label order should not create a different schema: `service, zone` and `zone,
service` describe the same pair of columns. Canonicalizing every sample by
label name makes the grouping key deterministic, while sorting the distinct
keys gives callers stable plan order. Label values never affect which table a
sample enters.

## When to Use

Use this algorithm after samples for one metric family have been collected in
memory and a downstream writer needs one homogeneous label schema per table or
batch. Inputs are frozen records containing exact label tuples, native numeric
measurements, and optional numeric timestamps. Empty label sets are supported,
and samples retain their arrival order inside each plan.

Keep metric-family identity outside this function, or include it in a separate
partitioning step. Use a schema-union transform instead when absent labels
should become nullable columns, and use a streaming partitioner when the
bounded in-memory limits are too small.

## Implementation

```python
import math
import re
from dataclasses import dataclass


MetricNumber = int | float
Label = tuple[str, str]
_MAX_SAMPLES = 10_000
_MAX_LABELS_PER_SAMPLE = 16
_MAX_TOTAL_LABELS = 100_000
_MAX_GROUPS = 512
_MAX_LABEL_VALUE_CHARACTERS = 256
_MIN_INTEGER = -(2**63)
_MAX_INTEGER = 2**63 - 1
_LABEL_NAME = re.compile(r"[A-Za-z_][A-Za-z0-9_.-]{0,63}", re.ASCII)


@dataclass(frozen=True, slots=True)
class MetricSample:
    labels: tuple[Label, ...]
    measurement: MetricNumber
    timestamp: MetricNumber | None = None


@dataclass(frozen=True, slots=True)
class MetricTablePlan:
    label_keys: tuple[str, ...]
    samples: tuple[MetricSample, ...]


def _metric_number(
    value: object,
    *,
    field: str,
    non_negative: bool,
) -> MetricNumber:
    if type(value) not in (int, float):
        raise TypeError(f"{field} must be a native integer or float")
    if type(value) is int and not _MIN_INTEGER <= value <= _MAX_INTEGER:
        raise ValueError(f"{field} integer is outside the supported range")
    if type(value) is float and not math.isfinite(value):
        raise ValueError(f"{field} must be finite")
    if non_negative and value < 0:
        raise ValueError(f"{field} must be non-negative")
    return value


def _canonical_labels(value: object) -> tuple[Label, ...]:
    if type(value) is not tuple:
        raise TypeError("labels must be an exact tuple")
    if len(value) > _MAX_LABELS_PER_SAMPLE:
        raise ValueError("label count exceeds the per-sample limit")

    labels: list[Label] = []
    names: set[str] = set()
    for label in value:
        if type(label) is not tuple or len(label) != 2:
            raise TypeError("labels must contain exact name-value tuples")
        name, label_value = label
        if type(name) is not str or _LABEL_NAME.fullmatch(name) is None:
            raise ValueError("a label name is outside the supported format")
        if name in names:
            raise ValueError("label names must be unique within a sample")
        if type(label_value) is not str:
            raise TypeError("label values must be exact strings")
        if (
            len(label_value) > _MAX_LABEL_VALUE_CHARACTERS
            or not all(character.isprintable() for character in label_value)
        ):
            raise ValueError("a label value is outside the supported format")
        names.add(name)
        labels.append((name, label_value))
    return tuple(sorted(labels))


def group_metric_samples_by_label_shape(
    samples: tuple[MetricSample, ...],
) -> tuple[MetricTablePlan, ...]:
    if type(samples) is not tuple:
        raise TypeError("samples must be an exact tuple")
    if len(samples) > _MAX_SAMPLES:
        raise ValueError("sample count exceeds the supported limit")

    total_labels = 0
    rows_by_shape: dict[tuple[str, ...], list[MetricSample]] = {}
    for sample in samples:
        if type(sample) is not MetricSample:
            raise TypeError("samples must contain exact MetricSample values")
        labels = _canonical_labels(sample.labels)
        total_labels += len(labels)
        if total_labels > _MAX_TOTAL_LABELS:
            raise ValueError("total label count exceeds the supported limit")

        measurement = _metric_number(
            sample.measurement,
            field="measurement",
            non_negative=False,
        )
        timestamp = (
            None
            if sample.timestamp is None
            else _metric_number(
                sample.timestamp,
                field="timestamp",
                non_negative=True,
            )
        )
        canonical_sample = MetricSample(labels, measurement, timestamp)
        shape = tuple(name for name, _ in labels)

        rows = rows_by_shape.get(shape)
        if rows is None:
            if len(rows_by_shape) == _MAX_GROUPS:
                raise ValueError("label-shape count exceeds the supported limit")
            rows = []
            rows_by_shape[shape] = rows
        rows.append(canonical_sample)

    return tuple(
        MetricTablePlan(shape, tuple(rows_by_shape[shape]))
        for shape in sorted(rows_by_shape)
    )
```

## Example

```python
samples = (
    MetricSample(
        labels=(("zone", "north"), ("service", "api")),
        measurement=3.5,
        timestamp=10,
    ),
    MetricSample(
        labels=(("service", "worker"),),
        measurement=7,
    ),
    MetricSample(
        labels=(("service", "api"), ("zone", "south")),
        measurement=4.25,
        timestamp=12,
    ),
)

plans = group_metric_samples_by_label_shape(samples)

assert (plans, samples[0].labels) == (
    (
        MetricTablePlan(
            label_keys=("service",),
            samples=(MetricSample((("service", "worker"),), 7, None),),
        ),
        MetricTablePlan(
            label_keys=("service", "zone"),
            samples=(
                MetricSample(
                    (("service", "api"), ("zone", "north")),
                    3.5,
                    10,
                ),
                MetricSample(
                    (("service", "api"), ("zone", "south")),
                    4.25,
                    12,
                ),
            ),
        ),
    ),
    (("zone", "north"), ("service", "api")),
)
```

## Trade-offs and Limitations

For `s` samples with at most `l` labels each, validation and canonicalization
cost `O(s * l log l)` time. Sorting `g` distinct schemas adds `O(g log g)`, and
the copied canonical records require `O(s * l)` memory. The explicit sample,
label, total-label, and group limits keep both work and retained data bounded.

The conservative label-name grammar is narrower than some telemetry systems,
and label values are treated as opaque printable text without normalization.
Measurements and timestamps keep their native integer or float types; integers
are restricted to the signed 64-bit range. This plan therefore does
not choose units, timestamp epochs, precision, serialization, or missing-value
policy. It also does not format rows, aggregate measurements, invent absent
labels, or contact a monitoring backend.

## Related Snippets

<!-- catalog:related:start -->
- [Group Items by an Exact Compatibility Signature and Report Unmatched Inputs](../data-processing/group-items-by-an-exact-compatibility-signature-and-report-unmatched-inputs.md)
- [Group a One-Dimensional NumPy Structured Array into Stable Index Sets](../algorithms-data-structures/group-a-one-dimensional-numpy-structured-array-into-stable-index-sets.md)
- [Format Log Records as JSON with Explicit Extra Fields](format-log-records-as-json-with-explicit-extra-fields.md)
<!-- catalog:related:end -->
