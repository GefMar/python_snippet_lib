---
title: "Compile a Bounded T-String into SQLite Qmark SQL and Parameters"
snippet_type: pattern
use_cases:
  - persistence
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/parse-a-bounded-flat-placeholder-template.md
  - plan-bounded-parameterized-backfill-statements-by-date-window.md
  - scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md
---

# Compile a Bounded T-String into SQLite Qmark SQL and Parameters

## Idea and Problem

Turn a trusted Python t-string into SQLite SQL whose interpolated values are always separate positional parameters.

A t-string preserves its static strings and interpolation objects instead of
formatting them immediately. Replacing each interpolation with one qmark
placeholder creates a small boundary between trusted query structure and
runtime values. Rejecting literal question marks makes the correspondence
unambiguous: every `?` in the result has exactly one tuple entry.

## When to Use

Use this pattern with Python 3.14 or newer when application code owns a fixed
SQLite statement but its values arrive at runtime. The accepted scalar set is
closed to the primitive values handled directly by SQLite's DB-API binding,
and explicit byte and item budgets keep the prepared result small.

Use a query builder when identifiers, optional clauses, variable-length
`IN` lists, or another SQL dialect must be assembled structurally. Keep using
the database driver's parameter mechanism; never interpolate the returned
parameters into SQL text yourself.

## Implementation

```python
import math
import sqlite3
from dataclasses import dataclass
from string.templatelib import Interpolation, Template

_MAX_INTERPOLATIONS = 64
_MAX_STATIC_CHARACTERS = 8_192
_MAX_STATIC_BYTES = 16_384
_MAX_SQL_CHARACTERS = 8_192
_MAX_VALUE_BYTES = 4_096
_MAX_PARAMETER_PAYLOAD = 65_536
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class PreparedStatement:
    sql: str
    parameters: tuple[object, ...]


def _encode_text(value: str, *, name: str) -> bytes:
    try:
        return value.encode("utf-8")
    except UnicodeEncodeError as error:
        raise ValueError(f"{name} must not contain Unicode surrogates") from error


def _validate_parameter(value: object) -> tuple[object, int]:
    if value is None:
        return None, 0
    if type(value) is int:
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError("integer parameter is outside the signed 64-bit range")
        return value, 8
    if type(value) is float:
        if not math.isfinite(value):
            raise ValueError("float parameter must be finite")
        return value, 8
    if type(value) is str:
        if len(value) > _MAX_VALUE_BYTES:
            raise ValueError("string parameter exceeds the supported byte count")
        encoded = _encode_text(value, name="string parameter")
        if len(encoded) > _MAX_VALUE_BYTES:
            raise ValueError("string parameter exceeds the supported byte count")
        return value, len(encoded)
    if type(value) is bytes:
        if len(value) > _MAX_VALUE_BYTES:
            raise ValueError("bytes parameter exceeds the supported byte count")
        return value, len(value)
    raise TypeError("interpolations must contain supported exact scalar values")


def compile_sqlite_template(template: Template) -> PreparedStatement:
    if type(template) is not Template:
        raise TypeError("template must be an exact string.templatelib.Template")

    sql_parts: list[str] = []
    parameters: list[object] = []
    static_characters = 0
    static_bytes = 0
    parameter_payload = 0

    for part in template:
        if type(part) is str:
            static_characters += len(part)
            if static_characters > _MAX_STATIC_CHARACTERS:
                raise ValueError("static SQL exceeds the supported character count")
            encoded = _encode_text(part, name="static SQL")
            if "\x00" in part:
                raise ValueError("static SQL must not contain NUL")
            if "?" in part:
                raise ValueError("literal qmark placeholders are not supported")
            static_bytes += len(encoded)
            if static_bytes > _MAX_STATIC_BYTES:
                raise ValueError("static SQL exceeds the supported UTF-8 byte count")
            sql_parts.append(part)
            continue

        if type(part) is not Interpolation:
            raise TypeError("template contains an unsupported exact part type")
        if len(parameters) >= _MAX_INTERPOLATIONS:
            raise ValueError("template exceeds the supported interpolation count")
        if part.conversion is not None:
            raise ValueError("interpolation conversions are not supported")
        if part.format_spec != "":
            raise ValueError("interpolation format specifications are not supported")

        parameter, payload_size = _validate_parameter(part.value)
        parameter_payload += payload_size
        if parameter_payload > _MAX_PARAMETER_PAYLOAD:
            raise ValueError("parameter payload exceeds the supported byte count")
        sql_parts.append("?")
        parameters.append(parameter)

    sql = "".join(sql_parts)
    if not sql:
        raise ValueError("compiled SQL must not be empty")
    if len(sql) > _MAX_SQL_CHARACTERS:
        raise ValueError("compiled SQL exceeds the supported character count")
    return PreparedStatement(sql, tuple(parameters))
```

## Example

```python
connection = sqlite3.connect(":memory:")
connection.execute(
    "CREATE TABLE account (name TEXT PRIMARY KEY, active INTEGER NOT NULL)"
)
connection.executemany(
    "INSERT INTO account VALUES (?, ?)",
    (("Ada", 1), ("Grace", 0)),
)

untrusted_name = "Ada' OR 1 = 1 --"
active = 1
statement = compile_sqlite_template(
    t"SELECT name FROM account WHERE name = {untrusted_name} AND active = {active}"
)
rows = connection.execute(statement.sql, statement.parameters).fetchall()

safe_statement = compile_sqlite_template(
    t"SELECT name FROM account WHERE name = {'Ada'} AND active = {active}"
)
safe_rows = connection.execute(
    safe_statement.sql,
    safe_statement.parameters,
).fetchall()
connection.close()

assert statement.sql == "SELECT name FROM account WHERE name = ? AND active = ?"
assert statement.parameters == (untrusted_name, 1)
assert rows == []
assert safe_rows == [("Ada",)]
```

## Trade-offs and Limitations

For `S` static characters, `n` interpolations, and `P` bytes of admitted
parameter payload, compilation uses `O(S + n + P)` time and state. The fixed
limits keep every term bounded before the immutable result is returned.

The template and all of its static SQL must originate in trusted code. Python
evaluates interpolation expressions while creating the t-string, before this
helper receives the `Template`, so this boundary cannot make an unsafe
expression inert. It only prevents admitted interpolation values from becoming
SQL structure.

A qmark represents one value, never an identifier, keyword, operator, clause,
or list of placeholders. The helper does not validate SQL grammar, execute a
statement, own a connection or transaction, invoke custom adapters, or support
other parameter styles. A dynamically constructed static template remains
unsafe even if its value interpolations are later bound correctly.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Flat Placeholder Template](../configuration-serialization/parse-a-bounded-flat-placeholder-template.md)
- [Plan Bounded Parameterized Backfill Statements by Date Window](plan-bounded-parameterized-backfill-statements-by-date-window.md)
- [Scope Caller-Owned SQLite Work with an Explicit Savepoint](scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md)
<!-- catalog:related:end -->
