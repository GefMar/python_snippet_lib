---
title: "Compare Bounded Apparent Sizes of Two File-Tree Snapshots"
snippet_type: recipe
use_cases:
  - automation
  - persistence
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - build-and-apply-a-deterministic-mapping-delta.md
  - check-whether-a-generated-file-is-older-than-its-inputs.md
---

# Compare Bounded Apparent Sizes of Two File-Tree Snapshots

## Idea and Problem

Capture two deterministic file-tree snapshots and report which paths changed after rolling leaf sizes into every parent directory.

The traversal records a flat, path-sorted tuple of regular files, directories,
and symbolic links. Regular files and links contribute their apparent
`st_size`; directories contribute the sum of their descendants. Comparing two
immutable snapshots then exposes added, removed, type-changed, and size-changed
paths without coupling the data to a renderer.

## When to Use

Use this recipe for a trusted, locally mounted tree that is stable for the
duration of each scan and small enough for an in-memory snapshot. It is useful
for bounded diagnostics, before-and-after reports, or quota investigations
where apparent byte size is the intended measure. Set limits from an expected
tree shape before traversing unfamiliar content.

## Implementation

```python
import os
import stat
from dataclasses import dataclass
from pathlib import Path
from typing import Literal


EntryKind = Literal["directory", "file", "symlink"]
ChangeKind = Literal["added", "removed", "type-changed", "size-changed"]


class SnapshotLimitError(RuntimeError):
    pass


@dataclass(frozen=True, slots=True)
class SizeEntry:
    path: str
    kind: EntryKind
    size: int


@dataclass(frozen=True, slots=True)
class SizeSnapshot:
    entries: tuple[SizeEntry, ...]

    @property
    def size(self) -> int:
        return next(entry.size for entry in self.entries if entry.path == ".")


@dataclass(frozen=True, slots=True)
class SizeChange:
    path: str
    change: ChangeKind
    before_kind: EntryKind | None
    after_kind: EntryKind | None
    before_size: int | None
    after_size: int | None


def snapshot_apparent_sizes(
    root: Path,
    *,
    max_entries: int = 10_000,
    max_depth: int = 64,
) -> SizeSnapshot:
    for name, value, lower, upper in (
        ("max_entries", max_entries, 1, 1_000_000),
        ("max_depth", max_depth, 0, 128),
    ):
        if isinstance(value, bool) or not isinstance(value, int):
            raise TypeError(f"{name} must be an integer")
        if not lower <= value <= upper:
            raise ValueError(f"{name} must be between {lower} and {upper}")

    root = Path(root)
    root_stat = os.stat(root, follow_symlinks=False)
    if not stat.S_ISDIR(root_stat.st_mode):
        raise ValueError("root must be a directory, not a link or special file")

    raw: dict[str, tuple[EntryKind, int]] = {}

    def visit(path: Path, parts: tuple[str, ...], depth: int) -> None:
        if len(raw) >= max_entries:
            raise SnapshotLimitError("max_entries was exceeded")

        metadata = os.stat(path, follow_symlinks=False)
        mode = metadata.st_mode
        relative = "/".join(parts) if parts else "."
        if stat.S_ISDIR(mode):
            kind: EntryKind = "directory"
            direct_size = 0
        elif stat.S_ISREG(mode):
            kind = "file"
            direct_size = metadata.st_size
        elif stat.S_ISLNK(mode):
            kind = "symlink"
            direct_size = metadata.st_size
        else:
            raise ValueError(f"unsupported special file: {relative}")
        raw[relative] = (kind, direct_size)

        if kind != "directory":
            return
        with os.scandir(path) as directory:
            for child in directory:
                if depth >= max_depth:
                    raise SnapshotLimitError("max_depth was exceeded")
                visit(Path(child.path), (*parts, child.name), depth + 1)

    visit(root, (), 0)

    cumulative = {path: size for path, (_, size) in raw.items()}
    for path, (kind, size) in raw.items():
        if kind == "directory":
            continue
        parts = path.split("/") if path != "." else []
        while parts:
            parts.pop()
            parent = "/".join(parts) if parts else "."
            cumulative[parent] += size

    return SizeSnapshot(
        entries=tuple(
            SizeEntry(path=path, kind=raw[path][0], size=cumulative[path])
            for path in sorted(raw)
        ),
    )


def compare_size_snapshots(
    before: SizeSnapshot,
    after: SizeSnapshot,
) -> tuple[SizeChange, ...]:
    before_by_path = {entry.path: entry for entry in before.entries}
    after_by_path = {entry.path: entry for entry in after.entries}
    changes: list[SizeChange] = []
    for path in sorted(before_by_path.keys() | after_by_path.keys()):
        old = before_by_path.get(path)
        new = after_by_path.get(path)
        if old is None:
            change: ChangeKind = "added"
        elif new is None:
            change = "removed"
        elif old.kind != new.kind:
            change = "type-changed"
        elif old.size != new.size:
            change = "size-changed"
        else:
            continue
        changes.append(
            SizeChange(
                path=path,
                change=change,
                before_kind=old.kind if old else None,
                after_kind=new.kind if new else None,
                before_size=old.size if old else None,
                after_size=new.size if new else None,
            ),
        )
    return tuple(changes)
```

## Example

```python
from tempfile import TemporaryDirectory


with TemporaryDirectory() as temporary_directory:
    root = Path(temporary_directory)
    (root / "sub").mkdir()
    (root / "a.bin").write_bytes(b"aaa")
    (root / "sub" / "b.bin").write_bytes(b"bb")
    before = snapshot_apparent_sizes(root)

    (root / "a.bin").write_bytes(b"aaaaa")
    (root / "sub" / "b.bin").unlink()
    (root / "sub" / "c.bin").write_bytes(b"cccc")
    after = snapshot_apparent_sizes(root)

changes = compare_size_snapshots(before, after)
summary = tuple(
    (change.path, change.change, change.before_size, change.after_size)
    for change in changes
)

assert (before.size, after.size, summary) == (
    5,
    9,
    (
        (".", "size-changed", 5, 9),
        ("a.bin", "size-changed", 3, 5),
        ("sub", "size-changed", 2, 4),
        ("sub/b.bin", "removed", 2, None),
        ("sub/c.bin", "added", None, 4),
    ),
)
```

## Trade-offs and Limitations

Snapshots require `O(n)` memory and sorting costs `O(n log n)`. Sparse files
use apparent `st_size`, not allocated blocks; hard links are counted once per
path; link sizes describe the stored link text. This is therefore not a `du`
replacement. The scan is read-only but not atomic: concurrent renames,
truncation, permission changes, or mount behavior can produce an error or a
mixed-time view. Any traversal error aborts without returning partial data.
The function never follows symlinks and rejects sockets, devices, and other
special files.

## Related Snippets

<!-- catalog:related:start -->
- [Build and Apply a Deterministic Mapping Delta](build-and-apply-a-deterministic-mapping-delta.md)
- [Check Whether a Generated File Is Older Than Its Inputs](check-whether-a-generated-file-is-older-than-its-inputs.md)
<!-- catalog:related:end -->
