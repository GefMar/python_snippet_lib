---
title: "Select Expired Backup Names with Strict UTC Timestamps"
snippet_type: recipe
use_cases:
  - automation
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - select-snapshot-representatives-by-utc-calendar-buckets.md
  - check-whether-a-generated-file-is-older-than-its-inputs.md
  - ../security-privacy/audit-symlinks-in-a-bounded-directory-tree.md
---

# Select Expired Backup Names with Strict UTC Timestamps

## Idea and Problem

Classify bounded backup names by a strict embedded UTC timestamp without reading metadata or changing anything on the filesystem.

Only names in the ASCII form `backup-YYYYMMDDTHHMMSSZ` are eligible. A valid
name expires when its timestamp is at or before the exact UTC cutoff. Malformed
names, impossible calendar dates, future timestamps, and duplicated names are
reported explicitly so a caller cannot mistake them for safe deletion targets.

## When to Use

Use this recipe after a trusted component has supplied a finite collection of
candidate names and retention is based solely on the timestamp encoded in each
name. Supply an aware UTC clock value and a non-negative age no greater than
the fixed retention ceiling. A timestamp exactly on the cutoff is expired; a
valid timestamp after the cutoff and no later than `now` is retained and does
not appear in either returned tuple.

Use storage-native lifecycle rules when they are available. If file identity,
ownership, type, or modification metadata affects the policy, verify those
properties in a separate filesystem-aware layer before taking action.

## Implementation

```python
import re
from collections import Counter
from collections.abc import Iterable
from dataclasses import dataclass
from datetime import UTC, datetime, timedelta
from itertools import islice


_BACKUP_NAME = re.compile(r"backup-([0-9]{8}T[0-9]{6}Z)", re.ASCII)
_MAX_NAMES = 4_096
_MAX_NAME_CHARACTERS = 128
_MAX_AGE = timedelta(days=36_500)


@dataclass(frozen=True, slots=True)
class IgnoredBackupName:
    name: str
    reason: str


@dataclass(frozen=True, slots=True)
class BackupNameSelection:
    expired: tuple[str, ...]
    ignored: tuple[IgnoredBackupName, ...]


def _bounded_names(names: Iterable[str]) -> tuple[str, ...]:
    if isinstance(names, (str, bytes)):
        raise TypeError("names must be a non-text iterable")
    try:
        snapshot = tuple(islice(names, _MAX_NAMES + 1))
    except TypeError as error:
        raise TypeError("names must be an iterable") from error
    if len(snapshot) > _MAX_NAMES:
        raise ValueError("name input exceeds the supported item limit")
    for name in snapshot:
        if type(name) is not str:
            raise TypeError("names must contain only text values")
        if len(name) > _MAX_NAME_CHARACTERS:
            raise ValueError("a name exceeds the supported character limit")
    return snapshot


def select_expired_backup_names(
    names: Iterable[str],
    *,
    now: datetime,
    max_age: timedelta,
) -> BackupNameSelection:
    if not isinstance(now, datetime):
        raise TypeError("now must be a datetime")
    if now.tzinfo is None or now.utcoffset() is None:
        raise ValueError("now must be timezone-aware")
    if now.utcoffset() != timedelta(0):
        raise ValueError("now must use a UTC offset")
    if not isinstance(max_age, timedelta):
        raise TypeError("max_age must be a timedelta")
    if not timedelta(0) <= max_age <= _MAX_AGE:
        raise ValueError("max_age is outside the supported range")

    now_utc = now.astimezone(UTC)
    try:
        cutoff = now_utc - max_age
    except OverflowError as error:
        raise ValueError("max_age cannot be represented relative to now") from error

    counts = Counter(_bounded_names(names))
    expired: list[str] = []
    ignored: list[IgnoredBackupName] = []
    for name in sorted(counts):
        if counts[name] > 1:
            ignored.append(IgnoredBackupName(name, "duplicate"))
            continue

        match = _BACKUP_NAME.fullmatch(name)
        if match is None:
            ignored.append(IgnoredBackupName(name, "invalid-format"))
            continue
        try:
            created_at = datetime.strptime(
                match.group(1),
                "%Y%m%dT%H%M%SZ",
            ).replace(tzinfo=UTC)
        except ValueError:
            ignored.append(IgnoredBackupName(name, "invalid-date"))
            continue

        if created_at > now_utc:
            ignored.append(IgnoredBackupName(name, "future"))
        elif created_at <= cutoff:
            expired.append(name)

    return BackupNameSelection(tuple(expired), tuple(ignored))
```

## Example

```python
now = datetime(2026, 7, 27, 12, 0, 0, tzinfo=UTC)
duplicate = "backup-20260719T120000Z"
selection = select_expired_backup_names(
    (
        "backup-20260720T120001Z",
        "backup-20260720T120000Z",
        "backup-20260720T115959Z",
        "backup-20260728T120000Z",
        "backup-20260230T120000Z",
        "backup-latest",
        duplicate,
        duplicate,
    ),
    now=now,
    max_age=timedelta(days=7),
)

invalid_clock_or_age = []
for invalid_now, invalid_age in (
    (datetime(2026, 7, 27, 12, 0, 0), timedelta(days=7)),
    (now, timedelta(seconds=-1)),
    (now, timedelta(days=36_501)),
):
    try:
        select_expired_backup_names(
            (),
            now=invalid_now,
            max_age=invalid_age,
        )
    except ValueError:
        invalid_clock_or_age.append(True)

assert (
    selection.expired,
    selection.ignored,
    len(invalid_clock_or_age),
) == (
    (
        "backup-20260720T115959Z",
        "backup-20260720T120000Z",
    ),
    (
        IgnoredBackupName("backup-20260230T120000Z", "invalid-date"),
        IgnoredBackupName(duplicate, "duplicate"),
        IgnoredBackupName("backup-20260728T120000Z", "future"),
        IgnoredBackupName("backup-latest", "invalid-format"),
    ),
    3,
)
```

## Trade-offs and Limitations

The function materializes a bounded input tuple and sorts its distinct names,
using linear memory and `O(n log n)` time. A repeated name is ignored as one
ambiguous candidate regardless of how many times it occurs; duplicates are not
silently collapsed into a deletion target. The fixed grammar excludes
subdirectories, extensions, alternate prefixes, fractional seconds, offsets,
and leap-second spellings.

This is name selection, not a deletion routine. It never stats paths, follows
links, opens files, or removes data. A privileged caller must separately bind a
selected name to an authorized storage location, reject unsafe file types and
links, handle concurrent replacement between checking and deletion, and apply
the required authentication, authorization, audit, and recovery policy. Those
filesystem checks still have time-of-check/time-of-use risks unless the storage
API provides an atomic identity-aware operation.

## Related Snippets

<!-- catalog:related:start -->
- [Select Snapshot Representatives by UTC Calendar Buckets](select-snapshot-representatives-by-utc-calendar-buckets.md)
- [Check Whether a Generated File Is Older Than Its Inputs](check-whether-a-generated-file-is-older-than-its-inputs.md)
- [Audit Symlinks in a Bounded Directory Tree](../security-privacy/audit-symlinks-in-a-bounded-directory-tree.md)
<!-- catalog:related:end -->
