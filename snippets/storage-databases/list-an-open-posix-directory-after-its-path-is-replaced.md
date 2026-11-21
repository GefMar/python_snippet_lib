---
title: "List an Open POSIX Directory After Its Path Is Replaced"
snippet_type: recipe
use_cases:
  - persistence
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - group-hard-linked-names-in-one-posix-directory.md
  - ../security-privacy/read-one-descriptor-relative-regular-file-without-following-its-final-symlink.md
---

# List an Open POSIX Directory After Its Path Is Replaced

## Idea and Problem

Keep listing the same local directory when a new directory takes over its old pathname.

Resolve the path once by opening the directory. A subsequent `os.listdir`
call can use that descriptor, so it enumerates the original directory
object even after a rename. A new path-based listing resolves the name again
and sees the replacement.

## When to Use

Use this for a trusted local directory when a cooperating process may rename
it, such as when rotating an output directory. The platform must support
`O_DIRECTORY` and descriptor arguments to `os.listdir`. Borrow the descriptor
within the scope and let the scope close it.

## Implementation

```python
import os
from collections.abc import Iterator
from contextlib import contextmanager
from pathlib import Path


@contextmanager
def open_directory(path: Path) -> Iterator[int]:
    if not hasattr(os, "O_DIRECTORY") or os.listdir not in os.supports_fd:
        raise NotImplementedError("directory descriptor listing is unavailable")

    descriptor = os.open(path, os.O_RDONLY | os.O_DIRECTORY)
    try:
        yield descriptor
    finally:
        os.close(descriptor)
```

## Example

```python
from errno import EBADF
from tempfile import TemporaryDirectory

with TemporaryDirectory() as directory:
    active = Path(directory) / "active"
    active.mkdir()
    (active / "first.txt").touch()

    with open_directory(active) as descriptor:
        retired = active.rename(active.with_name("retired"))
        active.mkdir()
        (active / "second.txt").touch()

        original_names = sorted(os.listdir(descriptor))
        replacement_names = sorted(os.listdir(active))
        (retired / "added.txt").touch()
        updated_original_names = sorted(os.listdir(descriptor))

    closed = False
    try:
        os.fstat(descriptor)
    except OSError as error:
        closed = error.errno == EBADF

assert (original_names, replacement_names, updated_original_names, closed) == (
    ["first.txt"],
    ["second.txt"],
    ["added.txt", "first.txt"],
    True,
)
```

## Trade-offs and Limitations

The descriptor preserves directory identity. Entries can still change in
that directory, even between two calls on the same descriptor. The order of
names from `os.listdir` is unspecified, so sort them when stable presentation
is useful.

Path resolution still occurs normally at the initial open. Use a separate
path policy for untrusted paths. The scope holds one descriptor until exit,
so keep it short and use each listing while its directory view is relevant.

## Related Snippets

<!-- catalog:related:start -->
- [Group Hard-Linked Names in One POSIX Directory](group-hard-linked-names-in-one-posix-directory.md)
- [Read One Descriptor-Relative Regular File Without Following Its Final Symlink](../security-privacy/read-one-descriptor-relative-regular-file-without-following-its-final-symlink.md)
<!-- catalog:related:end -->
