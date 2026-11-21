---
title: "Read Every Named ZIP Member by ZipInfo When Names Repeat"
snippet_type: recipe
use_cases:
  - parsing
  - persistence
tested_python:
  - "3.14"
dependencies: []
related:
  - ../security-privacy/audit-a-bounded-zip-central-directory-without-decompressing-members.md
  - ../testing-tooling/identify-a-corrupted-member-in-an-owned-in-memory-zip-fixture.md
---

# Read Every Named ZIP Member by ZipInfo When Names Repeat

## Idea and Problem

Read every ZIP member with an exact matching name, including repeated names, in archive order.

Use `infolist()` to get each entry's `ZipInfo`. Open the individual entry
object to read its own bytes. Looking up a string name selects a single entry
even when the archive contains several entries with that name.

## When to Use

Use this for a small trusted ZIP payload already in memory, such as a report
archive that repeats member names. Supply limits for the archive bytes and
for the total bytes to return. Use an exact archive name and treat each
matching nondirectory entry as a separate result.

## Implementation

```python
import sys
from io import BytesIO
from zipfile import ZipFile


def read_named_zip_members(
    payload: bytes,
    name: str,
    *,
    archive_limit: int,
    returned_limit: int,
) -> tuple[bytes, ...]:
    if archive_limit < 0 or not 0 <= returned_limit < sys.maxsize:
        raise ValueError("invalid ZIP size limits")
    if len(payload) > archive_limit:
        raise ValueError("ZIP payload exceeds the archive limit")

    remaining = returned_limit
    found: list[bytes] = []
    with BytesIO(payload) as source, ZipFile(source) as archive:
        for info in archive.infolist():
            if info.filename != name or info.is_dir():
                continue
            with archive.open(info) as entry:
                data = entry.read(remaining + 1)
            if len(data) > remaining:
                raise ValueError("matching ZIP members exceed the returned byte limit")
            found.append(data)
            remaining -= len(data)
    return tuple(found)
```

## Example

```python
import warnings

with BytesIO() as output:
    with ZipFile(output, "w") as archive:
        archive.writestr("summary.txt", b"morning")
        archive.writestr("notes.txt", b"unrelated")
        with warnings.catch_warnings():
            warnings.filterwarnings("ignore", message="Duplicate name:", category=UserWarning)
            archive.writestr("summary.txt", b"evening")
    fixture = output.getvalue()

assert (
    read_named_zip_members(fixture, "summary.txt", archive_limit=1024, returned_limit=14),
    read_named_zip_members(fixture, "missing.txt", archive_limit=1024, returned_limit=0),
) == ((b"morning", b"evening"), ())
```

## Trade-offs and Limitations

The function reads matching members into memory and closes their streams
before returning. It opens no output paths. The reader still parses metadata
for the entire archive and can buffer extra data while decompressing. Use the
size limits to bound inputs and returned values for this small archive use
case. Untrusted archives require a wider resource and validation policy.

The result follows the listed entry order and keeps duplicates separate.
When the caller needs one specific occurrence, select its entry explicitly
using additional archive metadata or a positional rule.

## Related Snippets

<!-- catalog:related:start -->
- [Audit a Bounded ZIP Central Directory Without Decompressing Members](../security-privacy/audit-a-bounded-zip-central-directory-without-decompressing-members.md)
- [Identify a Corrupted Member in an Owned In-Memory ZIP Fixture](../testing-tooling/identify-a-corrupted-member-in-an-owned-in-memory-zip-fixture.md)
<!-- catalog:related:end -->
