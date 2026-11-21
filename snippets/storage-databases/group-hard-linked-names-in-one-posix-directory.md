---
title: "Group Hard-Linked Names in One POSIX Directory"
snippet_type: recipe
use_cases:
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-bounded-file-manifest-with-internal-symlink-aliases.md
  - compare-bounded-apparent-sizes-of-two-file-tree-snapshots.md
---

# Group Hard-Linked Names in One POSIX Directory

## Idea and Problem

Find which regular-file names in one directory refer to the same filesystem object without reading file contents.

On POSIX filesystems with stable inode information, the device and inode
number together identify a file. Look up each directory entry without
following symlinks and collect names with the same identity. Keep only groups
with at least two names in this directory.

## When to Use

Use this to explain why changes through one path are visible through another
in a small, trusted, stable directory. The scan accepts ordinary files directly
under the directory and makes no changes. For content deduplication, compare
file contents instead.

## Implementation

```python
import os
import stat
from collections import defaultdict
from pathlib import Path


def hard_link_groups(directory: Path) -> tuple[tuple[str, ...], ...]:
    by_identity: dict[tuple[int, int], list[str]] = defaultdict(list)
    with os.scandir(directory) as entries:
        for entry in entries:
            details = entry.stat(follow_symlinks=False)
            if stat.S_ISREG(details.st_mode):
                by_identity[details.st_dev, details.st_ino].append(entry.name)

    return tuple(sorted(tuple(sorted(names)) for names in by_identity.values() if len(names) > 1))
```

## Example

```python
from tempfile import TemporaryDirectory

with TemporaryDirectory() as temporary:
    directory = Path(temporary)
    primary = directory / "primary.txt"
    primary.write_text("same bytes", encoding="utf-8")
    os.link(primary, directory / "alias.txt")
    (directory / "symbolic.txt").symlink_to(primary.name)
    (directory / "copy.txt").write_bytes(primary.read_bytes())

    subdirectory = directory / "nested"
    subdirectory.mkdir()
    os.link(primary, subdirectory / "also-linked.txt")
    observed = hard_link_groups(directory)

assert observed == (("alias.txt", "primary.txt"),)
```

## Trade-offs and Limitations

The scan stores the regular-file names and sorts the output groups. It finds
only names directly under the supplied path. A link outside the directory
will have no effect on the result. The starting path must already identify
the intended directory. Operational errors propagate, and the scan does not
provide a snapshot under concurrent modification.

The output describes shared identity, not how much disk space deletion would
recover. Filesystem features such as snapshots and copy-on-write allocation
affect that calculation.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Bounded File Manifest with Internal Symlink Aliases](build-a-bounded-file-manifest-with-internal-symlink-aliases.md)
- [Compare Bounded Apparent Sizes of Two File-Tree Snapshots](compare-bounded-apparent-sizes-of-two-file-tree-snapshots.md)
<!-- catalog:related:end -->
