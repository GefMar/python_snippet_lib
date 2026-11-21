---
title: "Prune Named Generated Subtrees with Top-Down Path.walk"
snippet_type: recipe
use_cases:
  - automation
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - ../storage-databases/build-a-bounded-file-manifest-with-internal-symlink-aliases.md
  - copy-a-test-fixture-tree-with-an-explicit-symlink-policy.md
---

# Prune Named Generated Subtrees with Top-Down Path.walk

## Idea and Problem

List source files in a scratch tree without visiting directories designated as generated output.

A top-down `Path.walk()` yields the list of child directory names before
descending. Edit that list in place to remove generated subtrees at every
level. Filtering file names after the walk would still incur the traversal.

## When to Use

Use this for a small, stable tree you control, such as a test fixture with
source files beside build output. Supply exact directory names to exclude.
The example treats symlinks as entries to skip and returns sorted relative
names of regular files.

## Implementation

```python
from pathlib import Path
from stat import S_ISDIR, S_ISREG


def raise_scan_error(error: OSError) -> None:
    raise error


def list_fixture_files(root: Path, skip_names: frozenset[str]) -> tuple[str, ...]:
    if not S_ISDIR(root.lstat().st_mode):
        raise NotADirectoryError("fixture root must be a directory entry")

    relative_paths: list[str] = []
    for folder, dirs, names in root.walk(
        top_down=True, follow_symlinks=False, on_error=raise_scan_error
    ):
        dirs[:] = [name for name in dirs if name not in skip_names]
        for name in names:
            entry = folder / name
            if S_ISREG(entry.lstat().st_mode):
                relative_paths.append(entry.relative_to(root).as_posix())
    return tuple(sorted(relative_paths))
```

## Example

```python
from tempfile import TemporaryDirectory


names = (
    "source/entry.txt",
    "docs/spec.txt",
    "build/report.txt",
    "docs/build/intermediate.txt",
    "source/generated/output.txt",
)
with TemporaryDirectory() as temporary:
    root = Path(temporary)
    for name in names:
        destination = root / name
        destination.parent.mkdir(parents=True, exist_ok=True)
        destination.write_text("fixture\n", encoding="utf-8")

    (root / "source" / "linked-build").symlink_to("../build", target_is_directory=True)
    (root / "linked-spec.txt").symlink_to("docs/spec.txt")
    kept = list_fixture_files(root, frozenset({"build", "generated"}))

assert kept == ("docs/spec.txt", "source/entry.txt")
```

## Trade-offs and Limitations

The slice assignment is essential because the walker reads the same list
after yielding it. A bottom-up walk has already visited the children.
This helper propagates errors from directory scans and entry inspection.
Sorting retains every selected relative name in memory.

The example needs a filesystem that permits creating symlinks. The walker
does not traverse symlinks it observes in the stable tree, and `lstat()`
keeps those links out of the file results. A concurrent replacement can
invalidate these observations. Use a controlled tree rather than treating
the walk as a filesystem security boundary.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Bounded File Manifest with Internal Symlink Aliases](../storage-databases/build-a-bounded-file-manifest-with-internal-symlink-aliases.md)
- [Copy a Test Fixture Tree with an Explicit Symlink Policy](copy-a-test-fixture-tree-with-an-explicit-symlink-policy.md)
<!-- catalog:related:end -->
