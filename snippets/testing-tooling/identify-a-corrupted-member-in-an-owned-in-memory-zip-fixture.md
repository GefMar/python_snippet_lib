---
title: "Identify a Corrupted Member in an Owned In-Memory ZIP Fixture"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../security-privacy/audit-a-bounded-zip-central-directory-without-decompressing-members.md
  - ../storage-databases/recover-a-verified-prefix-from-a-bounded-crc-framed-byte-log.md
---

# Identify a Corrupted Member in an Owned In-Memory ZIP Fixture

## Idea and Problem

Check member data in a small test-owned ZIP and identify the first damaged member without extracting files.

A ZIP can open and list its contents even when a member's stored data no
longer matches its CRC. `ZipFile.testzip()` reads member data and returns
the name of the first bad member, or `None` if the check passes.

## When to Use

Use this with synthetic ZIP bytes created by your own tests. The example
limits the archive to 64 KiB and accepts only stored members with unique
names. This is useful when validating a generated fixture or testing how
your code reports accidental damage to its bytes.

## Implementation

```python
from io import BytesIO
from zipfile import ZIP_STORED, ZipFile


MAX_FIXTURE_BYTES = 64 * 1024


def first_damaged_member(data: bytes) -> str | None:
    if not 1 <= len(data) <= MAX_FIXTURE_BYTES:
        raise ValueError("fixture size must be between 1 byte and 64 KiB")

    with BytesIO(data) as buffer, ZipFile(buffer) as archive:
        members = archive.infolist()
        if len({member.filename for member in members}) != len(members):
            raise ValueError("fixture member names must be unique")
        if any(member.compress_type != ZIP_STORED for member in members):
            raise ValueError("fixture members must use ZIP_STORED")
        return archive.testzip()
```

## Example

```python
from zipfile import BadZipFile


payload = b"region,count\nnorth,7\n"
with BytesIO() as buffer:
    with ZipFile(buffer, "w", compression=ZIP_STORED) as archive:
        archive.writestr("summary.txt", b"Sample region report\n")
        archive.writestr("rows.csv", payload)
    original = buffer.getvalue()

assert original.count(payload) == 1
# Change only the synthetic member payload, leaving its recorded CRC intact.
damaged = original.replace(payload, b"region,count\nnorth,8\n", 1)

try:
    first_damaged_member(b"not a ZIP container")
except BadZipFile:
    malformed_rejected = True
else:
    malformed_rejected = False

assert (first_damaged_member(original), first_damaged_member(damaged), malformed_rejected) == (
    None,
    "rows.csv",
    True,
)
```

## Trade-offs and Limitations

The check reads member data and stops at the first bad member. A malformed
container can fail while opening the archive, and other read errors can
propagate. Keep the fixture small and use stored data so the test can change
one payload without depending on a compression algorithm.

The CRC check can detect accidental corruption. It does not establish
the source of the archive, validate the meaning of the files, or make archive
paths safe for extraction. This recipe performs no extraction.

## Related Snippets

<!-- catalog:related:start -->
- [Audit a Bounded ZIP Central Directory Without Decompressing Members](../security-privacy/audit-a-bounded-zip-central-directory-without-decompressing-members.md)
- [Recover a Verified Prefix from a Bounded CRC-Framed Byte Log](../storage-databases/recover-a-verified-prefix-from-a-bounded-crc-framed-byte-log.md)
<!-- catalog:related:end -->
