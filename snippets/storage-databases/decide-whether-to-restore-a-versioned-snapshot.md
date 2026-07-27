---
title: "Decide Whether to Restore a Versioned Snapshot"
snippet_type: algorithm
use_cases:
  - configuration
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - plan-an-incremental-cumulative-snapshot-from-partition-dates.md
  - select-snapshot-representatives-by-utc-calendar-buckets.md
  - fingerprint-a-bounded-flat-file-set-with-framed-sha-256.md
---

# Decide Whether to Restore a Versioned Snapshot

## Idea and Problem

Make the restore-or-skip decision from exact immutable snapshot identities before any filesystem or archive operation begins.

An identity contains a neutral format identifier, a positive schema version,
and a lowercase SHA-256-shaped digest supplied by an earlier trusted boundary.
The candidate format and version are validated against the supported contract
before equality is considered, so an already-installed but unsupported
candidate can never be accepted as current by accident.

## When to Use

Use this algorithm as a pure preflight step after another component has
authenticated or calculated snapshot metadata. A caller can execute a returned
`restore` plan with its own bounded extraction, atomic replacement, rollback,
and durability policy, or avoid that work for an exactly current identity.

Do not treat a digest-shaped string as proof of integrity. Use a cryptographic
verification boundary before this decision when metadata or snapshot bytes can
be modified by an untrusted party.

## Implementation

```python
import re
from dataclasses import dataclass
from typing import Literal


_MAX_FORMAT_ID_CHARACTERS = 32
_MAX_VERSION = (1 << 31) - 1
_MAX_SUPPORTED_VERSIONS = 32
_FORMAT_ID = re.compile(r"[a-z][a-z0-9-]{0,31}", re.ASCII)
_LOWER_SHA256 = re.compile(r"[0-9a-f]{64}", re.ASCII)


@dataclass(frozen=True, slots=True)
class SnapshotIdentity:
    format_id: str
    version: int
    digest: str


@dataclass(frozen=True, slots=True)
class SnapshotRestorePlan:
    action: Literal["restore", "already-current"]
    candidate: SnapshotIdentity
    current: SnapshotIdentity | None


def _validated_format_id(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if (
        len(value) > _MAX_FORMAT_ID_CHARACTERS
        or _FORMAT_ID.fullmatch(value) is None
    ):
        raise ValueError(f"{field} is outside the supported format")
    return value


def _validated_version(value: object, *, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    if not 1 <= value <= _MAX_VERSION:
        raise ValueError(f"{field} is outside the supported range")
    return value


def _validated_identity(value: object, *, field: str) -> SnapshotIdentity:
    if type(value) is not SnapshotIdentity:
        raise TypeError(f"{field} must be an exact SnapshotIdentity")
    format_id = _validated_format_id(value.format_id, field=f"{field} format ID")
    version = _validated_version(value.version, field=f"{field} version")
    if type(value.digest) is not str:
        raise TypeError(f"{field} digest must be an exact string")
    if _LOWER_SHA256.fullmatch(value.digest) is None:
        raise ValueError(f"{field} digest must be a lowercase SHA-256 shape")
    return SnapshotIdentity(format_id, version, value.digest)


def _supported_version_tuple(value: object) -> tuple[int, ...]:
    if type(value) is not tuple:
        raise TypeError("supported_versions must be an exact tuple")
    if not 1 <= len(value) <= _MAX_SUPPORTED_VERSIONS:
        raise ValueError("supported version count is outside the supported range")

    versions = tuple(
        _validated_version(version, field="supported version")
        for version in value
    )
    if any(left >= right for left, right in zip(versions, versions[1:])):
        raise ValueError("supported_versions must be strictly increasing")
    return versions


def decide_snapshot_restore(
    candidate: SnapshotIdentity,
    current: SnapshotIdentity | None,
    *,
    supported_format_id: str,
    supported_versions: tuple[int, ...],
) -> SnapshotRestorePlan:
    expected_format = _validated_format_id(
        supported_format_id,
        field="supported format ID",
    )
    versions = _supported_version_tuple(supported_versions)
    proposed = _validated_identity(candidate, field="candidate")

    if proposed.format_id != expected_format or proposed.version not in versions:
        raise ValueError("candidate snapshot format or version is not supported")

    installed = (
        None
        if current is None
        else _validated_identity(current, field="current")
    )
    if installed is not None and installed.format_id != expected_format:
        raise ValueError("current snapshot uses a different format")
    action: Literal["restore", "already-current"] = "restore"
    if installed is not None and installed == proposed:
        action = "already-current"
    return SnapshotRestorePlan(action, proposed, installed)
```

## Example

```python
current = SnapshotIdentity(
    format_id="vector-index",
    version=2,
    digest="b" * 64,
)
candidate = SnapshotIdentity(
    format_id="vector-index",
    version=3,
    digest="a" * 64,
)
restore_plan = decide_snapshot_restore(
    candidate,
    current,
    supported_format_id="vector-index",
    supported_versions=(2, 3),
)
current_plan = decide_snapshot_restore(
    candidate,
    candidate,
    supported_format_id="vector-index",
    supported_versions=(2, 3),
)
lower_version_plan = decide_snapshot_restore(
    SnapshotIdentity("vector-index", 2, "d" * 64),
    candidate,
    supported_format_id="vector-index",
    supported_versions=(2, 3),
)

try:
    decide_snapshot_restore(
        SnapshotIdentity("vector-index", 4, "c" * 64),
        SnapshotIdentity("vector-index", 4, "c" * 64),
        supported_format_id="vector-index",
        supported_versions=(2, 3),
    )
except ValueError:
    unsupported_current_candidate_rejected = True
else:
    unsupported_current_candidate_rejected = False

assert (
    restore_plan.action,
    restore_plan.candidate,
    current_plan.action,
    lower_version_plan.action,
    unsupported_current_candidate_rejected,
) == (
    "restore",
    candidate,
    "already-current",
    "restore",
    True,
)
```

## Trade-offs and Limitations

Validation uses constant memory and work bounded by at most 32 supported
versions. The function defensively reconstructs each accepted identity and
returns only frozen values. A same-version candidate with a different digest
requires restoration. A different supported candidate version also produces a
restore plan; the caller owns upgrade and downgrade policy.

The digest is checked only for lowercase hexadecimal shape. This algorithm does
not calculate or authenticate it, parse a filename, open an archive, inspect
members, access a filesystem, delete a directory, replace installed data, or
perform rollback. A `restore` result is a plan, not evidence that execution is
safe or complete.

## Related Snippets

<!-- catalog:related:start -->
- [Plan an Incremental Cumulative Snapshot from Partition Dates](plan-an-incremental-cumulative-snapshot-from-partition-dates.md)
- [Select Snapshot Representatives by UTC Calendar Buckets](select-snapshot-representatives-by-utc-calendar-buckets.md)
- [Fingerprint a Bounded Flat File Set with Framed SHA-256](fingerprint-a-bounded-flat-file-set-with-framed-sha-256.md)
<!-- catalog:related:end -->
