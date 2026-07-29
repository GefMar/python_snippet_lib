---
title: "Audit a Bounded Test Matrix for Complete Pairwise Coverage"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/expand-a-bounded-plan-matrix-with-explicit-target-overrides.md
  - generate-a-seeded-metric-with-bounded-flapping-runs.md
  - shrink-a-bounded-failing-sequence-to-a-one-deletion-minimal-subsequence.md
---

# Audit a Bounded Test Matrix for Complete Pairwise Coverage

## Idea and Problem

Verify that every value of each parameter appears with every value of every other parameter in at least one declared test row.

Pairwise coverage is weaker than exhaustive Cartesian coverage, but it can
expose interactions between two settings without requiring every full
combination. This audit validates the complete matrix first, then records the
cross-parameter pairs that actually occur and reports every missing obligation
in declaration order.

## When to Use

Use this audit when a bounded test plan already exists and its intended
criterion is complete pairwise coverage across closed parameter values. The
explicit missing-pair result is useful in review or as a deterministic gate for
a separately generated matrix.

Keep generation separate from verification so a generator cannot silently
certify its own output. Use exhaustive combinations, higher-strength
combinatorial testing, or a constraint-aware tool when interactions among
three or more parameters or invalid combinations are part of the requirement.

## Implementation

```python
from dataclasses import dataclass

_MIN_PARAMETERS = 2
_MAX_PARAMETERS = 12
_MAX_VALUES_PER_PARAMETER = 16
_MAX_ROWS = 4_096
_MAX_TEXT_CHARACTERS = 64
_MAX_TEXT_BYTES = 256

TestRow = tuple[str, ...]


@dataclass(frozen=True, slots=True)
class TestParameter:
    name: str
    values: tuple[str, ...]


@dataclass(frozen=True, slots=True)
class MissingValuePair:
    left_parameter: str
    left_value: str
    right_parameter: str
    right_value: str


@dataclass(frozen=True, slots=True)
class PairwiseCoverageAudit:
    missing_pairs: tuple[MissingValuePair, ...]
    complete: bool


def _validated_text(value: object, *, location: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{location} must be an exact string")
    if not value:
        raise ValueError(f"{location} must not be empty")
    if len(value) > _MAX_TEXT_CHARACTERS:
        raise ValueError(f"{location} exceeds the character limit")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError:
        raise ValueError(f"{location} must be valid UTF-8 text") from None
    if len(encoded) > _MAX_TEXT_BYTES:
        raise ValueError(f"{location} exceeds the UTF-8 byte limit")
    return value


def audit_pairwise_coverage(
    parameters: tuple[TestParameter, ...],
    rows: tuple[TestRow, ...],
) -> PairwiseCoverageAudit:
    """Return all uncovered cross-parameter value pairs in canonical order."""
    if type(parameters) is not tuple:
        raise TypeError("parameters must be an exact tuple")
    if not _MIN_PARAMETERS <= len(parameters) <= _MAX_PARAMETERS:
        raise ValueError("parameter count is outside the supported range")

    parameter_names: set[str] = set()
    value_indexes: list[dict[str, int]] = []
    for parameter_index, parameter in enumerate(parameters):
        if type(parameter) is not TestParameter:
            raise TypeError(f"parameters[{parameter_index}] must be an exact TestParameter")
        name = _validated_text(
            parameter.name,
            location=f"parameters[{parameter_index}].name",
        )
        if name in parameter_names:
            raise ValueError(f"parameters[{parameter_index}].name is duplicated")
        parameter_names.add(name)

        if type(parameter.values) is not tuple:
            raise TypeError(f"parameters[{parameter_index}].values must be an exact tuple")
        if not 1 <= len(parameter.values) <= _MAX_VALUES_PER_PARAMETER:
            raise ValueError(
                f"parameters[{parameter_index}].values count is outside the supported range"
            )

        positions: dict[str, int] = {}
        for value_index, raw_value in enumerate(parameter.values):
            value = _validated_text(
                raw_value,
                location=f"parameters[{parameter_index}].values[{value_index}]",
            )
            if value in positions:
                raise ValueError(
                    f"parameters[{parameter_index}].values[{value_index}] is duplicated"
                )
            positions[value] = value_index
        value_indexes.append(positions)

    if type(rows) is not tuple:
        raise TypeError("rows must be an exact tuple")
    if not 1 <= len(rows) <= _MAX_ROWS:
        raise ValueError("row count is outside the supported range")

    for row_index, row in enumerate(rows):
        if type(row) is not tuple:
            raise TypeError(f"rows[{row_index}] must be an exact tuple")
        if len(row) != len(parameters):
            raise ValueError(f"rows[{row_index}] must contain one value per parameter")
        for parameter_index, value in enumerate(row):
            if type(value) is not str:
                raise TypeError(f"rows[{row_index}][{parameter_index}] must be an exact string")
            if value not in value_indexes[parameter_index]:
                raise ValueError(f"rows[{row_index}][{parameter_index}] is not a declared value")

    covered: set[tuple[int, int, int, int]] = set()
    for row in rows:
        for left_parameter in range(len(parameters) - 1):
            left_value = value_indexes[left_parameter][row[left_parameter]]
            for right_parameter in range(left_parameter + 1, len(parameters)):
                right_value = value_indexes[right_parameter][row[right_parameter]]
                covered.add((left_parameter, left_value, right_parameter, right_value))

    missing: list[MissingValuePair] = []
    for left_parameter in range(len(parameters) - 1):
        for right_parameter in range(left_parameter + 1, len(parameters)):
            for left_value, left_text in enumerate(parameters[left_parameter].values):
                for right_value, right_text in enumerate(parameters[right_parameter].values):
                    key = (
                        left_parameter,
                        left_value,
                        right_parameter,
                        right_value,
                    )
                    if key not in covered:
                        missing.append(
                            MissingValuePair(
                                left_parameter=parameters[left_parameter].name,
                                left_value=left_text,
                                right_parameter=parameters[right_parameter].name,
                                right_value=right_text,
                            )
                        )

    missing_pairs = tuple(missing)
    return PairwiseCoverageAudit(
        missing_pairs=missing_pairs,
        complete=not missing_pairs,
    )
```

## Example

```python
parameters = (
    TestParameter("browser", ("chrome", "firefox")),
    TestParameter("locale", ("en", "fr")),
    TestParameter("theme", ("light", "dark")),
)
rows = (
    ("chrome", "en", "light"),
    ("chrome", "fr", "dark"),
    ("firefox", "en", "dark"),
)

audit = audit_pairwise_coverage(parameters, rows)

assert audit == PairwiseCoverageAudit(
    missing_pairs=(
        MissingValuePair("browser", "firefox", "locale", "fr"),
        MissingValuePair("browser", "firefox", "theme", "light"),
        MissingValuePair("locale", "fr", "theme", "light"),
    ),
    complete=False,
)

completed_rows = (*rows, ("firefox", "fr", "light"))
assert audit_pairwise_coverage(parameters, completed_rows) == PairwiseCoverageAudit(
    missing_pairs=(),
    complete=True,
)
assert audit_pairwise_coverage(parameters, (*rows, rows[0])) == audit
```

## Trade-offs and Limitations

For `R` rows, `D` parameters, and `P` required cross-parameter value pairs,
validation and coverage take expected `O(R * D^2 + P)` set and dictionary work.
The declaration indexes, covered keys, missing-pair list, and frozen output use
`O(P)` memory. The parameter and value limits imply at most 16,896 pair
obligations. Per-name and per-value text limits bound individual hashing and
comparison costs.

The function audits an already materialized exact matrix; it does not generate
rows or prove that the matrix is minimal. Duplicate rows are accepted but do
not add coverage. Every row must use declared values, and there is no model for
forbidden combinations, wildcards, conditional parameters, or missing cells.

Pairwise coverage demonstrates only that every declared cross-parameter pair
appears somewhere. It does not establish three-way or higher interaction
coverage, test quality, independence, randomness, or application correctness.

## Related Snippets

<!-- catalog:related:start -->
- [Expand a Bounded Plan Matrix with Explicit Target Overrides](../configuration-serialization/expand-a-bounded-plan-matrix-with-explicit-target-overrides.md)
- [Generate a Seeded Metric with Bounded Flapping Runs](generate-a-seeded-metric-with-bounded-flapping-runs.md)
- [Shrink a Bounded Failing Sequence to a One-Deletion-Minimal Subsequence](shrink-a-bounded-failing-sequence-to-a-one-deletion-minimal-subsequence.md)
<!-- catalog:related:end -->
