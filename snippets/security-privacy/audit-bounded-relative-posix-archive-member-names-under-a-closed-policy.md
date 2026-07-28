---
title: "Audit Bounded Relative POSIX Archive Member Names Under a Closed Policy"
snippet_type: recipe
use_cases:
  - parsing
  - resource-management
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - audit-symlinks-in-a-bounded-directory-tree.md
  - ../configuration-serialization/build-a-deterministic-size-capped-ustar-archive-from-bytes.md
  - ../storage-databases/plan-a-bounded-named-posix-layout-under-explicit-roots.md
---

# Audit Bounded Relative POSIX Archive Member Names Under a Closed Policy

## Idea and Problem

Audit one exact tuple of already-decoded archive member names under bounded conservative relative POSIX lexical rules without opening an archive or touching a filesystem.

The audit validates member count, individual and aggregate ASCII byte lengths,
path depth, segment length, syntax, and exact uniqueness before returning
immutable name-and-parts records. Validation operates on the original strings
so repeated separators, dot segments, backslashes, and drive-looking spellings
cannot disappear through path normalization.

## When to Use

Use this recipe after a trusted archive parser has already produced decoded
member-name strings and a later application stage accepts only a deliberately
small portable naming profile. Choose every limit from the expected archive
shape, and keep the returned tuple in input order when later checks must refer
back to the parser's member order.

Use format-aware archive validation as well. A member name alone does not say
whether an entry is a regular file, directory, link, device, or another archive
type, and it carries no trustworthy size or extraction behavior.

## Implementation

```python
import re
from dataclasses import dataclass

_MAX_MEMBER_LIMIT = 10_000
_MAX_NAME_BYTE_LIMIT = 4_096
_MAX_TOTAL_NAME_BYTE_LIMIT = 8 * 1024 * 1024
_MAX_DEPTH_LIMIT = 128
_MAX_SEGMENT_BYTE_LIMIT = 255
_SEGMENT = re.compile(r"[A-Za-z0-9._-]+", re.ASCII)


class ArchiveMemberNameError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class AuditedArchiveMemberName:
    name: str
    parts: tuple[str, ...]


def _bounded_limit(
    value: object,
    *,
    field: str,
    minimum: int,
    maximum: int,
) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    if not minimum <= value <= maximum:
        raise ValueError(f"{field} is outside the supported range")
    return value


def audit_archive_member_names(
    names: tuple[str, ...],
    *,
    max_members: int,
    max_name_bytes: int,
    max_total_name_bytes: int,
    max_depth: int,
    max_segment_bytes: int,
) -> tuple[AuditedArchiveMemberName, ...]:
    if type(names) is not tuple:
        raise TypeError("names must be an exact tuple")

    member_limit = _bounded_limit(
        max_members,
        field="max_members",
        minimum=0,
        maximum=_MAX_MEMBER_LIMIT,
    )
    name_limit = _bounded_limit(
        max_name_bytes,
        field="max_name_bytes",
        minimum=1,
        maximum=_MAX_NAME_BYTE_LIMIT,
    )
    total_limit = _bounded_limit(
        max_total_name_bytes,
        field="max_total_name_bytes",
        minimum=0,
        maximum=_MAX_TOTAL_NAME_BYTE_LIMIT,
    )
    depth_limit = _bounded_limit(
        max_depth,
        field="max_depth",
        minimum=1,
        maximum=_MAX_DEPTH_LIMIT,
    )
    segment_limit = _bounded_limit(
        max_segment_bytes,
        field="max_segment_bytes",
        minimum=1,
        maximum=_MAX_SEGMENT_BYTE_LIMIT,
    )
    if len(names) > member_limit:
        raise ArchiveMemberNameError("member count exceeds max_members")

    audited: list[AuditedArchiveMemberName] = []
    seen: set[str] = set()
    total_bytes = 0
    for name in names:
        if type(name) is not str:
            raise TypeError("every member name must be an exact string")
        if not name:
            raise ArchiveMemberNameError("member names must not be empty")
        if len(name) > name_limit:
            raise ArchiveMemberNameError("a member name exceeds max_name_bytes")
        if not name.isascii():
            raise ArchiveMemberNameError("member names must contain only ASCII")
        if any(not 0x20 <= ord(character) <= 0x7E for character in name):
            raise ArchiveMemberNameError("member names must be printable ASCII")
        if name.startswith("/"):
            raise ArchiveMemberNameError("member names must be relative")
        if name.endswith("/"):
            raise ArchiveMemberNameError("member names must not end with a separator")
        if "//" in name:
            raise ArchiveMemberNameError("member names must not repeat separators")
        if "\\" in name:
            raise ArchiveMemberNameError("member names must not contain backslashes")
        if ":" in name:
            raise ArchiveMemberNameError("member names must not contain drive-looking segments")

        parts = tuple(name.split("/"))
        if len(parts) > depth_limit:
            raise ArchiveMemberNameError("a member name exceeds max_depth")
        for part in parts:
            if part in {".", ".."}:
                raise ArchiveMemberNameError("dot path segments are not allowed")
            if len(part) > segment_limit:
                raise ArchiveMemberNameError("a member segment exceeds max_segment_bytes")
            if _SEGMENT.fullmatch(part) is None:
                raise ArchiveMemberNameError("a member segment has invalid conservative syntax")

        if name in seen:
            raise ArchiveMemberNameError("member names must be exactly unique")
        seen.add(name)
        total_bytes += len(name)
        if total_bytes > total_limit:
            raise ArchiveMemberNameError("member names exceed max_total_name_bytes")
        audited.append(AuditedArchiveMemberName(name, parts))

    return tuple(audited)
```

## Example

```python
policy = {
    "max_members": 8,
    "max_name_bytes": 64,
    "max_total_name_bytes": 160,
    "max_depth": 4,
    "max_segment_bytes": 24,
}
accepted = audit_archive_member_names(
    (
        "docs/readme.txt",
        ".metadata/index-01",
        "data/chunk_0001.bin",
    ),
    **policy,
)

invalid_names = (
    "",
    "/absolute.txt",
    "trailing/",
    "a//b.txt",
    "./readme.txt",
    "docs/../readme.txt",
    "windows\\path.txt",
    "line\nbreak.txt",
    "café.txt",
    "C:/drive.txt",
    "name:stream",
    "space name.txt",
)
rejected = []
for candidate in invalid_names:
    try:
        audit_archive_member_names((candidate,), **policy)
    except ArchiveMemberNameError:
        rejected.append(candidate)

try:
    audit_archive_member_names(("same.txt", "same.txt"), **policy)
except ArchiveMemberNameError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

try:
    audit_archive_member_names(
        ("one.txt", "two.txt"),
        **(policy | {"max_members": 1}),
    )
except ArchiveMemberNameError:
    member_limit_enforced = True
else:
    member_limit_enforced = False

assert (
    accepted,
    tuple(rejected),
    duplicate_rejected,
    member_limit_enforced,
) == (
    (
        AuditedArchiveMemberName("docs/readme.txt", ("docs", "readme.txt")),
        AuditedArchiveMemberName(
            ".metadata/index-01",
            (".metadata", "index-01"),
        ),
        AuditedArchiveMemberName(
            "data/chunk_0001.bin",
            ("data", "chunk_0001.bin"),
        ),
    ),
    invalid_names,
    True,
    True,
)
```

## Trade-offs and Limitations

The limits cover the number of supplied strings, accepted ASCII bytes in each
name and across all names, segment count, and segment bytes. Because accepted
text is ASCII, character counts and encoded byte counts are identical. The
function preserves input order and rejects only exact duplicates; it does not
detect case-folding, Unicode-normalization, reserved-name, trailing-dot, or
other collisions imposed by a destination platform.

This function audits already-decoded names only. It does not open or parse an
archive, classify member types, inspect links, validate declared or expanded
sizes, read a filesystem, or perform extraction. A successful result is not a
claim that extraction is safe: format-specific filtering, resource limits,
destination isolation, and race-resistant filesystem operations remain
separate requirements.

## Related Snippets

<!-- catalog:related:start -->
- [Audit Symlinks in a Bounded Directory Tree](audit-symlinks-in-a-bounded-directory-tree.md)
- [Build a Deterministic Size-Capped USTAR Archive from Bytes](../configuration-serialization/build-a-deterministic-size-capped-ustar-archive-from-bytes.md)
- [Plan a Bounded Named POSIX Layout Under Explicit Roots](../storage-databases/plan-a-bounded-named-posix-layout-under-explicit-roots.md)
<!-- catalog:related:end -->
