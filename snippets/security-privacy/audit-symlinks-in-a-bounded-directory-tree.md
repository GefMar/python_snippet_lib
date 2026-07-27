---
title: "Audit Symlinks in a Bounded Directory Tree"
snippet_type: recipe
use_cases:
  - resource-management
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../storage-databases/compare-bounded-apparent-sizes-of-two-file-tree-snapshots.md
  - validate-a-conservative-unicode-filename-component.md
---

# Audit Symlinks in a Bounded Directory Tree

## Idea and Problem

Inspect a stable directory without following links and reject any symbolic link whose resolved target is missing or outside the inspected root.

The audit also rejects special files and limits depth, entry count, and total
apparent leaf size. Internal links are classified but never traversed, so a
link cannot make the walk leave the root or repeatedly visit the same subtree.

## When to Use

Use this preflight for a locally controlled resource tree that should contain
ordinary directories, regular files, and optional links to other entries in
the same tree. Run it while the tree is quiescent, before a trusted process
performs a separate packaging or deployment step. Treat all limits as policy
chosen from the expected bundle shape.

Do not use a successful audit as an authorization decision for an actively
mutated, attacker-controlled tree. Path resolution and later file access are
separate operations; the tree can change between them.

## Implementation

```python
import os
import stat
from dataclasses import dataclass
from pathlib import Path


_MAX_ENTRIES = 100_000
_MAX_DEPTH = 128
_MAX_TOTAL_BYTES = 1 << 40


class UnsafeTreeError(ValueError):
    pass


class TreeLimitError(RuntimeError):
    pass


@dataclass(frozen=True, slots=True)
class TreeAudit:
    entries: int
    regular_files: int
    internal_symlinks: int
    apparent_bytes: int


def _bounded_tree_limit(
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


def audit_directory_symlinks(
    root: Path,
    *,
    max_entries: int = 10_000,
    max_depth: int = 32,
    max_total_bytes: int = 256 * 1024 * 1024,
) -> TreeAudit:
    max_entries = _bounded_tree_limit(
        max_entries,
        name="max_entries",
        minimum=1,
        maximum=_MAX_ENTRIES,
    )
    max_depth = _bounded_tree_limit(
        max_depth,
        name="max_depth",
        minimum=0,
        maximum=_MAX_DEPTH,
    )
    max_total_bytes = _bounded_tree_limit(
        max_total_bytes,
        name="max_total_bytes",
        minimum=0,
        maximum=_MAX_TOTAL_BYTES,
    )

    root = Path(root)
    root_metadata = os.stat(root, follow_symlinks=False)
    if not stat.S_ISDIR(root_metadata.st_mode):
        raise UnsafeTreeError("root must be a real directory")
    root_resolved = root.resolve(strict=True)

    entries = 0
    regular_files = 0
    internal_symlinks = 0
    apparent_bytes = 0
    pending = [(root, 0)]

    while pending:
        directory, depth = pending.pop()
        with os.scandir(directory) as children:
            for child in children:
                entries += 1
                if entries > max_entries:
                    raise TreeLimitError("max_entries was exceeded")

                metadata = child.stat(follow_symlinks=False)
                mode = metadata.st_mode
                child_path = Path(child.path)
                if stat.S_ISLNK(mode):
                    try:
                        target = child_path.resolve(strict=True)
                    except (OSError, RuntimeError) as error:
                        raise UnsafeTreeError(
                            "a symbolic link has no stable target"
                        ) from error
                    if not target.is_relative_to(root_resolved):
                        raise UnsafeTreeError(
                            "a symbolic link resolves outside the root"
                        )
                    internal_symlinks += 1
                    apparent_bytes += metadata.st_size
                elif stat.S_ISDIR(mode):
                    child_depth = depth + 1
                    if child_depth > max_depth:
                        raise TreeLimitError("max_depth was exceeded")
                    pending.append((child_path, child_depth))
                elif stat.S_ISREG(mode):
                    regular_files += 1
                    apparent_bytes += metadata.st_size
                else:
                    raise UnsafeTreeError("the tree contains a special file")
                if apparent_bytes > max_total_bytes:
                    raise TreeLimitError("max_total_bytes was exceeded")

    return TreeAudit(
        entries=entries,
        regular_files=regular_files,
        internal_symlinks=internal_symlinks,
        apparent_bytes=apparent_bytes,
    )
```

## Example

```python
from tempfile import TemporaryDirectory


with TemporaryDirectory() as temporary_directory:
    root = Path(temporary_directory) / "bundle"
    data = root / "data"
    data.mkdir(parents=True)
    (data / "note.txt").write_bytes(b"hello")
    (root / "current").symlink_to("data/note.txt")

    audit = audit_directory_symlinks(root, max_entries=8, max_total_bytes=64)
    expected_apparent_bytes = 5 + os.stat(
        root / "current",
        follow_symlinks=False,
    ).st_size

    outside = root.parent / "outside.txt"
    outside.write_bytes(b"private")
    (root / "escape").symlink_to(outside)
    try:
        audit_directory_symlinks(root)
    except UnsafeTreeError as error:
        escaping_link_rejected = "outside" in str(error)
    else:
        escaping_link_rejected = False

assert (audit, escaping_link_rejected) == (
    TreeAudit(
        entries=3,
        regular_files=1,
        internal_symlinks=1,
        apparent_bytes=expected_apparent_bytes,
    ),
    True,
)
```

## Trade-offs and Limitations

The walk is not atomic. A rename, replacement, permission change, or link
retargeting during or after the audit can invalidate its result. For a hostile
mutable tree, consume files through descriptor-relative operations with
platform-appropriate no-follow controls, or first isolate an immutable
snapshot; a path-only preflight cannot close the check/use gap.

Internal links are accepted but not followed, so their targets contribute only
when also reachable as ordinary entries. Hard links and mount boundaries are
not detected. Apparent sizes do not represent allocated disk blocks, and an
individual filesystem call can still block. Any traversal error aborts without
returning a partial audit.

## Related Snippets

<!-- catalog:related:start -->
- [Compare Bounded Apparent Sizes of Two File-Tree Snapshots](../storage-databases/compare-bounded-apparent-sizes-of-two-file-tree-snapshots.md)
- [Validate a Conservative Unicode Filename Component](validate-a-conservative-unicode-filename-component.md)
<!-- catalog:related:end -->
