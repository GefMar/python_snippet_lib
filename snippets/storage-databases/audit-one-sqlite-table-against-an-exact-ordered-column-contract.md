---
title: "Audit One SQLite Table Against an Exact Ordered Column Contract"
snippet_type: testing-technique
use_cases:
  - persistence
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - plan-an-additive-sqlite-column-projection.md
  - plan-bounded-table-initialization-and-ordered-row-batches.md
  - open-a-verified-read-only-sqlite-connection-under-a-closed-hardening-profile.md
---

# Audit One SQLite Table Against an Exact Ordered Column Contract

## Idea and Problem

Compare one live SQLite main-schema table with an explicit ordered lexical column contract without constructing SQL from identifiers or changing the database.

SQLite's `table_xinfo` metadata includes ordinary, generated, and hidden
columns. Its table-valued form accepts the table name as a bound parameter, so
one statement can distinguish a missing table name from another schema-object
kind and read every column without interpolating an identifier. Ordering by
SQLite's `cid` exposes the declared column sequence.

The audit deliberately compares the returned type and default-expression text
exactly. A mismatch returns the complete bounded observed tuple so a caller can
review the drift instead of receiving only a Boolean or a partially collected
schema.

## When to Use

Use this audit for an in-memory fixture, a migration verification step, or a
startup compatibility check when one component depends on a small exact SQLite
column layout. It is especially useful when column order, generated-column
kind, nullability, or composite-primary-key position is part of the adapter's
contract.

Use a migration framework or a separately reviewed schema inspector when
indexes, foreign keys, checks, triggers, table options, or generated expressions
also matter. Run the audit in the same transaction or snapshot as dependent
work when concurrent schema changes are possible; a successful result is only
an observation, not a lock.

## Implementation

```python
import re
import sqlite3
from dataclasses import dataclass
from enum import StrEnum


_MAX_SQLITE_TABLE_COLUMNS = 64
_MAX_SQLITE_TABLE_NAME_BYTES = 63
_MAX_SQLITE_COLUMN_NAME_BYTES = 128
_MAX_SQLITE_DECLARED_TYPE_BYTES = 128
_MAX_SQLITE_DEFAULT_SQL_BYTES = 1_024
_SQLITE_TABLE_NAME = re.compile(r"[A-Za-z_][A-Za-z0-9_]{0,62}", re.ASCII)
_SQLITE_SCHEMA_ALIASES = frozenset(("sqlite_master", "sqlite_schema"))
_SQLITE_HIDDEN_KINDS = frozenset((0, 1, 2, 3))
_TABLE_XINFO_QUERY = """
SELECT
    schema_object.type,
    column_info.cid,
    column_info.name,
    column_info.type,
    column_info."notnull",
    column_info.dflt_value,
    column_info.pk,
    column_info.hidden
FROM (
    SELECT CAST(type AS TEXT) AS type
    FROM main.sqlite_schema
    WHERE name = ? COLLATE BINARY
    ORDER BY CASE type WHEN 'table' THEN 0 ELSE 1 END, type
    LIMIT 1
) AS schema_object
LEFT JOIN pragma_table_xinfo(?, 'main') AS column_info
    ON schema_object.type = 'table'
ORDER BY column_info.cid
"""


@dataclass(frozen=True, slots=True)
class SQLiteColumnContract:
    name: str
    declared_type: str
    not_null: bool
    default_sql: str | None
    primary_key_ordinal: int
    hidden_kind: int


class SQLiteTableAuditStatus(StrEnum):
    MATCH = "match"
    MISSING = "missing"
    NOT_A_TABLE = "not-a-table"
    MISMATCH = "mismatch"


@dataclass(frozen=True, slots=True)
class SQLiteTableColumnAudit:
    status: SQLiteTableAuditStatus
    observed: tuple[SQLiteColumnContract, ...]


class SQLiteTableContractError(RuntimeError):
    """Raised when SQLite metadata cannot fit the closed audit contract."""


def _validated_utf8_text(
    value: object,
    *,
    field: str,
    maximum_bytes: int,
    allow_empty: bool,
) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be exact text")
    if not allow_empty and not value:
        raise ValueError(f"{field} must not be empty")
    # Every Unicode scalar occupies at least one UTF-8 byte. This cheap check
    # bounds the temporary allocation made by the exact byte-length check.
    if len(value) > maximum_bytes:
        raise ValueError(f"{field} exceeds its UTF-8 byte limit")
    if "\x00" in value:
        raise ValueError(f"{field} must not contain NUL")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError as error:
        raise ValueError(f"{field} must be valid UTF-8 text") from error
    if len(encoded) > maximum_bytes:
        raise ValueError(f"{field} exceeds its UTF-8 byte limit")
    return value


def _validated_expected_columns(
    value: object,
) -> tuple[SQLiteColumnContract, ...]:
    if type(value) is not tuple:
        raise TypeError("expected must be an exact tuple")
    if not 1 <= len(value) <= _MAX_SQLITE_TABLE_COLUMNS:
        raise ValueError("expected must contain between 1 and 64 columns")

    checked: list[SQLiteColumnContract] = []
    names: set[str] = set()
    primary_key_ordinals: list[int] = []
    for index, column in enumerate(value):
        if type(column) is not SQLiteColumnContract:
            raise TypeError(f"expected[{index}] must be an exact SQLiteColumnContract")
        _validated_utf8_text(
            column.name,
            field=f"expected[{index}].name",
            maximum_bytes=_MAX_SQLITE_COLUMN_NAME_BYTES,
            allow_empty=False,
        )
        if column.name in names:
            raise ValueError("expected column names must be unique")
        names.add(column.name)

        _validated_utf8_text(
            column.declared_type,
            field=f"expected[{index}].declared_type",
            maximum_bytes=_MAX_SQLITE_DECLARED_TYPE_BYTES,
            allow_empty=True,
        )
        if type(column.not_null) is not bool:
            raise TypeError(f"expected[{index}].not_null must be a Boolean")
        if column.default_sql is not None:
            _validated_utf8_text(
                column.default_sql,
                field=f"expected[{index}].default_sql",
                maximum_bytes=_MAX_SQLITE_DEFAULT_SQL_BYTES,
                allow_empty=True,
            )
        if type(column.primary_key_ordinal) is not int:
            raise TypeError(f"expected[{index}].primary_key_ordinal must be an exact integer")
        if not 0 <= column.primary_key_ordinal <= len(value):
            raise ValueError(f"expected[{index}].primary_key_ordinal is outside the column range")
        if column.primary_key_ordinal:
            primary_key_ordinals.append(column.primary_key_ordinal)
        if type(column.hidden_kind) is not int:
            raise TypeError(f"expected[{index}].hidden_kind must be an exact integer")
        if column.hidden_kind not in _SQLITE_HIDDEN_KINDS:
            raise ValueError(f"expected[{index}].hidden_kind is outside 0..3")
        checked.append(column)

    if sorted(primary_key_ordinals) != list(range(1, len(primary_key_ordinals) + 1)):
        raise ValueError("nonzero primary-key ordinals must be unique and contiguous")
    return tuple(checked)


def _observed_text(
    value: object,
    *,
    field: str,
    maximum_bytes: int,
    allow_empty: bool,
) -> str:
    try:
        return _validated_utf8_text(
            value,
            field=field,
            maximum_bytes=maximum_bytes,
            allow_empty=allow_empty,
        )
    except (TypeError, ValueError) as error:
        raise SQLiteTableContractError(
            f"SQLite returned unsupported metadata for {field}"
        ) from error


def _observed_column(
    row: tuple[object, ...],
    *,
    index: int,
) -> tuple[int, SQLiteColumnContract]:
    if type(row) is not tuple or len(row) != 8:
        raise SQLiteTableContractError("SQLite returned an unexpected metadata row")
    (
        object_type,
        cid,
        name,
        declared_type,
        not_null,
        default_sql,
        primary_key_ordinal,
        hidden_kind,
    ) = row
    if object_type != "table":
        raise SQLiteTableContractError("table metadata changed during one statement")
    if type(cid) is not int:
        raise SQLiteTableContractError("SQLite returned a non-integer column rank")
    if type(not_null) is not int or not_null not in (0, 1):
        raise SQLiteTableContractError("SQLite returned invalid nullability metadata")
    if (
        type(primary_key_ordinal) is not int
        or not 0 <= primary_key_ordinal <= _MAX_SQLITE_TABLE_COLUMNS
    ):
        raise SQLiteTableContractError("SQLite returned an invalid primary-key ordinal")
    if type(hidden_kind) is not int or hidden_kind not in _SQLITE_HIDDEN_KINDS:
        raise SQLiteTableContractError("SQLite returned an invalid hidden-column kind")
    if default_sql is not None:
        default_sql = _observed_text(
            default_sql,
            field=f"observed[{index}].default_sql",
            maximum_bytes=_MAX_SQLITE_DEFAULT_SQL_BYTES,
            allow_empty=True,
        )
    return (
        cid,
        SQLiteColumnContract(
            name=_observed_text(
                name,
                field=f"observed[{index}].name",
                maximum_bytes=_MAX_SQLITE_COLUMN_NAME_BYTES,
                allow_empty=False,
            ),
            declared_type=_observed_text(
                declared_type,
                field=f"observed[{index}].declared_type",
                maximum_bytes=_MAX_SQLITE_DECLARED_TYPE_BYTES,
                allow_empty=True,
            ),
            not_null=bool(not_null),
            default_sql=default_sql,
            primary_key_ordinal=primary_key_ordinal,
            hidden_kind=hidden_kind,
        ),
    )


def audit_sqlite_table_columns(
    connection: sqlite3.Connection,
    table_name: str,
    expected: tuple[SQLiteColumnContract, ...],
) -> SQLiteTableColumnAudit:
    """Compare one main-schema table with an exact ordered column contract."""
    if type(connection) is not sqlite3.Connection:
        raise TypeError("connection must be an exact sqlite3.Connection")
    if connection.text_factory is not str:
        raise ValueError("connection.text_factory must be the default str type")
    if type(table_name) is not str:
        raise TypeError("table_name must be exact text")
    if _SQLITE_TABLE_NAME.fullmatch(table_name) is None:
        raise ValueError("table_name must be a conservative ASCII identifier")
    if table_name.casefold() in _SQLITE_SCHEMA_ALIASES:
        raise ValueError("SQLite schema-table aliases are outside this contract")
    if len(table_name.encode("ascii")) > _MAX_SQLITE_TABLE_NAME_BYTES:
        raise ValueError("table_name exceeds its byte limit")
    checked_expected = _validated_expected_columns(expected)

    cursor = connection.cursor()
    cursor.row_factory = None
    try:
        cursor.execute(_TABLE_XINFO_QUERY, (table_name, table_name))
        rows = cursor.fetchmany(_MAX_SQLITE_TABLE_COLUMNS + 2)
    finally:
        cursor.close()

    if not rows:
        return SQLiteTableColumnAudit(SQLiteTableAuditStatus.MISSING, ())
    if rows[0][0] != "table":
        if len(rows) != 1 or type(rows[0][0]) is not str:
            raise SQLiteTableContractError("SQLite returned an ambiguous schema object")
        return SQLiteTableColumnAudit(SQLiteTableAuditStatus.NOT_A_TABLE, ())
    if len(rows) > _MAX_SQLITE_TABLE_COLUMNS:
        raise SQLiteTableContractError("the observed table has more than 64 columns")

    observed: list[SQLiteColumnContract] = []
    previous_cid: int | None = None
    for index, row in enumerate(rows):
        cid, column = _observed_column(row, index=index)
        if previous_cid is not None and cid <= previous_cid:
            raise SQLiteTableContractError("SQLite returned non-increasing column ranks")
        previous_cid = cid
        observed.append(column)

    frozen_observed = tuple(observed)
    status = (
        SQLiteTableAuditStatus.MATCH
        if frozen_observed == checked_expected
        else SQLiteTableAuditStatus.MISMATCH
    )
    return SQLiteTableColumnAudit(status, frozen_observed)
```

## Example

```python
from dataclasses import replace


def assert_raises(error_type, operation):
    try:
        operation()
    except error_type:
        return True
    raise AssertionError(f"{error_type.__name__} was not raised")


connection = sqlite3.connect(":memory:")
connection.execute("CREATE TABLE ordinary_items (name TEXT, amount INTEGER DEFAULT 7)")
connection.execute(
    """
    CREATE TABLE strict_items (
        item_id INTEGER PRIMARY KEY,
        label TEXT NOT NULL DEFAULT 'ready',
        score REAL,
        label_size INTEGER GENERATED ALWAYS AS (length(label)) STORED
    ) STRICT
    """
)
connection.execute(
    """
    CREATE TABLE composite_keys (
        region TEXT NOT NULL,
        sequence INTEGER NOT NULL,
        payload BLOB,
        PRIMARY KEY (region, sequence)
    ) WITHOUT ROWID, STRICT
    """
)
connection.execute("CREATE VIEW item_view AS SELECT item_id, label FROM strict_items")

boundary_columns_sql = ", ".join(f"c{index} INTEGER" for index in range(_MAX_SQLITE_TABLE_COLUMNS))
connection.execute(f"CREATE TABLE boundary_columns ({boundary_columns_sql})")
over_limit_columns_sql = ", ".join(
    f"c{index} INTEGER" for index in range(_MAX_SQLITE_TABLE_COLUMNS + 1)
)
connection.execute(f"CREATE TABLE over_limit_columns ({over_limit_columns_sql})")

maximum_table_name = "t" * _MAX_SQLITE_TABLE_NAME_BYTES
connection.execute(f'CREATE TABLE "{maximum_table_name}" (value TEXT)')

ordinary_expected = (
    SQLiteColumnContract("name", "TEXT", False, None, 0, 0),
    SQLiteColumnContract("amount", "INTEGER", False, "7", 0, 0),
)
strict_expected = (
    SQLiteColumnContract("item_id", "INTEGER", False, None, 1, 0),
    SQLiteColumnContract("label", "TEXT", True, "'ready'", 0, 0),
    SQLiteColumnContract("score", "REAL", False, None, 0, 0),
    SQLiteColumnContract("label_size", "INTEGER", False, None, 0, 3),
)
composite_expected = (
    SQLiteColumnContract("region", "TEXT", True, None, 1, 0),
    SQLiteColumnContract("sequence", "INTEGER", True, None, 2, 0),
    SQLiteColumnContract("payload", "BLOB", False, None, 0, 0),
)
boundary_expected = tuple(
    SQLiteColumnContract(f"c{index}", "INTEGER", False, None, 0, 0)
    for index in range(_MAX_SQLITE_TABLE_COLUMNS)
)
maximum_name_expected = (SQLiteColumnContract("value", "TEXT", False, None, 0, 0),)

state_before = (
    connection.total_changes,
    connection.execute("PRAGMA main.schema_version").fetchone()[0],
    connection.in_transaction,
)

ordinary = audit_sqlite_table_columns(
    connection,
    "ordinary_items",
    ordinary_expected,
)
strict = audit_sqlite_table_columns(
    connection,
    "strict_items",
    strict_expected,
)
composite = audit_sqlite_table_columns(
    connection,
    "composite_keys",
    composite_expected,
)
boundary = audit_sqlite_table_columns(
    connection,
    "boundary_columns",
    boundary_expected,
)
maximum_name = audit_sqlite_table_columns(
    connection,
    maximum_table_name,
    maximum_name_expected,
)
view = audit_sqlite_table_columns(
    connection,
    "item_view",
    strict_expected[:2],
)
missing = audit_sqlite_table_columns(
    connection,
    "absent_table",
    ordinary_expected,
)

mismatch_contracts = (
    tuple(reversed(ordinary_expected)),
    (replace(ordinary_expected[0], declared_type="BLOB"), ordinary_expected[1]),
    (replace(ordinary_expected[0], not_null=True), ordinary_expected[1]),
    (ordinary_expected[0], replace(ordinary_expected[1], default_sql="8")),
    (
        replace(ordinary_expected[0], primary_key_ordinal=1),
        ordinary_expected[1],
    ),
    (replace(ordinary_expected[0], hidden_kind=2), ordinary_expected[1]),
    ordinary_expected[:1],
    (*ordinary_expected, SQLiteColumnContract("extra", "TEXT", False, None, 0, 0)),
)
mismatches = tuple(
    audit_sqlite_table_columns(connection, "ordinary_items", expected)
    for expected in mismatch_contracts
)

invalid_inputs = (
    lambda: audit_sqlite_table_columns(connection, "ordinary_items", ()),
    lambda: audit_sqlite_table_columns(
        connection,
        "ordinary_items",
        (
            *boundary_expected,
            SQLiteColumnContract("extra", "INTEGER", False, None, 0, 0),
        ),
    ),
    lambda: audit_sqlite_table_columns(
        connection,
        "invalid-name",
        ordinary_expected,
    ),
    lambda: audit_sqlite_table_columns(
        connection,
        "ordinary_items",
        (replace(ordinary_expected[0], default_sql="x" * 1_025),),
    ),
    lambda: audit_sqlite_table_columns(
        connection,
        "sqlite_schema",
        ordinary_expected,
    ),
    lambda: audit_sqlite_table_columns(
        connection,
        "SQLITE_MASTER",
        ordinary_expected,
    ),
)
invalid_rejections = tuple(assert_raises(ValueError, call) for call in invalid_inputs)
boolean_ordinal_rejected = assert_raises(
    TypeError,
    lambda: audit_sqlite_table_columns(
        connection,
        "ordinary_items",
        (replace(ordinary_expected[0], primary_key_ordinal=True),),
    ),
)
observed_limit_rejected = assert_raises(
    SQLiteTableContractError,
    lambda: audit_sqlite_table_columns(
        connection,
        "over_limit_columns",
        boundary_expected,
    ),
)

bytes_connection = sqlite3.connect(":memory:")
bytes_connection.text_factory = bytes
text_factory_rejected = assert_raises(
    ValueError,
    lambda: audit_sqlite_table_columns(
        bytes_connection,
        "anything",
        maximum_name_expected,
    ),
)
bytes_connection.close()

previous_text_converter = sqlite3.converters.get("TEXT")
sqlite3.register_converter("TEXT", lambda raw: f"converted:{raw.decode('utf-8')}")
try:
    converter_connection = sqlite3.connect(
        ":memory:",
        detect_types=sqlite3.PARSE_DECLTYPES,
    )
    converter_connection.execute(
        "CREATE TABLE converter_items (name TEXT, amount INTEGER DEFAULT 7)"
    )
    converter_safe = audit_sqlite_table_columns(
        converter_connection,
        "converter_items",
        ordinary_expected,
    )
finally:
    converter_connection.close()
    if previous_text_converter is None:
        sqlite3.converters.pop("TEXT", None)
    else:
        sqlite3.converters["TEXT"] = previous_text_converter

state_after = (
    connection.total_changes,
    connection.execute("PRAGMA main.schema_version").fetchone()[0],
    connection.in_transaction,
)
connection.close()

assert (
    ordinary,
    strict,
    composite,
    boundary.status,
    maximum_name.status,
    view,
    missing,
    tuple(result.status for result in mismatches),
    all(result.observed == ordinary_expected for result in mismatches),
    invalid_rejections,
    boolean_ordinal_rejected,
    observed_limit_rejected,
    text_factory_rejected,
    converter_safe,
    state_after == state_before,
) == (
    SQLiteTableColumnAudit(SQLiteTableAuditStatus.MATCH, ordinary_expected),
    SQLiteTableColumnAudit(SQLiteTableAuditStatus.MATCH, strict_expected),
    SQLiteTableColumnAudit(SQLiteTableAuditStatus.MATCH, composite_expected),
    SQLiteTableAuditStatus.MATCH,
    SQLiteTableAuditStatus.MATCH,
    SQLiteTableColumnAudit(SQLiteTableAuditStatus.NOT_A_TABLE, ()),
    SQLiteTableColumnAudit(SQLiteTableAuditStatus.MISSING, ()),
    (SQLiteTableAuditStatus.MISMATCH,) * len(mismatch_contracts),
    True,
    (True,) * len(invalid_inputs),
    True,
    True,
    True,
    SQLiteTableColumnAudit(SQLiteTableAuditStatus.MATCH, ordinary_expected),
    True,
)
```

## Trade-offs and Limitations

The statement reads schema metadata only and returns at most 64 columns. Work
and retained output are `O(C + B)` for the observed column count `C` and the
bounded metadata text size `B`; fetching one row beyond the cap makes an
oversized table fail without returning a partial contract. Local character
preflights bound UTF-8 temporaries for caller-provided text, but SQLite and
`sqlite3` can materialize returned metadata before the audit enforces its byte
limits. The statement does not read table rows or modify the connection's
transaction, row factory, text factory, or database contents.

Comparison is lexical, not semantic. SQLite may preserve different type
spelling or default-expression text for equivalent declarations, and an
`INTEGER PRIMARY KEY` can report `notnull = 0` unless `NOT NULL` was explicit.
The contract records those values exactly. Generated expressions themselves,
collations, uniqueness, checks, foreign keys, indexes, triggers, `STRICT`, and
`WITHOUT ROWID` are outside the result.

Only the `main` schema and conservative ASCII table names are supported. A
view is reported separately, while virtual and shadow tables that SQLite lists
as tables are judged by their `table_xinfo` columns, including hidden kind
`1`. The linked SQLite library must provide the table-valued `table_xinfo`
pragma. Operational database errors propagate, and a custom `text_factory` is
rejected so returned metadata cannot silently change type.

The observed tuple can contain default-expression literals and should not be
logged where schema text is sensitive. A match proves only one statement's
snapshot; it does not reserve the table, authorize later SQL, or prevent a
different connection from changing the schema after the audit returns.

## Related Snippets

<!-- catalog:related:start -->
- [Plan an Additive SQLite Column Projection](plan-an-additive-sqlite-column-projection.md)
- [Plan Bounded Table Initialization and Ordered Row Batches](plan-bounded-table-initialization-and-ordered-row-batches.md)
- [Open a Verified Read-Only SQLite Connection Under a Closed Hardening Profile](open-a-verified-read-only-sqlite-connection-under-a-closed-hardening-profile.md)
<!-- catalog:related:end -->
