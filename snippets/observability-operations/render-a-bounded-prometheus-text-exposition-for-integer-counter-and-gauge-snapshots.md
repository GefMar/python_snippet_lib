---
title: "Render a Bounded Prometheus Text Exposition for Integer Counter and Gauge Snapshots"
snippet_type: recipe
use_cases:
  - observability
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - collect-validated-metric-specifications-from-explicit-classes.md
  - group-metric-samples-by-their-exact-label-key-shape.md
  - route-metric-label-values-through-a-stable-overflow-bucket-under-a-cardinality-cap.md
---

# Render a Bounded Prometheus Text Exposition for Integer Counter and Gauge Snapshots

## Idea and Problem

Render a deterministic Prometheus text snapshot without silently merging equivalent series or expanding into the full exposition protocol.

The input is a closed set of counter and gauge families with integer values,
legacy ASCII names, explicit HELP text, and finite label tuples. Validation
normalizes label order, rejects reserved or empty label states, and detects a
duplicate complete series before rendering canonical HELP, TYPE, and sample
lines.

## When to Use

Use this recipe for a small exporter or deterministic fixture whose complete
integer metric snapshot is already materialized and whose consumer accepts the
Prometheus text format with legacy metric names. It is useful when stable byte
output and duplicate-series rejection matter more than dynamic registration.

Use an official client library for long-lived collectors, concurrent updates,
HTTP negotiation, standard process metrics, histograms, summaries, exemplars,
native histograms, or OpenMetrics. This renderer does not make a counter
monotonic across scrapes or decide whether a label has acceptable cardinality.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum

_MIN_INT64 = -(2**63)
_MAX_INT64 = 2**63 - 1
_MAX_FAMILIES = 64
_MAX_SAMPLES = 4_096
_MAX_LABELS = 32
_MAX_HELP_BYTES = 1_024
_MAX_LABEL_VALUE_BYTES = 1_024
_MAX_INPUT_TEXT_BYTES = 1_048_576
_MAX_OUTPUT_BYTES = 1_048_576
_METRIC_NAME = re.compile(
    r"[A-Za-z_:][A-Za-z0-9_:]{0,127}",
    re.ASCII,
).fullmatch
_LABEL_NAME = re.compile(
    r"[A-Za-z_][A-Za-z0-9_]{0,127}",
    re.ASCII,
).fullmatch


class MetricKind(StrEnum):
    COUNTER = "counter"
    GAUGE = "gauge"


@dataclass(frozen=True, slots=True)
class MetricLabel:
    name: str
    value: str


@dataclass(frozen=True, slots=True)
class IntegerMetricSample:
    labels: tuple[MetricLabel, ...]
    value: int


@dataclass(frozen=True, slots=True)
class IntegerMetricFamily:
    name: str
    kind: MetricKind
    help_text: str
    samples: tuple[IntegerMetricSample, ...]


def _validated_text(
    value: object,
    *,
    field: str,
    minimum_bytes: int,
    maximum_bytes: int,
) -> tuple[str, int]:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    try:
        size = len(value.encode("utf-8"))
    except UnicodeEncodeError:
        raise ValueError(f"{field} must be UTF-8 encodable") from None
    if not minimum_bytes <= size <= maximum_bytes:
        raise ValueError(f"{field} is outside its UTF-8 byte limit")
    if any(
        (ord(character) < 0x20 and character not in "\n\t")
        or ord(character) == 0x7F
        for character in value
    ):
        raise ValueError(f"{field} contains a disallowed control character")
    return value, size


def _escape_help(value: str) -> str:
    return value.replace("\\", "\\\\").replace("\n", "\\n")


def _escape_label(value: str) -> str:
    return (
        value.replace("\\", "\\\\")
        .replace("\n", "\\n")
        .replace('"', '\\"')
    )


def render_prometheus_text(
    families: tuple[IntegerMetricFamily, ...],
) -> str:
    if type(families) is not tuple:
        raise TypeError("families must be an exact tuple")
    if not 1 <= len(families) <= _MAX_FAMILIES:
        raise ValueError("family count is outside 1..64")

    normalized_families: list[IntegerMetricFamily] = []
    seen_family_names: set[str] = set()
    total_samples = 0
    input_text_bytes = 0
    for family_index, family in enumerate(families):
        field = f"families[{family_index}]"
        if type(family) is not IntegerMetricFamily:
            raise TypeError(f"{field} must be an exact IntegerMetricFamily")
        if type(family.name) is not str:
            raise TypeError(f"{field}.name must be an exact string")
        if _METRIC_NAME(family.name) is None:
            raise ValueError(f"{field}.name is outside the legacy metric grammar")
        if family.name in seen_family_names:
            raise ValueError("family names must be unique")
        seen_family_names.add(family.name)
        if type(family.kind) is not MetricKind:
            raise TypeError(f"{field}.kind must be an exact MetricKind")
        help_text, help_bytes = _validated_text(
            family.help_text,
            field=f"{field}.help_text",
            minimum_bytes=0,
            maximum_bytes=_MAX_HELP_BYTES,
        )
        input_text_bytes += help_bytes
        if input_text_bytes > _MAX_INPUT_TEXT_BYTES:
            raise ValueError("HELP and label text exceeds the input byte limit")

        if type(family.samples) is not tuple:
            raise TypeError(f"{field}.samples must be an exact tuple")
        if not family.samples:
            raise ValueError(f"{field}.samples must not be empty")
        total_samples += len(family.samples)
        if total_samples > _MAX_SAMPLES:
            raise ValueError("total sample count exceeds 4096")

        normalized_samples: list[IntegerMetricSample] = []
        seen_series: set[tuple[MetricLabel, ...]] = set()
        for sample_index, sample in enumerate(family.samples):
            sample_field = f"{field}.samples[{sample_index}]"
            if type(sample) is not IntegerMetricSample:
                raise TypeError(
                    f"{sample_field} must be an exact IntegerMetricSample"
                )
            if type(sample.labels) is not tuple:
                raise TypeError(f"{sample_field}.labels must be an exact tuple")
            if len(sample.labels) > _MAX_LABELS:
                raise ValueError(f"{sample_field}.labels count exceeds 32")

            checked_labels: list[MetricLabel] = []
            seen_label_names: set[str] = set()
            for label_index, label in enumerate(sample.labels):
                label_field = f"{sample_field}.labels[{label_index}]"
                if type(label) is not MetricLabel:
                    raise TypeError(f"{label_field} must be an exact MetricLabel")
                if type(label.name) is not str:
                    raise TypeError(f"{label_field}.name must be an exact string")
                if _LABEL_NAME(label.name) is None or label.name.startswith("__"):
                    raise ValueError(
                        f"{label_field}.name is outside the public label grammar"
                    )
                if label.name in seen_label_names:
                    raise ValueError(f"{sample_field}.labels names must be unique")
                seen_label_names.add(label.name)
                label_value, value_bytes = _validated_text(
                    label.value,
                    field=f"{label_field}.value",
                    minimum_bytes=1,
                    maximum_bytes=_MAX_LABEL_VALUE_BYTES,
                )
                input_text_bytes += value_bytes
                if input_text_bytes > _MAX_INPUT_TEXT_BYTES:
                    raise ValueError("HELP and label text exceeds the input byte limit")
                checked_labels.append(MetricLabel(label.name, label_value))

            normalized_labels = tuple(
                sorted(checked_labels, key=lambda label: label.name)
            )
            if normalized_labels in seen_series:
                raise ValueError(f"{field} contains a duplicate complete series")
            seen_series.add(normalized_labels)

            if type(sample.value) is not int:
                raise TypeError(f"{sample_field}.value must be an exact integer")
            minimum_value = 0 if family.kind is MetricKind.COUNTER else _MIN_INT64
            if not minimum_value <= sample.value <= _MAX_INT64:
                raise ValueError(f"{sample_field}.value is outside its kind's range")
            normalized_samples.append(
                IntegerMetricSample(normalized_labels, sample.value)
            )

        normalized_families.append(
            IntegerMetricFamily(
                family.name,
                family.kind,
                help_text,
                tuple(
                    sorted(
                        normalized_samples,
                        key=lambda sample: tuple(
                            (label.name, label.value) for label in sample.labels
                        ),
                    )
                ),
            )
        )

    lines: list[str] = []
    output_bytes = 0

    def append_line(line: str) -> None:
        nonlocal output_bytes
        output_bytes += len(line.encode("utf-8"))
        if output_bytes > _MAX_OUTPUT_BYTES:
            raise ValueError("rendered exposition exceeds the output byte limit")
        lines.append(line)

    for family in sorted(normalized_families, key=lambda item: item.name):
        append_line(f"# HELP {family.name} {_escape_help(family.help_text)}\n")
        append_line(f"# TYPE {family.name} {family.kind.value}\n")
        for sample in family.samples:
            if sample.labels:
                rendered_labels = ",".join(
                    f'{label.name}="{_escape_label(label.value)}"'
                    for label in sample.labels
                )
                series = f"{family.name}{{{rendered_labels}}}"
            else:
                series = family.name
            append_line(f"{series} {sample.value}\n")
    return "".join(lines)
```

## Example

```python
def unescape_prometheus(value: str) -> str:
    decoded: list[str] = []
    index = 0
    while index < len(value):
        if value[index] != "\\":
            decoded.append(value[index])
            index += 1
            continue
        escape = value[index + 1]
        decoded.append("\n" if escape == "n" else escape)
        index += 2
    return "".join(decoded)


def parse_labels(value: str) -> tuple[tuple[str, str], ...]:
    labels: list[tuple[str, str]] = []
    index = 0
    while index < len(value):
        name_end = value.index('="', index)
        name = value[index:name_end]
        index = name_end + 2
        encoded: list[str] = []
        while True:
            character = value[index]
            if character == "\\":
                encoded.extend((character, value[index + 1]))
                index += 2
            elif character == '"':
                index += 1
                break
            else:
                encoded.append(character)
                index += 1
        labels.append((name, unescape_prometheus("".join(encoded))))
        if index < len(value):
            assert value[index] == ","
            index += 1
    return tuple(labels)


def parse_narrow_exposition(
    rendered: str,
) -> tuple[tuple[str, str, str, tuple[tuple[tuple[str, str], ...], int], ...], ...]:
    metadata: dict[str, tuple[str, str]] = {}
    samples: dict[str, list[tuple[tuple[tuple[str, str], ...], int]]] = {}
    for line in rendered.splitlines():
        if line.startswith("# HELP "):
            name, encoded_help = line[7:].split(" ", 1)
            previous = metadata.get(name, ("", ""))
            metadata[name] = (unescape_prometheus(encoded_help), previous[1])
        elif line.startswith("# TYPE "):
            name, kind = line[7:].split(" ", 1)
            previous = metadata.get(name, ("", ""))
            metadata[name] = (previous[0], kind)
        else:
            series, raw_value = line.rsplit(" ", 1)
            if "{" in series:
                name, encoded_labels = series[:-1].split("{", 1)
                labels = parse_labels(encoded_labels)
            else:
                name = series
                labels = ()
            samples.setdefault(name, []).append((labels, int(raw_value)))
    return tuple(
        (name, metadata[name][1], metadata[name][0], tuple(samples[name]))
        for name in sorted(metadata)
    )


families = (
    IntegerMetricFamily(
        "temperature_celsius",
        MetricKind.GAUGE,
        "",
        (IntegerMetricSample((), -2),),
    ),
    IntegerMetricFamily(
        "requests_total",
        MetricKind.COUNTER,
        "Accepted\\requests\nby path",
        (
            IntegerMetricSample((MetricLabel("method", "POST"),), 5),
            IntegerMetricSample(
                (
                    MetricLabel("path", '/a"b'),
                    MetricLabel("method", "GET"),
                ),
                3,
            ),
        ),
    ),
)
rendered = render_prometheus_text(families)
reordered = tuple(
    IntegerMetricFamily(
        family.name,
        family.kind,
        family.help_text,
        tuple(
            IntegerMetricSample(tuple(reversed(sample.labels)), sample.value)
            for sample in reversed(family.samples)
        ),
    )
    for family in reversed(families)
)

expected_model = (
    (
        "requests_total",
        "counter",
        "Accepted\\requests\nby path",
        (
            ((("method", "GET"), ("path", '/a"b')), 3),
            ((("method", "POST"),), 5),
        ),
    ),
    ("temperature_celsius", "gauge", "", (((), -2),)),
)

escape_values = ("\\", '"', "\n", "\t", '\\"\n')
escaping_family = IntegerMetricFamily(
    "escaping_gauge",
    MetricKind.GAUGE,
    'slash\\ quote" newline\n tab\t',
    tuple(
        IntegerMetricSample((MetricLabel("value", value),), index)
        for index, value in enumerate(escape_values)
    ),
)
escaping_model = parse_narrow_exposition(
    render_prometheus_text((escaping_family,))
)
expected_escaping_samples = tuple(
    ((("value", value),), index)
    for index, value in sorted(
        enumerate(escape_values),
        key=lambda item: item[1],
    )
)

duplicate_series = IntegerMetricFamily(
    "duplicate_gauge",
    MetricKind.GAUGE,
    "",
    (
        IntegerMetricSample(
            (MetricLabel("a", "1"), MetricLabel("b", "2")),
            0,
        ),
        IntegerMetricSample(
            (MetricLabel("b", "2"), MetricLabel("a", "1")),
            0,
        ),
    ),
)
try:
    render_prometheus_text((duplicate_series,))
except ValueError as error:
    duplicate_rejected = "duplicate complete series" in str(error)
else:
    duplicate_rejected = False

invalid_labels = (
    MetricLabel("label", ""),
    MetricLabel("__reserved", "value"),
    MetricLabel("label", "bad\rvalue"),
)
rejected = 0
for invalid_label in invalid_labels:
    try:
        render_prometheus_text(
            (
                IntegerMetricFamily(
                    "metric",
                    MetricKind.GAUGE,
                    "",
                    (IntegerMetricSample((invalid_label,), 0),),
                ),
            )
        )
    except ValueError:
        rejected += 1

boundary_family = IntegerMetricFamily(
    "boundary_gauge",
    MetricKind.GAUGE,
    "h" * 1_024,
    (
        IntegerMetricSample(
            (MetricLabel("label", "v" * 1_024),),
            0,
        ),
    ),
)
boundary_rendered = render_prometheus_text((boundary_family,))
field_limit_rejections = 0
for over_limit_family in (
    IntegerMetricFamily(
        "help_gauge",
        MetricKind.GAUGE,
        "h" * 1_025,
        (IntegerMetricSample((), 0),),
    ),
    IntegerMetricFamily(
        "label_gauge",
        MetricKind.GAUGE,
        "",
        (
            IntegerMetricSample(
                (MetricLabel("label", "v" * 1_025),),
                0,
            ),
        ),
    ),
):
    try:
        render_prometheus_text((over_limit_family,))
    except ValueError:
        field_limit_rejections += 1

large_samples = tuple(
    IntegerMetricSample(
        (MetricLabel("key", f"{index:04d}" + "x" * 1_020),),
        index,
    )
    for index in range(1_024)
)
try:
    render_prometheus_text(
        (
            IntegerMetricFamily(
                "large_gauge",
                MetricKind.GAUGE,
                "",
                large_samples,
            ),
        )
    )
except ValueError as error:
    output_cap_enforced = str(error) == (
        "rendered exposition exceeds the output byte limit"
    )
else:
    output_cap_enforced = False

try:
    render_prometheus_text(
        (
            IntegerMetricFamily(
                "large_gauge",
                MetricKind.GAUGE,
                "h",
                large_samples,
            ),
        )
    )
except ValueError as error:
    input_cap_enforced = str(error) == (
        "HELP and label text exceeds the input byte limit"
    )
else:
    input_cap_enforced = False

try:
    render_prometheus_text(
        (
            IntegerMetricFamily(
                "almost_full_gauge",
                MetricKind.GAUGE,
                "h",
                large_samples[:-1],
            ),
            IntegerMetricFamily(
                "final_help_gauge",
                MetricKind.GAUGE,
                "h" * 1_024,
                (IntegerMetricSample((), 0),),
            ),
        )
    )
except ValueError as error:
    help_cap_enforced = str(error) == (
        "HELP and label text exceeds the input byte limit"
    )
else:
    help_cap_enforced = False

assert (
    rendered == render_prometheus_text(reordered)
    and parse_narrow_exposition(rendered) == expected_model
    and escaping_model
    == (
        (
            "escaping_gauge",
            "gauge",
            'slash\\ quote" newline\n tab\t',
            expected_escaping_samples,
        ),
    )
    and duplicate_rejected
    and rejected == len(invalid_labels)
    and boundary_rendered.endswith("\n")
    and field_limit_rejections == 2
    and output_cap_enforced
    and input_cap_enforced
    and help_cap_enforced
)
```

## Trade-offs and Limitations

For `S` samples and `L` total labels, validation and escaping take
`O(S + L + B)` work for `B` inspected UTF-8 bytes. Sorting families, samples,
and labels adds their ordinary comparison costs; normalized inputs and output
use `O(S + L + B)` memory within the explicit count and byte limits.

Canonical ordering improves reviewability but is not required by Prometheus.
Rejecting empty label values is a deliberate closed-profile rule that prevents
an empty label from becoming a second spelling of absence. Integer snapshots
avoid cross-language float spelling, but they cannot represent fractional,
infinite, or NaN observations even though the broader exposition format can.

## Related Snippets

<!-- catalog:related:start -->
- [Collect Validated Metric Specifications from Explicit Classes](collect-validated-metric-specifications-from-explicit-classes.md)
- [Group Metric Samples by Their Exact Label-Key Shape](group-metric-samples-by-their-exact-label-key-shape.md)
- [Route Metric Label Values Through a Stable Overflow Bucket Under a Cardinality Cap](route-metric-label-values-through-a-stable-overflow-bucket-under-a-cardinality-cap.md)
<!-- catalog:related:end -->
