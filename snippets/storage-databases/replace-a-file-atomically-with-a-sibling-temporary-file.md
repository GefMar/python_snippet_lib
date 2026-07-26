---
title: "Replace a File Atomically with a Sibling Temporary File"
snippet_type: recipe
use_cases:
  - persistence
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - store-bytes-by-their-content-digest.md
  - ../algorithms-data-structures/sort-newline-terminated-binary-records-with-bounded-merge-passes.md
---

# Replace a File Atomically with a Sibling Temporary File

## Idea and Problem

Write and synchronize a sibling temporary file before replacing the target so readers never observe partially written content.

Creating the temporary in the target directory keeps both paths on the same
filesystem for `os.replace`. If writing, flushing, or synchronization fails,
the old target remains in place and the temporary is removed.

## When to Use

Use this recipe for small state files, generated configuration, or other local
artifacts that one process writes as a complete new version. The parent
directory must already exist and be trusted. Choose explicit permissions for a
new file; an existing regular file keeps its ordinary user/group/other
permission bits. Use a
database or a lock when several related files or concurrent writers require a
transaction or conflict detection.

## Implementation

```python
import os
import stat
import tempfile
from collections.abc import Iterator
from contextlib import contextmanager
from pathlib import Path
from typing import BinaryIO, Literal, TextIO


def _replacement_permissions(target: Path, new_permissions: int) -> int:
    if isinstance(new_permissions, bool) or not isinstance(new_permissions, int):
        raise TypeError("new_permissions must be an integer")
    if not 0 <= new_permissions <= 0o777:
        raise ValueError("new_permissions must be between 0o000 and 0o777")

    try:
        existing = target.lstat()
    except FileNotFoundError:
        return new_permissions
    if not stat.S_ISREG(existing.st_mode):
        raise ValueError("target must be absent or a regular file")
    return stat.S_IMODE(existing.st_mode) & 0o777


@contextmanager
def atomic_file_writer(
    path: str | os.PathLike[str],
    *,
    mode: Literal["w", "wb"] = "w",
    encoding: str = "utf-8",
    new_permissions: int = 0o600,
) -> Iterator[TextIO | BinaryIO]:
    if mode not in {"w", "wb"}:
        raise ValueError("mode must be 'w' or 'wb'")

    target = Path(path)
    parent = target.parent
    if not parent.is_dir():
        raise FileNotFoundError("target parent directory does not exist")
    permissions = _replacement_permissions(target, new_permissions)

    descriptor, temporary_name = tempfile.mkstemp(
        prefix=f".{target.name}.",
        suffix=".tmp",
        dir=parent,
    )
    temporary = Path(temporary_name)
    published = False
    try:
        os.chmod(temporary, permissions)
        open_options = {} if mode == "wb" else {"encoding": encoding, "newline": ""}
        stream = os.fdopen(descriptor, mode, **open_options)
        descriptor = -1
        with stream:
            yield stream
            stream.flush()
            os.fsync(stream.fileno())

        os.replace(temporary, target)
        published = True
    finally:
        if descriptor >= 0:
            os.close(descriptor)
        if not published:
            temporary.unlink(missing_ok=True)
```

## Example

```python
import stat
from pathlib import Path
from tempfile import TemporaryDirectory


with TemporaryDirectory() as directory:
    root = Path(directory)
    target = root / "state.txt"
    target.write_text("old\n", encoding="utf-8")
    target.chmod(0o604)

    with atomic_file_writer(target) as stream:
        unchanged_while_open = target.read_text(encoding="utf-8")
        stream.write("new\n")

    try:
        with atomic_file_writer(target) as stream:
            stream.write("broken\n")
            raise RuntimeError("stop before publication")
    except RuntimeError:
        pass

    binary_target = root / "payload.bin"
    with atomic_file_writer(
        binary_target,
        mode="wb",
        new_permissions=0o640,
    ) as stream:
        stream.write(b"\x00\x01")

    leftovers = list(root.glob(f".{target.name}.*.tmp"))
    observed = (
        unchanged_while_open,
        target.read_text(encoding="utf-8"),
        stat.S_IMODE(target.stat().st_mode),
        binary_target.read_bytes(),
        stat.S_IMODE(binary_target.stat().st_mode),
        leftovers,
    )

assert observed == ("old\n", "new\n", 0o604, b"\x00\x01", 0o640, [])
```

## Trade-offs and Limitations

This provides atomic reader-visible replacement only where `os.replace` has
the required filesystem semantics. Synchronizing the temporary file does not
make the directory entry crash-durable; systems that need power-loss guarantees
must add a platform-specific parent-directory synchronization step. Concurrent
writers still use last-writer-wins behavior, and no compare-and-swap check is
performed. Existing symlinks and other non-regular targets are rejected. The
recipe changes one path only and does not preserve special set-id/sticky mode
bits, ownership, extended attributes, access-control lists, or timestamps.

## Related Snippets

<!-- catalog:related:start -->
- [Store Bytes by Their Content Digest](store-bytes-by-their-content-digest.md)
- [Sort Newline-Terminated Binary Records with Bounded Merge Passes](../algorithms-data-structures/sort-newline-terminated-binary-records-with-bounded-merge-passes.md)
<!-- catalog:related:end -->
