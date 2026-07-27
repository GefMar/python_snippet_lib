---
title: "Read Bounded CSV Text into pandas Under an Explicit Schema"
snippet_type: integration
use_cases:
  - parsing
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: numpy
    version: "2.5.1"
  - name: pandas
    version: "3.0.5"
related:
  - validate-parsed-csv-rows-with-bounded-structured-problems.md
  - normalize-optional-csv-columns-in-a-single-pass.md
  - audit-pandas-missing-value-shares-against-column-policies.md
---

# Read Bounded CSV Text into pandas Under an Explicit Schema

## Idea and Problem

Parse size-limited in-memory CSV text, validate its complete rectangular shape, and construct a pandas DataFrame from one immutable closed schema.

The standard-library CSV parser handles quoting and embedded newlines before
pandas is involved. Every logical record, column, cell, and cell length is
bounded first. The header must contain each declared source name exactly once
and no other name. Columns are then reordered and optionally renamed according
to the schema, so the result does not depend on the input header order.

Empty fields are the sole missing-value spelling. They become `pd.NA` in one
of four nullable pandas dtypes: Python-backed `string`, `Int64`, `Float64`, or
`boolean`. Critical columns do not cause rows to be dropped; instead, the
result reports a bounded prefix of value-free record-and-column locations.

## When to Use

Use this integration when a bounded CSV payload is already in memory and its
transport has established the text encoding. Define the entire accepted
header, output names, dtypes, and critical columns before reading the payload.
Integer cells use signed decimal syntax, float cells use finite ASCII decimal
or exponent syntax, and boolean cells are exactly `true` or `false`.

This is intentionally a structural ingestion boundary. It does not open paths,
infer dates, accept user callbacks, validate business rules, drop incomplete
rows, or preserve an undeclared column. Apply domain validation after checking
the returned critical-null report.

## Implementation

```python
import csv
import io
import math
import re
from dataclasses import dataclass

import numpy as np
import pandas as pd


_MAX_CSV_CHARACTERS = 4_000_000
_MAX_CSV_UTF8_BYTES = 8_000_000
_MAX_LOGICAL_RECORDS = 100_001
_MAX_COLUMNS = 128
_MAX_CELLS = 1_000_000
_MAX_CELL_CHARACTERS = 16_384
_MAX_NAME_CHARACTERS = 128
_MAX_REPORTED_CRITICAL_NULLS = 1_000
_INT64_MIN = int(np.iinfo(np.int64).min)
_INT64_MAX = int(np.iinfo(np.int64).max)
_INTEGER = re.compile(r"[+-]?[0-9]+", re.ASCII)
_FLOAT = re.compile(
    r"[+-]?(?:(?:[0-9]+(?:\.[0-9]*)?)|(?:\.[0-9]+))"
    r"(?:[eE][+-]?[0-9]+)?",
    re.ASCII,
)
_DTYPES = frozenset({"string", "Int64", "Float64", "boolean"})


@dataclass(frozen=True, slots=True)
class CsvColumn:
    source_name: str
    output_name: str
    dtype: str
    critical: bool = False


@dataclass(frozen=True, slots=True)
class CriticalNullLocation:
    record_number: int
    column: str


@dataclass(frozen=True, slots=True)
class CsvFrameRead:
    frame: pd.DataFrame
    critical_null_count: int
    critical_null_locations: tuple[CriticalNullLocation, ...]
    critical_null_locations_truncated: bool


def _bounded_name(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not value or len(value) > _MAX_NAME_CHARACTERS:
        raise ValueError(f"{field} length is outside the supported range")
    if value != value.strip() or not value.isprintable():
        raise ValueError(f"{field} must be stripped printable text")
    return value


def _validated_schema(value: object) -> tuple[CsvColumn, ...]:
    if type(value) is not tuple or not value:
        raise TypeError("schema must be a non-empty exact tuple")
    if len(value) > _MAX_COLUMNS:
        raise ValueError("schema contains too many columns")

    columns: list[CsvColumn] = []
    for column in value:
        if type(column) is not CsvColumn:
            raise TypeError("schema must contain exact CsvColumn values")
        source = _bounded_name(column.source_name, field="source name")
        output = _bounded_name(column.output_name, field="output name")
        if type(column.dtype) is not str or column.dtype not in _DTYPES:
            raise ValueError("schema dtype is not in the closed supported set")
        if type(column.critical) is not bool:
            raise TypeError("critical flags must be exact bool values")
        columns.append(CsvColumn(source, output, column.dtype, column.critical))

    if len({column.source_name for column in columns}) != len(columns):
        raise ValueError("schema source names must be unique")
    if len({column.output_name for column in columns}) != len(columns):
        raise ValueError("schema output names must be unique after renaming")
    return tuple(columns)


def _bounded_csv_records(text: object) -> tuple[tuple[str, ...], ...]:
    if type(text) is not str:
        raise TypeError("text must be an exact string")
    if not text:
        raise ValueError("text must not be empty")
    if len(text) > _MAX_CSV_CHARACTERS:
        raise ValueError("text exceeds the supported character limit")
    if "\x00" in text:
        raise ValueError("text must not contain NUL characters")
    try:
        encoded_size = len(text.encode("utf-8"))
    except UnicodeEncodeError as error:
        raise ValueError("text must be valid UTF-8 text") from error
    if encoded_size > _MAX_CSV_UTF8_BYTES:
        raise ValueError("text exceeds the supported UTF-8 byte limit")

    records: list[tuple[str, ...]] = []
    cell_count = 0
    try:
        reader = csv.reader(
            io.StringIO(text, newline=""),
            delimiter=",",
            quotechar='"',
            doublequote=True,
            skipinitialspace=False,
            strict=True,
        )
        for record_number, row in enumerate(reader, start=1):
            if record_number > _MAX_LOGICAL_RECORDS:
                raise ValueError("CSV exceeds the supported logical-record limit")
            if len(row) > _MAX_COLUMNS:
                raise ValueError("a CSV record exceeds the supported column limit")
            if len(row) > _MAX_CELLS - cell_count:
                raise ValueError("CSV exceeds the supported cell limit")
            for cell in row:
                if len(cell) > _MAX_CELL_CHARACTERS:
                    raise ValueError("a CSV cell exceeds the supported character limit")
            cell_count += len(row)
            records.append(tuple(row))
    except csv.Error as error:
        raise ValueError("text is not well-formed CSV") from error

    if not records:
        raise ValueError("CSV must contain a header record")
    return tuple(records)


def _parse_cell(value: str, *, dtype: str) -> object:
    if value == "":
        return pd.NA
    if dtype == "string":
        return value
    if dtype == "boolean":
        if value == "true":
            return True
        if value == "false":
            return False
        raise ValueError("a boolean cell has invalid syntax")
    if dtype == "Int64":
        if _INTEGER.fullmatch(value) is None:
            raise ValueError("an integer cell has invalid syntax")
        digits = value[1:] if value[:1] in {"+", "-"} else value
        significant_digits = digits.lstrip("0") or "0"
        if len(significant_digits) > 19:
            raise ValueError("an integer cell is outside the Int64 range")
        integer = int(significant_digits, 10)
        if value.startswith("-"):
            integer = -integer
        if not _INT64_MIN <= integer <= _INT64_MAX:
            raise ValueError("an integer cell is outside the Int64 range")
        return integer
    if _FLOAT.fullmatch(value) is None:
        raise ValueError("a float cell has invalid syntax")
    number = float(value)
    if not math.isfinite(number):
        raise ValueError("a float cell must be finite")
    return number


def read_bounded_csv_text(
    text: str,
    *,
    schema: tuple[CsvColumn, ...],
) -> CsvFrameRead:
    columns = _validated_schema(schema)
    records = _bounded_csv_records(text)
    raw_header = records[0]
    if not raw_header:
        raise ValueError("CSV header must not be empty")
    header = tuple(
        _bounded_name(name, field="CSV header name") for name in raw_header
    )
    if len(set(header)) != len(header):
        raise ValueError("CSV header names must be unique")

    expected = {column.source_name for column in columns}
    observed = set(header)
    if observed != expected or len(header) != len(columns):
        raise ValueError("CSV header must exactly match schema source names")

    for row in records[1:]:
        if len(row) != len(header):
            raise ValueError("every CSV data record must match the header width")

    indexes = {name: index for index, name in enumerate(header)}
    converted: dict[str, pd.api.extensions.ExtensionArray] = {}
    critical_count = 0
    retained: list[CriticalNullLocation] = []
    for record_number, row in enumerate(records[1:], start=2):
        for column in columns:
            if column.critical and row[indexes[column.source_name]] == "":
                critical_count += 1
                if len(retained) < _MAX_REPORTED_CRITICAL_NULLS:
                    retained.append(
                        CriticalNullLocation(record_number, column.output_name)
                    )

    for column in columns:
        raw_values = tuple(row[indexes[column.source_name]] for row in records[1:])
        parsed = tuple(_parse_cell(value, dtype=column.dtype) for value in raw_values)

        dtype: object
        if column.dtype == "string":
            dtype = pd.StringDtype(storage="python", na_value=pd.NA)
        elif column.dtype == "Int64":
            dtype = pd.Int64Dtype()
        elif column.dtype == "Float64":
            dtype = pd.Float64Dtype()
        else:
            dtype = pd.BooleanDtype()
        converted[column.output_name] = pd.array(parsed, dtype=dtype)

    frame = pd.DataFrame(
        converted,
        index=pd.RangeIndex(len(records) - 1),
        copy=True,
    )
    return CsvFrameRead(
        frame=frame,
        critical_null_count=critical_count,
        critical_null_locations=tuple(retained),
        critical_null_locations_truncated=(
            critical_count > len(retained)
        ),
    )
```

## Example

```python
payload = (
    "enabled,amount,record_id,note\n"
    "true,+00000000000000000010,item-1,ready\n"
    "false,,item-2,\n"
)
schema = (
    CsvColumn("record_id", "record_id", "string", True),
    CsvColumn("note", "comment", "string", True),
    CsvColumn("amount", "amount", "Int64", True),
    CsvColumn("enabled", "enabled", "boolean"),
)
result = read_bounded_csv_text(payload, schema=schema)

try:
    read_bounded_csv_text('record_id\n"unterminated', schema=schema[:1])
except ValueError:
    malformed_rejected = True
else:
    malformed_rejected = False

amounts = result.frame["amount"].tolist()
comments = result.frame["comment"].tolist()
assert (
    result.frame.columns.tolist(),
    result.frame["record_id"].tolist(),
    amounts[0],
    pd.isna(amounts[1]),
    comments[0],
    pd.isna(comments[1]),
    tuple(str(dtype) for dtype in result.frame.dtypes),
    result.critical_null_count,
    result.critical_null_locations,
    result.critical_null_locations_truncated,
    malformed_rejected,
    payload.endswith("\n"),
) == (
    ["record_id", "comment", "amount", "enabled"],
    ["item-1", "item-2"],
    10,
    True,
    "ready",
    True,
    ("string", "string", "Int64", "boolean"),
    2,
    (
        CriticalNullLocation(3, "comment"),
        CriticalNullLocation(3, "amount"),
    ),
    False,
    True,
    True,
)
```

## Trade-offs and Limitations

The preflight retains the complete bounded CSV as tuples and conversion creates
another column-oriented representation before the DataFrame is returned. Peak
memory is therefore greater than the payload plus final frame, and quoted
newlines make logical record numbers different from physical line numbers.
The fixed limits bound counts and cell text, but they do not estimate every
Python or pandas object overhead byte.

Parsing is deliberately narrow: whitespace is data, an empty field is missing,
and there are no alternate null, integer, float, or boolean spellings. Nullable
dtypes preserve every input record but do not make a critical null acceptable
to a downstream domain. Error messages and null locations never include cell
values. The returned DataFrame is a caller-owned copy and is mutable; the
schema and retained location tuple are immutable. Concurrent mutation is not
relevant for exact string and frozen schema inputs.

## Related Snippets

<!-- catalog:related:start -->
- [Validate Parsed CSV Rows with Bounded Structured Problems](validate-parsed-csv-rows-with-bounded-structured-problems.md)
- [Normalize Optional CSV Columns in a Single Pass](normalize-optional-csv-columns-in-a-single-pass.md)
- [Audit pandas Missing-Value Shares Against Column Policies](audit-pandas-missing-value-shares-against-column-policies.md)
<!-- catalog:related:end -->
