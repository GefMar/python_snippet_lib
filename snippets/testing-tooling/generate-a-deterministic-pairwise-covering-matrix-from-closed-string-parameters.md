---
title: "Generate a Deterministic Pairwise-Covering Matrix from Closed String Parameters"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - audit-a-bounded-test-matrix-for-complete-pairwise-coverage.md
  - ../configuration-serialization/expand-a-bounded-plan-matrix-with-explicit-target-overrides.md
---

# Generate a Deterministic Pairwise-Covering Matrix from Closed String Parameters

## Idea and Problem

Generate test rows that cover every cross-parameter pair without expanding the complete Cartesian product.

The first value of each parameter is its declared default. One all-default row
covers every default-to-default pair, single substitutions cover pairs between
one non-default and every other default, and double substitutions cover pairs
between two non-defaults. Keeping those three groups in a fixed order makes the
matrix reproducible and straightforward to review.

## When to Use

Use this generator when each parameter has a small, closed set of valid string
values, every cross-parameter combination is permitted, and two-way interaction
coverage is an appropriate test-plan target. It is useful when a full Cartesian
matrix is too large but a deterministic, dependency-free construction matters
more than finding the fewest possible rows.

Use a constraint-aware covering-array tool when some combinations are invalid,
or a higher-strength design when failures may require three or more interacting
parameters. Audit the returned matrix independently when pairwise coverage is a
release or compliance requirement.

## Implementation

```python
_MIN_PARAMETERS = 2
_MAX_PARAMETERS = 12
_MAX_VALUES_PER_PARAMETER = 16
_MAX_TEXT_CHARACTERS = 64
_MAX_ROWS = 4_096

TestParameters = tuple[tuple[str, ...], ...]
TestRow = tuple[str, ...]


def _validated_parameters(value: object) -> TestParameters:
    if type(value) is not tuple:
        raise TypeError("parameters must be an exact tuple")
    if not _MIN_PARAMETERS <= len(value) <= _MAX_PARAMETERS:
        raise ValueError("parameter count is outside the supported range")

    validated: list[tuple[str, ...]] = []
    for parameter_index, parameter in enumerate(value):
        if type(parameter) is not tuple:
            raise TypeError(f"parameters[{parameter_index}] must be an exact tuple")
        if not 1 <= len(parameter) <= _MAX_VALUES_PER_PARAMETER:
            raise ValueError(
                f"parameters[{parameter_index}] value count is outside the supported range"
            )

        seen: set[str] = set()
        for value_index, item in enumerate(parameter):
            location = f"parameters[{parameter_index}][{value_index}]"
            if type(item) is not str:
                raise TypeError(f"{location} must be an exact string")
            if not item:
                raise ValueError(f"{location} must not be empty")
            if len(item) > _MAX_TEXT_CHARACTERS:
                raise ValueError(f"{location} exceeds the character limit")
            try:
                item.encode("utf-8")
            except UnicodeEncodeError:
                raise ValueError(f"{location} must be valid UTF-8 text") from None
            if item in seen:
                raise ValueError(f"{location} is duplicated within its parameter")
            seen.add(item)
        validated.append(parameter)

    return tuple(validated)


def generate_pairwise_covering_matrix(parameters: TestParameters) -> tuple[TestRow, ...]:
    """Return default, single-substitution, then double-substitution rows."""
    checked = _validated_parameters(parameters)
    nondefault_counts = tuple(len(parameter) - 1 for parameter in checked)

    row_count = 1 + sum(nondefault_counts)
    for left in range(len(checked) - 1):
        for right in range(left + 1, len(checked)):
            row_count += nondefault_counts[left] * nondefault_counts[right]
    if row_count > _MAX_ROWS:
        raise ValueError("pairwise matrix exceeds the supported row count")

    defaults = tuple(parameter[0] for parameter in checked)
    rows: list[TestRow] = [defaults]

    for parameter_index, parameter in enumerate(checked):
        for item in parameter[1:]:
            row = list(defaults)
            row[parameter_index] = item
            rows.append(tuple(row))

    for left in range(len(checked) - 1):
        for right in range(left + 1, len(checked)):
            for left_item in checked[left][1:]:
                for right_item in checked[right][1:]:
                    row = list(defaults)
                    row[left] = left_item
                    row[right] = right_item
                    rows.append(tuple(row))

    return tuple(rows)
```

## Example

```python
def required_value_pairs(parameters: TestParameters) -> set[tuple[int, str, int, str]]:
    return {
        (left, left_item, right, right_item)
        for left in range(len(parameters) - 1)
        for right in range(left + 1, len(parameters))
        for left_item in parameters[left]
        for right_item in parameters[right]
    }


def covered_value_pairs(rows: tuple[TestRow, ...]) -> set[tuple[int, str, int, str]]:
    return {
        (left, row[left], right, row[right])
        for row in rows
        for left in range(len(row) - 1)
        for right in range(left + 1, len(row))
    }


parameters = (
    ("cpython", "pypy"),
    ("linux", "windows"),
    ("sync", "async"),
)
rows = generate_pairwise_covering_matrix(parameters)
expected_order = (
    ("cpython", "linux", "sync"),
    ("pypy", "linux", "sync"),
    ("cpython", "windows", "sync"),
    ("cpython", "linux", "async"),
    ("pypy", "windows", "sync"),
    ("pypy", "linux", "async"),
    ("cpython", "windows", "async"),
)

single_value_rows = generate_pairwise_covering_matrix(
    (("fixed",), ("small", "large"), ("off", "on"))
)

type_error_count = 0
for invalid in (
    [("a",), ("b",)],
    (("a",), ["b"]),
    (("a",), ("b", 1)),
):
    try:
        generate_pairwise_covering_matrix(invalid)
    except TypeError:
        type_error_count += 1

value_error_count = 0
for invalid in (
    (("a", "a"), ("b",)),
    (("",), ("b",)),
    (("\ud800",), ("b",)),
):
    try:
        generate_pairwise_covering_matrix(invalid)
    except ValueError:
        value_error_count += 1

try:
    generate_pairwise_covering_matrix(tuple(tuple(str(i) for i in range(16)) for _ in range(12)))
except ValueError:
    row_cap_rejected = True
else:
    row_cap_rejected = False

assert (
    rows == expected_order,
    required_value_pairs(parameters) == covered_value_pairs(rows),
    len(rows) == len(set(rows)),
    single_value_rows,
    required_value_pairs((("fixed",), ("small", "large"), ("off", "on")))
    == covered_value_pairs(single_value_rows),
    type_error_count,
    value_error_count,
    row_cap_rejected,
) == (
    True,
    True,
    True,
    (
        ("fixed", "small", "off"),
        ("fixed", "large", "off"),
        ("fixed", "small", "on"),
        ("fixed", "large", "on"),
    ),
    True,
    3,
    3,
    True,
)
```

## Trade-offs and Limitations

For `D` parameters, `V` declared values, and `R` returned rows, validation takes
expected `O(V)` set work plus UTF-8 encoding time. Building the matrix takes
`O(R * D)` time and output-proportional memory because every row contains one
value for every parameter. The temporary row list and returned tuple briefly
coexist at return. The preflight rejects a result above 4,096 rows before any
rows are materialized.

This construction is deterministic and complete but not minimal. Most rows
retain defaults in every unrelated parameter, so the result is deliberately
default-biased rather than balanced. With exactly two parameters it produces
their full Cartesian product; with more parameters it can use fewer rows than
that product, but an optimized covering array can often use fewer still.

The generator cannot express forbidden combinations, conditional parameters,
wildcards, parameter names, or mutable values. Pairwise coverage does not cover
higher-order interactions and does not demonstrate test quality or application
correctness. The function materializes the complete matrix and offers no
streaming or randomized mode.

## Related Snippets

<!-- catalog:related:start -->
- [Audit a Bounded Test Matrix for Complete Pairwise Coverage](audit-a-bounded-test-matrix-for-complete-pairwise-coverage.md)
- [Expand a Bounded Plan Matrix with Explicit Target Overrides](../configuration-serialization/expand-a-bounded-plan-matrix-with-explicit-target-overrides.md)
<!-- catalog:related:end -->
