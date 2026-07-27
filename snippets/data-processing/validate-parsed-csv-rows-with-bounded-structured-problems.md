---
title: "Validate Parsed CSV Rows with Bounded Structured Problems"
snippet_type: recipe
use_cases:
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - normalize-optional-csv-columns-in-a-single-pass.md
  - collect-expected-parse-failures-without-stopping-a-batch.md
  - parse-pipe-delimited-tables-with-continuation-rows.md
---

# Validate Parsed CSV Rows with Bounded Structured Problems

## Idea and Problem

Validate already parsed CSV rows against one fixed schema while retaining only a bounded prefix of structured, value-free problem records.

Configuration is checked before the row source is touched. A correct header
establishes field positions; a bad or missing header ends validation because
later cells cannot be assigned safely. Data rows with the wrong width receive
one shape problem and skip field and uniqueness checks. Valid-width rows run
trusted predicates in declared order, followed by an optional exact composite
uniqueness check.

## When to Use

Use this recipe after `csv.reader` or another reviewed parser has already
produced finite sequences of strings. It fits bounded imports where callers
need stable row numbers and machine-readable problem codes without putting raw
cells or complete records into diagnostics.

Predicates must be deterministic, side-effect-free functions that return an
exact `bool`. Treat a raised exception or another return type as a programming
or configuration error, not as invalid input. Normalize values in a separate
step before validation when whitespace, aliases, or alternate spellings are
accepted.

## Implementation

```python
import re
from collections.abc import Iterable, Sequence
from dataclasses import dataclass
from typing import Protocol


_MAX_COLUMNS = 128
_MAX_RULES = 128
_MAX_UNIQUE_FIELDS = 16
_MAX_PHYSICAL_ROWS = 100_000
_MAX_CELL_BYTES = 16_384
_MAX_RETAINED_PROBLEMS = 10_000
_MAX_UNIQUE_KEYS = 50_000
_MAX_UNIQUE_KEY_BYTES = 64_000_000
_PROBLEM_CODE = re.compile(r"[a-z][a-z0-9_]{0,63}", re.ASCII)
_RESERVED_CODES = frozenset(
    {"bad_header", "missing_header", "wrong_width"}
)


class CsvValuePredicate(Protocol):
    def __call__(self, value: str, /) -> bool: ...


class CsvRecordPredicate(Protocol):
    def __call__(self, values: tuple[str, ...], /) -> bool: ...


@dataclass(frozen=True, slots=True)
class CsvValueRule:
    code: str
    field: str
    predicate: CsvValuePredicate


@dataclass(frozen=True, slots=True)
class CsvRecordRule:
    code: str
    fields: tuple[str, ...]
    predicate: CsvRecordPredicate


@dataclass(frozen=True, slots=True)
class CsvUniqueRule:
    code: str
    fields: tuple[str, ...]


@dataclass(frozen=True, slots=True)
class CsvProblem:
    row_number: int
    code: str
    fields: tuple[str, ...] = ()


@dataclass(frozen=True, slots=True)
class CsvValidationReport:
    physical_rows: int
    data_rows: int
    problem_count: int
    problems: tuple[CsvProblem, ...]
    truncated: bool


def _field_name(value: object) -> str:
    if type(value) is not str:
        raise TypeError("field names must be exact strings")
    if not value or value != value.strip() or not value.isprintable():
        raise ValueError("field names must be stripped printable text")
    if len(value.encode("utf-8")) > 128:
        raise ValueError("a field name exceeds the supported byte limit")
    return value


def _problem_code(value: object) -> str:
    if type(value) is not str:
        raise TypeError("problem codes must be exact strings")
    if _PROBLEM_CODE.fullmatch(value) is None or value in _RESERVED_CODES:
        raise ValueError("a problem code is invalid or reserved")
    return value


def _rule_fields(
    values: object,
    *,
    known_fields: frozenset[str],
) -> tuple[str, ...]:
    if type(values) is not tuple or not values:
        raise TypeError("rule fields must be a non-empty tuple")
    fields = tuple(_field_name(value) for value in values)
    if len(fields) > _MAX_UNIQUE_FIELDS:
        raise ValueError("a rule references too many fields")
    if len(set(fields)) != len(fields):
        raise ValueError("fields within a rule must be unique")
    if not set(fields).issubset(known_fields):
        raise ValueError("every rule field must exist in the expected header")
    return fields


def _validated_row(row: object) -> tuple[str, ...]:
    if type(row) not in (list, tuple):
        raise TypeError("parsed rows must be lists or tuples")
    if len(row) > _MAX_COLUMNS:
        raise ValueError("a parsed row has too many columns")
    values = tuple(row)
    for value in values:
        if type(value) is not str:
            raise TypeError("parsed cells must be exact strings")
        try:
            encoded = value.encode("utf-8")
        except UnicodeEncodeError as error:
            raise ValueError("a parsed cell is not valid UTF-8 text") from error
        if len(encoded) > _MAX_CELL_BYTES:
            raise ValueError("a parsed cell exceeds the supported byte limit")
    return values


def validate_parsed_csv_rows(
    rows: Iterable[Sequence[str]],
    *,
    expected_header: tuple[str, ...],
    value_rules: tuple[CsvValueRule, ...] = (),
    record_rules: tuple[CsvRecordRule, ...] = (),
    unique_rule: CsvUniqueRule | None = None,
    expect_header: bool = True,
    max_physical_rows: int = 10_000,
    max_problems: int = 100,
    max_unique_keys: int = 10_000,
    max_unique_key_bytes: int = 8_000_000,
) -> CsvValidationReport:
    if type(expected_header) is not tuple or not expected_header:
        raise TypeError("expected_header must be a non-empty tuple")
    if len(expected_header) > _MAX_COLUMNS:
        raise ValueError("expected_header has too many columns")
    header = tuple(_field_name(field) for field in expected_header)
    if len(set(header)) != len(header):
        raise ValueError("expected_header fields must be unique")
    known_fields = frozenset(header)

    if type(value_rules) is not tuple or type(record_rules) is not tuple:
        raise TypeError("rule collections must be tuples")
    if len(value_rules) + len(record_rules) > _MAX_RULES:
        raise ValueError("too many validation rules")

    validated_value_rules = []
    validated_record_rules = []
    codes = set()
    for rule in value_rules:
        if type(rule) is not CsvValueRule:
            raise TypeError("value_rules must contain CsvValueRule values")
        code = _problem_code(rule.code)
        field = _field_name(rule.field)
        if field not in known_fields:
            raise ValueError("a value rule references an unknown field")
        if not callable(rule.predicate):
            raise TypeError("a value predicate is not callable")
        if code in codes:
            raise ValueError("problem codes must be unique")
        codes.add(code)
        validated_value_rules.append((code, field, rule.predicate))

    for rule in record_rules:
        if type(rule) is not CsvRecordRule:
            raise TypeError("record_rules must contain CsvRecordRule values")
        code = _problem_code(rule.code)
        fields = _rule_fields(rule.fields, known_fields=known_fields)
        if not callable(rule.predicate):
            raise TypeError("a record predicate is not callable")
        if code in codes:
            raise ValueError("problem codes must be unique")
        codes.add(code)
        validated_record_rules.append((code, fields, rule.predicate))

    validated_unique_rule = None
    if unique_rule is not None:
        if type(unique_rule) is not CsvUniqueRule:
            raise TypeError("unique_rule must be a CsvUniqueRule or None")
        code = _problem_code(unique_rule.code)
        fields = _rule_fields(unique_rule.fields, known_fields=known_fields)
        if code in codes:
            raise ValueError("problem codes must be unique")
        codes.add(code)
        validated_unique_rule = (code, fields)

    if type(expect_header) is not bool:
        raise TypeError("expect_header must be a bool")
    if isinstance(max_physical_rows, bool) or not isinstance(
        max_physical_rows, int
    ):
        raise TypeError("max_physical_rows must be an integer")
    if not 1 <= max_physical_rows <= _MAX_PHYSICAL_ROWS:
        raise ValueError("max_physical_rows is outside the supported range")
    if isinstance(max_problems, bool) or not isinstance(max_problems, int):
        raise TypeError("max_problems must be an integer")
    if not 0 <= max_problems <= _MAX_RETAINED_PROBLEMS:
        raise ValueError("max_problems is outside the supported range")
    if type(max_unique_keys) is not int:
        raise TypeError("max_unique_keys must be an exact integer")
    if not 0 <= max_unique_keys <= _MAX_UNIQUE_KEYS:
        raise ValueError("max_unique_keys is outside the supported range")
    if type(max_unique_key_bytes) is not int:
        raise TypeError("max_unique_key_bytes must be an exact integer")
    if not 0 <= max_unique_key_bytes <= _MAX_UNIQUE_KEY_BYTES:
        raise ValueError("max_unique_key_bytes is outside the supported range")
    if isinstance(rows, (str, bytes)):
        raise TypeError("rows must be a non-text iterable")
    try:
        iterator = iter(rows)
    except TypeError as error:
        raise TypeError("rows must be iterable") from error

    indexes = {field: index for index, field in enumerate(header)}
    retained: list[CsvProblem] = []
    problem_count = 0
    physical_rows = 0
    data_rows = 0
    seen_keys: set[tuple[str, ...]] = set()
    unique_key_bytes = 0

    def add_problem(problem: CsvProblem) -> None:
        nonlocal problem_count
        problem_count += 1
        if len(retained) < max_problems:
            retained.append(problem)

    for physical_rows, raw_row in enumerate(iterator, start=1):
        if physical_rows > max_physical_rows:
            raise ValueError("rows exceed the supported physical-row limit")
        row = _validated_row(raw_row)

        if physical_rows == 1 and expect_header:
            if row != header:
                add_problem(CsvProblem(1, "bad_header"))
                break
            continue

        data_rows += 1
        if len(row) != len(header):
            add_problem(CsvProblem(physical_rows, "wrong_width"))
            continue

        for code, field, predicate in validated_value_rules:
            accepted = predicate(row[indexes[field]])
            if type(accepted) is not bool:
                raise TypeError("a value predicate must return an exact bool")
            if not accepted:
                add_problem(CsvProblem(physical_rows, code, (field,)))

        for code, fields, predicate in validated_record_rules:
            selected = tuple(row[indexes[field]] for field in fields)
            accepted = predicate(selected)
            if type(accepted) is not bool:
                raise TypeError("a record predicate must return an exact bool")
            if not accepted:
                add_problem(CsvProblem(physical_rows, code, fields))

        if validated_unique_rule is not None:
            code, fields = validated_unique_rule
            key = tuple(row[indexes[field]] for field in fields)
            if key in seen_keys:
                add_problem(CsvProblem(physical_rows, code, fields))
            else:
                key_bytes = sum(len(value.encode("utf-8")) for value in key)
                if len(seen_keys) >= max_unique_keys:
                    raise ValueError("unique keys exceed max_unique_keys")
                if key_bytes > max_unique_key_bytes - unique_key_bytes:
                    raise ValueError(
                        "unique key bytes exceed max_unique_key_bytes"
                    )
                seen_keys.add(key)
                unique_key_bytes += key_bytes

    if expect_header and physical_rows == 0:
        add_problem(CsvProblem(1, "missing_header"))

    return CsvValidationReport(
        physical_rows=physical_rows,
        data_rows=data_rows,
        problem_count=problem_count,
        problems=tuple(retained),
        truncated=problem_count > len(retained),
    )
```

## Example

```python
rows = [
    ["record_id", "email", "region"],
    ["1", "first@example.test", "north"],
    ["1", "not-an-email", "south"],
    ["2", "missing-region@example.test"],
]
report = validate_parsed_csv_rows(
    rows,
    expected_header=("record_id", "email", "region"),
    value_rules=(
        CsvValueRule(
            "invalid_email",
            "email",
            lambda value: "@" in value,
        ),
    ),
    record_rules=(
        CsvRecordRule(
            "empty_region",
            ("region",),
            lambda values: bool(values[0]),
        ),
    ),
    unique_rule=CsvUniqueRule("duplicate_id", ("record_id",)),
    max_problems=2,
)

try:
    validate_parsed_csv_rows(
        [("record_id",), ("1234",)],
        expected_header=("record_id",),
        unique_rule=CsvUniqueRule("duplicate_id", ("record_id",)),
        max_unique_key_bytes=3,
    )
except ValueError:
    unique_budget_rejected = True
else:
    unique_budget_rejected = False

assert (report, unique_budget_rejected) == (
    CsvValidationReport(
        physical_rows=4,
        data_rows=3,
        problem_count=3,
        problems=(
            CsvProblem(3, "invalid_email", ("email",)),
            CsvProblem(3, "duplicate_id", ("record_id",)),
        ),
        truncated=True,
    ),
    True,
)
```

## Trade-offs and Limitations

The function consumes rows once and does not retain valid records, but an exact
uniqueness rule stores one composite string tuple per distinct valid-width
row. Its key count and cumulative UTF-8 payload have separate explicit limits;
the count also bounds tuple, string-reference, and set overhead that byte
accounting does not include. Work is `O(rows * rules)`, and a malicious or
expensive trusted predicate can still dominate runtime. Reaching a row or
uniqueness limit raises after earlier rows and predicate calls have already
been observed.

Stopping after a bad header avoids assigning cells to the wrong fields, but it
also means later physical rows are not inspected. Wrong-width rows intentionally
produce only one structural problem. Retained problems are a bounded prefix;
`problem_count` still counts every issue found before completion. The helper
does not parse CSV syntax, normalize data, infer types, expose raw rejected
values, repair records, persist a report, or make untrusted callback code safe.

## Related Snippets

<!-- catalog:related:start -->
- [Normalize Optional CSV Columns in a Single Pass](normalize-optional-csv-columns-in-a-single-pass.md)
- [Collect Expected Parse Failures Without Stopping a Batch](collect-expected-parse-failures-without-stopping-a-batch.md)
- [Parse Pipe-Delimited Tables with Continuation Rows](parse-pipe-delimited-tables-with-continuation-rows.md)
<!-- catalog:related:end -->
