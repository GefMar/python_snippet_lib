---
title: "Fingerprint a Bounded Flat File Set with Framed SHA-256"
snippet_type: recipe
use_cases:
  - caching
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - store-bytes-by-their-content-digest.md
  - compare-bounded-apparent-sizes-of-two-file-tree-snapshots.md
  - ../configuration-serialization/fingerprint-a-set-like-json-array-deterministically.md
---

# Fingerprint a Bounded Flat File Set with Framed SHA-256

## Idea and Problem

Hash a stable flat directory deterministically by sorting file names and framing every name, size, and streamed content before updating SHA-256.

Including names detects renames, while explicit lengths keep different file
boundaries unambiguous. The scan accepts direct regular files only, enforces
file and byte limits before reading, and verifies each opened file against the
metadata observed during discovery.

## When to Use

Use this recipe as a local change fingerprint for a small, quiescent bundle of
direct files, such as deciding whether a generated artifact is stale. Define
the directory membership and file-name encoding as part of the format, and
keep the versioned domain marker unchanged for every producer that must agree.

Use a recursive manifest when subdirectories are meaningful. Use a keyed MAC
or digital signature when an attacker can choose the files or fingerprint;
an unkeyed digest detects changes but does not authenticate their origin.

## Implementation

```python
import os
import stat
from dataclasses import dataclass
from hashlib import sha256
from pathlib import Path


_FINGERPRINT_DOMAIN = b"flat-file-set-sha256-v1\x00"
_MAX_FILES = 100_000
_MAX_FILE_BYTES = 1 << 40
_MAX_TOTAL_BYTES = 1 << 44
_MAX_CHUNK_BYTES = 1024 * 1024


class FileSetError(ValueError):
    pass


class FileSetLimitError(FileSetError):
    pass


@dataclass(frozen=True, slots=True)
class FileSetFingerprint:
    files: int
    total_bytes: int
    sha256_hex: str


@dataclass(frozen=True, slots=True)
class _DiscoveredFile:
    name: str
    encoded_name: bytes
    device: int
    inode: int
    size: int


def _bounded_file_set_integer(
    value: int,
    *,
    name: str,
    minimum: int,
    maximum: int,
) -> int:
    if isinstance(value, bool) or not isinstance(value, int):
        raise TypeError(f"{name} must be an integer")
    if not minimum <= value <= maximum:
        raise ValueError(f"{name} is outside the supported range")
    return value


def fingerprint_flat_file_set(
    root: Path,
    *,
    max_files: int = 10_000,
    max_file_bytes: int = 256 * 1024 * 1024,
    max_total_bytes: int = 1024 * 1024 * 1024,
    chunk_bytes: int = 64 * 1024,
) -> FileSetFingerprint:
    max_files = _bounded_file_set_integer(
        max_files,
        name="max_files",
        minimum=0,
        maximum=_MAX_FILES,
    )
    max_file_bytes = _bounded_file_set_integer(
        max_file_bytes,
        name="max_file_bytes",
        minimum=0,
        maximum=_MAX_FILE_BYTES,
    )
    max_total_bytes = _bounded_file_set_integer(
        max_total_bytes,
        name="max_total_bytes",
        minimum=0,
        maximum=_MAX_TOTAL_BYTES,
    )
    chunk_bytes = _bounded_file_set_integer(
        chunk_bytes,
        name="chunk_bytes",
        minimum=1,
        maximum=_MAX_CHUNK_BYTES,
    )

    root = Path(root)
    root_metadata = os.stat(root, follow_symlinks=False)
    if not stat.S_ISDIR(root_metadata.st_mode):
        raise FileSetError("root must be a real directory")

    discovered: list[_DiscoveredFile] = []
    total_bytes = 0
    with os.scandir(root) as entries:
        for entry in entries:
            if len(discovered) >= max_files:
                raise FileSetLimitError("max_files was exceeded")
            metadata = entry.stat(follow_symlinks=False)
            if not stat.S_ISREG(metadata.st_mode):
                raise FileSetError(
                    "the flat file set contains a link, directory, or special file"
                )
            if metadata.st_size > max_file_bytes:
                raise FileSetLimitError("max_file_bytes was exceeded")
            total_bytes += metadata.st_size
            if total_bytes > max_total_bytes:
                raise FileSetLimitError("max_total_bytes was exceeded")
            try:
                encoded_name = entry.name.encode("utf-8", errors="strict")
            except UnicodeEncodeError as error:
                raise FileSetError("a file name is not valid Unicode text") from error
            discovered.append(
                _DiscoveredFile(
                    name=entry.name,
                    encoded_name=encoded_name,
                    device=metadata.st_dev,
                    inode=metadata.st_ino,
                    size=metadata.st_size,
                )
            )

    discovered.sort(key=lambda item: item.encoded_name)
    digest = sha256()
    digest.update(_FINGERPRINT_DOMAIN)
    digest.update(len(discovered).to_bytes(4, "big"))

    for item in discovered:
        digest.update(len(item.encoded_name).to_bytes(4, "big"))
        digest.update(item.encoded_name)
        digest.update(item.size.to_bytes(8, "big"))

        with (root / item.name).open("rb") as stream:
            opened = os.fstat(stream.fileno())
            if (
                not stat.S_ISREG(opened.st_mode)
                or opened.st_dev != item.device
                or opened.st_ino != item.inode
                or opened.st_size != item.size
            ):
                raise FileSetError("a file changed between discovery and opening")

            remaining = item.size
            while remaining:
                chunk = stream.read(min(remaining, chunk_bytes))
                if not chunk:
                    raise FileSetError("a file shrank while it was being read")
                digest.update(chunk)
                remaining -= len(chunk)
            if stream.read(1):
                raise FileSetError("a file grew while it was being read")
            after = os.fstat(stream.fileno())
            if (
                after.st_dev != item.device
                or after.st_ino != item.inode
                or after.st_size != item.size
            ):
                raise FileSetError("a file changed while it was being read")

    return FileSetFingerprint(
        files=len(discovered),
        total_bytes=total_bytes,
        sha256_hex=digest.hexdigest(),
    )
```

## Example

```python
from tempfile import TemporaryDirectory


with TemporaryDirectory() as temporary_directory:
    base = Path(temporary_directory)
    first_root = base / "first"
    second_root = base / "second"
    first_root.mkdir()
    second_root.mkdir()

    (first_root / "b.txt").write_bytes(b"two")
    (first_root / "a.txt").write_bytes(b"one")
    (second_root / "a.txt").write_bytes(b"one")
    (second_root / "b.txt").write_bytes(b"two")

    first = fingerprint_flat_file_set(first_root)
    second = fingerprint_flat_file_set(second_root)
    (second_root / "a.txt").rename(second_root / "renamed.txt")
    renamed = fingerprint_flat_file_set(second_root)

assert (
    first.files,
    first.total_bytes,
    first.sha256_hex == second.sha256_hex,
    first.sha256_hex != renamed.sha256_hex,
) == (2, 6, True, True)
```

## Trade-offs and Limitations

Discovery stores one bounded metadata record per file and sorting costs
`O(n log n)`; content hashing then costs `O(total bytes)`. SHA-256 is slower
than a non-cryptographic checksum, but its framing and collision resistance
make accidental ambiguity less likely. The digest is still unkeyed and cannot
prove that a hostile directory or claimed fingerprint is trustworthy.

The scan is not a filesystem snapshot. Identity and size checks catch common
replacement, truncation, and growth races, but an in-place rewrite that keeps
the same inode and size can produce a mixed-time digest. Require a stable tree
or snapshot it first. This format rejects subdirectories, links, special files,
and file names that cannot be encoded as UTF-8; changing those policies or the
domain marker creates a different fingerprint format.

## Related Snippets

<!-- catalog:related:start -->
- [Store Bytes by Their Content Digest](store-bytes-by-their-content-digest.md)
- [Compare Bounded Apparent Sizes of Two File-Tree Snapshots](compare-bounded-apparent-sizes-of-two-file-tree-snapshots.md)
- [Fingerprint a Set-Like JSON Array Deterministically](../configuration-serialization/fingerprint-a-set-like-json-array-deterministically.md)
<!-- catalog:related:end -->
