---
title: "Scope a Current Directory Change to One Synchronous Operation"
snippet_type: recipe
use_cases:
  - lifecycle-management
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/materialize-a-package-resource-directory-for-a-path-only-reader.md
  - ../testing-tooling/clean-up-unittest-fixtures-when-setup-fails-with-entercontext.md
---

# Scope a Current Directory Change to One Synchronous Operation

## Idea and Problem

Run a legacy operation from a temporary working directory and restore the previous directory on exit.

`contextlib.chdir()` saves the current working directory and changes it for
the body of the `with` block. The scope restores the saved directory on
ordinary exit and when the body raises, provided the original directory is
still reachable at its saved path.

## When to Use

Prefer passing an explicit base path when the API accepts one. Use this
scope only for a short synchronous call that insists on resolving relative
paths from the working directory, in a process you control exclusively.

## Implementation

```python
from collections.abc import Callable
from contextlib import chdir
from pathlib import Path


def run_from_directory[Result](directory: Path, operation: Callable[[], Result]) -> Result:
    with chdir(directory):
        return operation()
```

## Example

```python
from tempfile import TemporaryDirectory


def read_legacy_report() -> tuple[Path, str]:
    return Path.cwd(), Path("report.txt").read_text(encoding="utf-8")


original = Path.cwd()
with TemporaryDirectory() as temporary:
    empty = Path(temporary).resolve()
    workspace = empty / "report"
    workspace.mkdir()
    (workspace / "report.txt").write_text("ready\n", encoding="utf-8")

    used_directory, text = run_from_directory(workspace, read_legacy_report)
    restored_after_success = Path.cwd() == original

    try:
        run_from_directory(empty, read_legacy_report)
    except FileNotFoundError as error:
        missing_report = error.filename == "report.txt"
    else:
        missing_report = False
    restored_after_error = Path.cwd() == original

assert (
    used_directory == workspace
    and text == "ready\n"
    and missing_report
    and restored_after_success
    and restored_after_error
)
```

## Trade-offs and Limitations

The working directory is process-wide, not local to the call. Do not use
this around concurrent threads, async tasks, or a generator that yields
control while the scope is open.

The target must exist and be accessible at entry. The context saves a path,
not an open directory handle. Removing, renaming, or replacing the original
directory can prevent restoration of the original location. The scope does
not undo filesystem changes made by the operation.

## Related Snippets

<!-- catalog:related:start -->
- [Materialize a Package Resource Directory for a Path-Only Reader](../python-language/materialize-a-package-resource-directory-for-a-path-only-reader.md)
- [Clean Up unittest Fixtures When setUp Fails with enterContext](../testing-tooling/clean-up-unittest-fixtures-when-setup-fails-with-entercontext.md)
<!-- catalog:related:end -->
