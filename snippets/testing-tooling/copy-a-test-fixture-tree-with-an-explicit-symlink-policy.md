---
title: "Copy a Test Fixture Tree with an Explicit Symlink Policy"
snippet_type: testing-technique
use_cases:
  - resource-management
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-collision-audited-artifact-copy-plan.md
  - ../storage-databases/build-a-bounded-file-manifest-with-internal-symlink-aliases.md
---

# Copy a Test Fixture Tree with an Explicit Symlink Policy

## Idea and Problem

Give a test its own writable fixture directory and choose whether source symlinks stay links in the copy.

`Path.copy()` can reproduce a symlink or copy its target. The choice matters
when a fixture contains a relative alias to one of its files. Keeping the
alias in a self-contained tree preserves that relationship. Copying the
target gives the alias path its own file.

## When to Use

Use this in Python 3.14 or later for small, trusted fixture trees that contain
relative links to entries within the tree. Use a system that supports
symlink creation. The temporary destination must be outside the source tree.
Choose to keep the links if the test needs to preserve the alias behavior.

## Implementation

```python
from collections.abc import Iterator
from contextlib import contextmanager
from pathlib import Path
from tempfile import TemporaryDirectory


@contextmanager
def copied_fixture(source: Path, *, follow_symlinks: bool) -> Iterator[Path]:
    if not source.is_dir(follow_symlinks=False):
        raise ValueError("source must be a fixture directory")
    with TemporaryDirectory() as temporary:
        destination = Path(temporary) / "fixture"
        yield source.copy(destination, follow_symlinks=follow_symlinks)
```

## Example

```python
with TemporaryDirectory() as temporary:
    base = Path(temporary) / "base"
    base.mkdir()
    (base / "settings.txt").write_text("mode=base", encoding="utf-8")
    (base / "current.txt").symlink_to("settings.txt")

    with (
        copied_fixture(base, follow_symlinks=False) as linked,
        copied_fixture(base, follow_symlinks=True) as resolved,
    ):
        linked_alias = linked / "current.txt"
        resolved_alias = resolved / "current.txt"
        assert linked_alias.readlink() == Path("settings.txt")
        assert resolved_alias.is_file() and not resolved_alias.is_symlink()

        linked_alias.write_text("mode=linked", encoding="utf-8")
        resolved_alias.write_text("mode=resolved", encoding="utf-8")
        observed = (
            (linked / "settings.txt").read_text(encoding="utf-8"),
            resolved_alias.read_text(encoding="utf-8"),
            (resolved / "settings.txt").read_text(encoding="utf-8"),
            (base / "current.txt").read_text(encoding="utf-8"),
        )

    assert not linked.exists() and not resolved.exists()

assert observed == ("mode=linked", "mode=resolved", "mode=base", "mode=base")
```

## Trade-offs and Limitations

This relies on a stable, self-contained fixture. It does not check every link
target. A preserved absolute link or a relative link that escapes the tree
can point outside the scratch directory. Following such a link during copying
can also read files outside the source. Use an acyclic tree when following
links, and avoid dangling links in that mode.

The destination exists only within the context. File data is copied, with
copy-on-write used when the operating system and filesystem support it. Size
the fixture for a full copy and request metadata preservation explicitly if
the test needs filesystem attributes.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Collision-Audited Artifact Copy Plan](build-a-collision-audited-artifact-copy-plan.md)
- [Build a Bounded File Manifest with Internal Symlink Aliases](../storage-databases/build-a-bounded-file-manifest-with-internal-symlink-aliases.md)
<!-- catalog:related:end -->
