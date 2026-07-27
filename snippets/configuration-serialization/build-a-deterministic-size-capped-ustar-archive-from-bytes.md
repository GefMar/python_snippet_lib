---
title: "Build a Deterministic Size-Capped USTAR Archive from Bytes"
snippet_type: recipe
use_cases:
  - resource-management
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../security-privacy/validate-a-conservative-unicode-filename-component.md
  - ../storage-databases/split-a-binary-stream-into-exclusively-created-numbered-parts.md
  - ../storage-databases/store-bytes-by-their-content-digest.md
---

# Build a Deterministic Size-Capped USTAR Archive from Bytes

## Idea and Problem

Build one reproducible USTAR payload from explicit in-memory files only after proving its final padded size fits a fixed budget.

Filesystem archiving can capture changing metadata, recurse unexpectedly, and
include links or special files. Supplying named immutable bytes instead makes
the input closed and testable. Sorting safe names, fixing every metadata field,
and counting 512-byte USTAR blocks before allocation produces stable archive
bytes with an exact upper bound.

## When to Use

Use this builder for small generated bundles whose complete contents already
exist in memory and must be reproducible across input order. Keep the strict
ASCII path grammar if consumers need portable, predictable member names. Use a
streaming archive API or a dedicated packaging format for large data,
filesystem snapshots, compression, executable metadata, or Unicode paths.

## Implementation

```python
import io
import re
import tarfile
from collections.abc import Iterable


_TAR_BLOCK_SIZE = 512
_TAR_RECORD_SIZE = 10_240
_MAX_ARCHIVE_BYTES = 16 * 1024 * 1024
_MAX_MEMBERS = 64
_MAX_MEMBER_BYTES = 8 * 1024 * 1024
_SAFE_COMPONENT = re.compile(r"[A-Za-z0-9._-]+", re.ASCII)


class ArchiveSizeError(ValueError):
    pass


def _validate_member_name(name: str) -> str:
    if not isinstance(name, str):
        raise TypeError("member names must be strings")
    try:
        encoded = name.encode("ascii")
    except UnicodeEncodeError as error:
        raise ValueError("member names must be ASCII") from error
    if not 1 <= len(encoded) <= 100:
        raise ValueError("member name length is outside the supported range")
    if name.startswith("/") or name.endswith("/") or "\\" in name:
        raise ValueError("member name must be a relative POSIX file path")

    components = name.split("/")
    if any(
        component in {"", ".", ".."}
        or _SAFE_COMPONENT.fullmatch(component) is None
        for component in components
    ):
        raise ValueError("member name contains an unsafe path component")
    return name


def _predicted_ustar_size(entries: list[tuple[str, bytes]]) -> int:
    blocks = 2
    for _, data in entries:
        data_blocks = (len(data) + _TAR_BLOCK_SIZE - 1) // _TAR_BLOCK_SIZE
        blocks += 1 + data_blocks
    records = (blocks + 19) // 20
    return records * _TAR_RECORD_SIZE


def build_ustar_archive(
    members: Iterable[tuple[str, bytes]],
    *,
    max_archive_bytes: int,
) -> bytes:
    if (
        isinstance(max_archive_bytes, bool)
        or not isinstance(max_archive_bytes, int)
    ):
        raise TypeError("max_archive_bytes must be an integer")
    if not _TAR_RECORD_SIZE <= max_archive_bytes <= _MAX_ARCHIVE_BYTES:
        raise ValueError("max_archive_bytes is outside the supported range")

    entries: list[tuple[str, bytes]] = []
    names: set[str] = set()
    for member in members:
        if len(entries) >= _MAX_MEMBERS:
            raise ValueError("members exceed the supported count")
        if not isinstance(member, tuple) or len(member) != 2:
            raise TypeError("each member must be a name-data tuple")
        raw_name, data = member
        name = _validate_member_name(raw_name)
        if not isinstance(data, bytes):
            raise TypeError("member data must be immutable bytes")
        if len(data) > _MAX_MEMBER_BYTES:
            raise ArchiveSizeError("member data exceeds the supported size")
        if name in names:
            raise ValueError("member names must be unique")
        names.add(name)
        entries.append((name, data))

    entries.sort(key=lambda entry: entry[0])
    predicted_size = _predicted_ustar_size(entries)
    if predicted_size > max_archive_bytes:
        raise ArchiveSizeError("archive exceeds max_archive_bytes")

    output = io.BytesIO()
    with tarfile.open(
        fileobj=output,
        mode="w",
        format=tarfile.USTAR_FORMAT,
        encoding="ascii",
        errors="strict",
    ) as archive:
        for name, data in entries:
            info = tarfile.TarInfo(name)
            info.type = tarfile.REGTYPE
            info.size = len(data)
            info.mode = 0o644
            info.mtime = 0
            info.uid = 0
            info.gid = 0
            info.uname = ""
            info.gname = ""
            archive.addfile(info, io.BytesIO(data))

    payload = output.getvalue()
    if len(payload) != predicted_size:
        raise RuntimeError("tarfile produced an unexpected archive size")
    return payload
```

## Example

```python
entries = [
    ("docs/readme.txt", b"example\n"),
    ("config/settings.json", b'{"enabled":true}\n'),
]
first = build_ustar_archive(entries, max_archive_bytes=10_240)
second = build_ustar_archive(
    list(reversed(entries)),
    max_archive_bytes=10_240,
)

with tarfile.open(fileobj=io.BytesIO(first), mode="r:") as archive:
    names = archive.getnames()
    members = archive.getmembers()
    readme_file = archive.extractfile("docs/readme.txt")
    if readme_file is None:
        raise AssertionError("expected a regular file")
    readme = readme_file.read()

assert (
    first == second,
    len(first),
    names,
    readme,
    {(member.mode, member.mtime, member.uid, member.gid) for member in members},
) == (
    True,
    10_240,
    ["config/settings.json", "docs/readme.txt"],
    b"example\n",
    {(0o644, 0, 0, 0)},
)
```

## Trade-offs and Limitations

The cap covers the final uncompressed archive, including headers, end markers,
and record padding. USTAR's fixed blocks make even an empty archive 10,240
bytes, and the conservative 100-byte ASCII name limit gives up the format's
prefix field and Unicode extensions. Input contents are already resident in
memory, and the completed archive creates a second bounded allocation.

Only regular files can be emitted: there is no filesystem walk, directory
entry, executable mode, link, device, compression, or preservation of source
metadata. Safe construction does not make arbitrary archive extraction safe.
Consumers must still isolate the destination, impose extraction resource
limits, and use the standard library's restrictive extraction filter when
reading data they do not fully trust.

## Related Snippets

<!-- catalog:related:start -->
- [Validate a Conservative Unicode Filename Component](../security-privacy/validate-a-conservative-unicode-filename-component.md)
- [Split a Binary Stream into Exclusively Created Numbered Parts](../storage-databases/split-a-binary-stream-into-exclusively-created-numbered-parts.md)
- [Store Bytes by Their Content Digest](../storage-databases/store-bytes-by-their-content-digest.md)
<!-- catalog:related:end -->
