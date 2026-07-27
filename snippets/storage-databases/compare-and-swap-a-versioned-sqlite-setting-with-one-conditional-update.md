---
title: "Compare and Swap a Versioned SQLite Setting with One Conditional Update"
snippet_type: recipe
use_cases:
  - concurrency-control
  - persistence
tested_python:
  - "3.14"
dependencies: []
related:
  - build-and-apply-a-deterministic-mapping-delta.md
  - replace-a-file-atomically-with-a-sibling-temporary-file.md
  - store-bytes-by-their-content-digest.md
---

# Compare and Swap a Versioned SQLite Setting with One Conditional Update

## Idea and Problem

Replace a SQLite setting only when its stored version still matches the version previously observed by the caller.

One conditional `UPDATE` changes the bytes and increments the integer version
atomically. Its row count distinguishes success from a missing or stale row;
there is no separate read-check race. The function owns a fresh connection and
transaction, commits before returning, and lets storage failures remain
distinct from an ordinary compare-and-swap miss.

## When to Use

Use this recipe for a fixed local SQLite schema when multiple writers may base
changes on the same earlier value and every writer follows the version
protocol. The caller should read the value and version through its normal data
access path, compute replacement bytes, then submit the observed version. Use
a richer transaction when one decision spans multiple rows or invariants.

## Implementation

```python
import math
import re
import sqlite3
from contextlib import closing
from pathlib import Path


_SETTING_NAME = re.compile(r"[A-Za-z][A-Za-z0-9_.-]{0,127}\Z")
_MAX_VALUE_BYTES = 1_000_000
_MAX_VERSION_BEFORE_INCREMENT = 2**63 - 2


def compare_and_swap_setting(
    database: Path,
    *,
    name: str,
    expected_version: int,
    value: bytes,
    busy_timeout: float = 5.0,
) -> bool:
    if not isinstance(name, str) or _SETTING_NAME.fullmatch(name) is None:
        raise ValueError("name must be a bounded canonical setting key")
    if isinstance(expected_version, bool) or not isinstance(expected_version, int):
        raise TypeError("expected_version must be an integer")
    if not 0 <= expected_version <= _MAX_VERSION_BEFORE_INCREMENT:
        raise ValueError("expected_version is outside SQLite's incrementable range")
    if not isinstance(value, bytes):
        raise TypeError("value must be immutable bytes")
    if len(value) > _MAX_VALUE_BYTES:
        raise ValueError(f"value cannot exceed {_MAX_VALUE_BYTES} bytes")
    if not isinstance(busy_timeout, (int, float)) or isinstance(busy_timeout, bool):
        raise TypeError("busy_timeout must be a real number")
    try:
        busy_timeout = float(busy_timeout)
    except OverflowError as error:
        raise ValueError("busy_timeout must be finite") from error
    if not math.isfinite(busy_timeout) or not 0.0 <= busy_timeout <= 60.0:
        raise ValueError("busy_timeout must be finite and between 0 and 60 seconds")

    connection = sqlite3.connect(
        Path(database),
        timeout=busy_timeout,
        autocommit=False,
    )
    with closing(connection):
        try:
            cursor = connection.execute(
                """
                UPDATE settings
                SET value = ?, version = version + 1
                WHERE name = ? AND version = ?
                """,
                (value, name, expected_version),
            )
            if cursor.rowcount not in (0, 1):
                raise RuntimeError("the settings key is not unique")
            updated = cursor.rowcount == 1
            connection.commit()
        except BaseException:
            connection.rollback()
            raise
    return updated
```

## Example

```python
from tempfile import TemporaryDirectory


with TemporaryDirectory() as temporary_directory:
    database = Path(temporary_directory) / "settings.sqlite3"
    with closing(sqlite3.connect(database, autocommit=False)) as setup:
        with setup:
            setup.execute(
                """
                CREATE TABLE settings (
                    name TEXT PRIMARY KEY,
                    value BLOB NOT NULL,
                    version INTEGER NOT NULL CHECK (version >= 0)
                )
                """,
            )
            setup.execute(
                "INSERT INTO settings(name, value, version) VALUES (?, ?, ?)",
                ("feature.flag", b"off", 0),
            )

    first = compare_and_swap_setting(
        database,
        name="feature.flag",
        expected_version=0,
        value=b"first",
    )
    stale = compare_and_swap_setting(
        database,
        name="feature.flag",
        expected_version=0,
        value=b"stale",
    )
    second = compare_and_swap_setting(
        database,
        name="feature.flag",
        expected_version=1,
        value=b"second",
    )
    missing = compare_and_swap_setting(
        database,
        name="missing.flag",
        expected_version=0,
        value=b"unused",
    )
    with closing(sqlite3.connect(database)) as reader:
        stored = reader.execute(
            "SELECT value, version FROM settings WHERE name = ?",
            ("feature.flag",),
        ).fetchone()

assert (first, stale, second, missing, stored) == (
    True,
    False,
    True,
    False,
    (b"second", 2),
)
```

## Trade-offs and Limitations

Every call opens and closes a connection, which favors ownership clarity over
high-throughput batching. A zero row count intentionally does not distinguish
a missing key from a stale version. Lock contention, I/O errors, and commit
failures propagate and are not converted to `False` or retried. All writers
must increment the version or ABA-style changes can evade detection. Schema
creation, JSON encoding, auditing, migrations, multi-row transactions, and
conflict retry policy are outside this primitive.

## Related Snippets

<!-- catalog:related:start -->
- [Build and Apply a Deterministic Mapping Delta](build-and-apply-a-deterministic-mapping-delta.md)
- [Replace a File Atomically with a Sibling Temporary File](replace-a-file-atomically-with-a-sibling-temporary-file.md)
- [Store Bytes by Their Content Digest](store-bytes-by-their-content-digest.md)
<!-- catalog:related:end -->
