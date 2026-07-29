---
title: "Group Bounded SQLite Console Lines into Lexically Complete Batches Without Execution"
snippet_type: recipe
use_cases:
  - parsing
  - validation
  - automation
tested_python:
  - "3.14"
dependencies: []
related:
  - compile-a-bounded-t-string-into-sqlite-qmark-sql-and-parameters.md
  - scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md
  - page-bounded-sqlite-rows-with-a-composite-keyset-cursor.md
---

# Group Bounded SQLite Console Lines into Lexically Complete Batches Without Execution

## Idea and Problem

Collect bounded SQLite console lines until each buffer is lexically complete, preserving completed batches and an unfinished tail without opening a database or executing SQL.

The result preserves every completed batch and one optional unfinished tail.
Lexical completeness is deliberately weaker than SQL validity: `nonsense;` is
complete, while a valid prefix without its terminating semicolon is not.

## When to Use

Use this for a bounded interactive SQLite prompt, migration-file editor,
syntax-aware input collector, or test fixture that needs to decide when to hand
text to a separate trusted execution layer. It is appropriate when newline
insertion between caller-provided physical lines is the intended reconstruction
rule.

Do not use it to split arbitrary SQL into individual statements, validate a
schema, bind parameters, or enforce transaction policy. A completed batch may
contain multiple statements or invalid SQLite syntax. Keep execution and
authorization in a later boundary with explicit connection ownership.

## Implementation

```python
from dataclasses import dataclass
from sqlite3 import complete_statement

_MAX_SQL_LINES = 4_096
_MAX_SQL_LINE_CHARACTERS = 4_096
_MAX_SQL_LINE_BYTES = 4_096
_MAX_SQL_TEXT_BYTES = 131_072
_MAX_SQL_BATCHES = 4_096


class SqlConsoleLimitError(ValueError):
    """Raised when console lines violate the bounded lexical profile."""


@dataclass(frozen=True, slots=True)
class SqlConsoleBatches:
    complete: tuple[str, ...]
    incomplete_tail: str | None


def _validate_sql_console_lines(lines: tuple[str, ...]) -> None:
    if type(lines) is not tuple:
        raise TypeError("lines must be an exact tuple")
    if len(lines) > _MAX_SQL_LINES:
        raise SqlConsoleLimitError("too many SQL console lines")

    total_bytes = 0
    for index, line in enumerate(lines):
        if type(line) is not str:
            raise TypeError(f"lines[{index}] must be an exact str")
        if any(character in line for character in ("\r", "\n", "\0")):
            raise SqlConsoleLimitError(
                f"lines[{index}] must contain one physical line"
            )
        if len(line) > _MAX_SQL_LINE_CHARACTERS:
            raise SqlConsoleLimitError(f"lines[{index}] is too long")
        try:
            line_bytes = len(line.encode("utf-8"))
        except UnicodeEncodeError as exc:
            raise SqlConsoleLimitError(
                f"lines[{index}] must be valid UTF-8 text"
            ) from exc
        if line_bytes > _MAX_SQL_LINE_BYTES:
            raise SqlConsoleLimitError(f"lines[{index}] is too large")
        total_bytes += line_bytes
        if total_bytes > _MAX_SQL_TEXT_BYTES:
            raise SqlConsoleLimitError("SQL console text exceeds the byte limit")


def group_sqlite_console_lines(lines: tuple[str, ...]) -> SqlConsoleBatches:
    """Group physical lines into lexical SQLite batches without execution."""
    _validate_sql_console_lines(lines)

    complete: list[str] = []
    pending: list[str] = []
    for line in lines:
        pending.append(line)
        candidate = "\n".join(pending)
        if complete_statement(candidate):
            complete.append(candidate)
            pending.clear()
            if len(complete) > _MAX_SQL_BATCHES:
                raise SqlConsoleLimitError("too many complete SQL batches")

    return SqlConsoleBatches(
        complete=tuple(complete),
        incomplete_tail="\n".join(pending) if pending else None,
    )
```

## Example

```python
trigger_lines = (
    "CREATE TRIGGER audit_insert AFTER INSERT ON items",
    "BEGIN",
    "  INSERT INTO audit_log(message) VALUES ('created;still quoted');",
    "  UPDATE counters SET value = value + 1;",
    "END;",
)
input_lines = (
    *trigger_lines,
    "nonsense;",
    "SELECT 1; SELECT 2;",
    "SELECT 'unfinished",
)
grouped = group_sqlite_console_lines(input_lines)

quoted = group_sqlite_console_lines(("SELECT ';' AS marker;",))
empty = group_sqlite_console_lines(())
batch_boundary = group_sqlite_console_lines((";",) * 4_096)
line_boundary = group_sqlite_console_lines((" " * 4_096,))
utf8_line_boundary = group_sqlite_console_lines(("é" * 2_048,))
aggregate_boundary = group_sqlite_console_lines((" " * 4_096,) * 32)

invalid = 0
for candidate in (
    (" " * 4_097,),
    ("é" * 2_049,),
    ("SELECT\n1;",),
    ("SELECT\r1;",),
    ("SELECT\0;",),
    ("\ud800",),
    (";",) * 4_097,
    (" " * 4_096,) * 33,
):
    try:
        group_sqlite_console_lines(candidate)
    except SqlConsoleLimitError:
        invalid += 1

assert (
    grouped.complete
    == (
        "\n".join(trigger_lines),
        "nonsense;",
        "SELECT 1; SELECT 2;",
    )
    and grouped.incomplete_tail == "SELECT 'unfinished"
    and quoted.complete == ("SELECT ';' AS marker;",)
    and quoted.incomplete_tail is None
    and empty == SqlConsoleBatches((), None)
    and len(batch_boundary.complete) == 4_096
    and line_boundary.incomplete_tail == " " * 4_096
    and utf8_line_boundary.incomplete_tail == "é" * 2_048
    and len(aggregate_boundary.incomplete_tail or "") == 131_103
    and invalid == 8
)
```

## Trade-offs and Limitations

`complete_statement` answers only whether SQLite sees a terminated lexical
unit. It does not parse names, validate grammar, check tables, prepare SQL, bind
values, open a connection, or execute anything. One completed batch may contain
several statements; comments and whitespace are retained exactly, and the
helper inserts one LF only between declared input lines.

Each new line causes the current batch to be joined and scanned again. A long
unfinished batch can therefore require quadratic work in its accumulated text
length. The line, per-line, aggregate-byte, and batch limits make that cost
finite but do not make it linear. Completion rules belong to SQLite and should
be rechecked when the supported SQLite/Python build changes.

## Related Snippets

<!-- catalog:related:start -->
- [Compile a Bounded T-String into SQLite Qmark SQL and Parameters](compile-a-bounded-t-string-into-sqlite-qmark-sql-and-parameters.md)
- [Scope Caller-Owned SQLite Work with an Explicit Savepoint](scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md)
- [Page Bounded SQLite Rows with a Composite Keyset Cursor](page-bounded-sqlite-rows-with-a-composite-keyset-cursor.md)
<!-- catalog:related:end -->
