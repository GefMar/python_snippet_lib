---
title: "Materialize a Package Resource Directory for a Path-Only Reader"
snippet_type: recipe
use_cases:
  - interoperability
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - load-text-templates-from-package-resources.md
  - ../storage-databases/hand-off-bounded-bytes-through-a-scoped-named-temporary-path.md
---

# Materialize a Package Resource Directory for a Path-Only Reader

## Idea and Problem

Give a path-only reader a real directory for packaged assets while preserving the relative layout of its files.

`importlib.resources.files()` can return a resource container backed by a
ZIP archive. A reader that opens a layout and follows relative references
needs the whole directory. Use `as_file()` on that directory to make its
files available together for the duration of the read.

## When to Use

Use this when a trusted package supplies a small asset bundle and a
synchronous reader accepts a real directory. A layout that references a
neighboring stylesheet or a nested template is a good fit. If the consumer
only needs text or bytes, read directly from the resource container.

## Implementation

```python
from collections.abc import Callable
from importlib.resources import as_file, files
from pathlib import Path
from types import ModuleType


def read_packaged_layout[T](anchor: ModuleType, reader: Callable[[Path], T]) -> T:
    layout = files(anchor).joinpath("layout")
    with as_file(layout) as directory:
        return reader(directory)
```

## Example

```python
from importlib.util import module_from_spec
from tempfile import TemporaryDirectory
from zipfile import ZipFile
from zipimport import zipimporter


class ReaderStopped(Exception):
    pass


observed_paths: list[Path] = []


def render_layout(directory: Path) -> str:
    observed_paths.append(directory)
    shell = (directory / "shell.txt").read_text(encoding="utf-8")
    heading = (directory / "heading.txt").read_text(encoding="utf-8")
    body = (directory / "parts" / "body.txt").read_text(encoding="utf-8")
    return shell.format(heading=heading, body=body)


def stop_after_rendering(directory: Path) -> str:
    render_layout(directory)
    raise ReaderStopped


with TemporaryDirectory() as temporary:
    archive_path = Path(temporary) / "assets.zip"
    with ZipFile(archive_path, "w") as archive:
        archive.writestr("layout_assets/__init__.py", "")
        archive.writestr("layout_assets/layout/shell.txt", "{heading}\n{body}")
        archive.writestr("layout_assets/layout/heading.txt", "Release notes")
        archive.writestr("layout_assets/layout/parts/body.txt", "Two updates")

    spec = zipimporter(str(archive_path)).find_spec("layout_assets")
    assert spec is not None
    anchor = module_from_spec(spec)

    result = read_packaged_layout(anchor, render_layout)
    assert len(observed_paths) == 1 and not observed_paths[0].exists()
    try:
        read_packaged_layout(anchor, stop_after_rendering)
    except ReaderStopped:
        pass
    else:
        raise AssertionError("reader exception must propagate")

    assert archive_path.is_file()
    assert len(observed_paths) == 2 and not observed_paths[1].exists()

assert result == "Release notes\nTwo updates"
```

## Trade-offs and Limitations

Directory support in `as_file()` requires Python 3.12 or later. An unpacked
package may already expose a real directory, which the context does not
remove. An archive-backed loader can require temporary storage for the whole
directory. Limit this pattern to small, trusted bundles.

The reader must finish within the context and close any files it opens.
Return eager results such as rendered text. Paths, lazy readers and background
work can outlive the materialized directory and fail on later access. The
example keeps the observed paths only long enough to assert cleanup.

## Related Snippets

<!-- catalog:related:start -->
- [Load Text Templates from Package Resources](load-text-templates-from-package-resources.md)
- [Hand Off Bounded Bytes Through a Scoped Named Temporary Path](../storage-databases/hand-off-bounded-bytes-through-a-scoped-named-temporary-path.md)
<!-- catalog:related:end -->
