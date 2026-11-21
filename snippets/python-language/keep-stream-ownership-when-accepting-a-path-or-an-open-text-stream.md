---
title: "Keep Stream Ownership When Accepting a Path or an Open Text Stream"
snippet_type: recipe
use_cases:
  - interoperability
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/read-csv-from-a-borrowed-binary-stream-without-closing-it.md
  - ../storage-databases/hand-off-bounded-bytes-through-a-scoped-named-temporary-path.md
---

# Keep Stream Ownership When Accepting a Path or an Open Text Stream

## Idea and Problem

Use one reading scope for a path or an existing text stream while leaving closure with the appropriate owner.

When the adapter receives a `Path`, it opens and owns the resulting text
file. When it receives an already open text stream, it yields the same
object through `nullcontext()`. Both branches support one `with` statement,
and the borrowed stream stays open even if the reading operation raises.

## When to Use

Use this when a reader accepts either a UTF-8 file path or a text stream
managed by its caller. Enter the scope for each read and give the reader
exclusive access to the supplied stream while it reads.

## Implementation

```python
from collections.abc import Iterator
from contextlib import contextmanager, nullcontext
from pathlib import Path
from typing import TextIO


@contextmanager
def text_reader(source: Path | TextIO) -> Iterator[TextIO]:
    manager = source.open(encoding="utf-8") if isinstance(source, Path) else nullcontext(source)
    with manager as reader:
        yield reader
```

## Example

```python
from io import StringIO
from tempfile import TemporaryDirectory


class StopReading(RuntimeError):
    pass


def read_probe(source: Path | TextIO, *, fail: bool) -> tuple[str, bool, bool]:
    try:
        with text_reader(source) as reader:
            if not isinstance(source, Path):
                assert reader is source
            line = reader.readline()
            if fail:
                raise StopReading("fixture operation failed")
    except StopReading:
        saw_error = True
    else:
        saw_error = False
    return line, reader.closed, saw_error


with TemporaryDirectory() as temporary:
    path = Path(temporary) / "report.txt"
    path.write_text("heading\nrow one\n", encoding="utf-8")
    owned_normal = read_probe(path, fail=False)
    owned_failure = read_probe(path, fail=True)

with StringIO("heading\nrow one\nrow two\nrow three\n") as borrowed:
    assert borrowed.readline() == "heading\n"
    borrowed_normal = read_probe(borrowed, fail=False)
    borrowed_failure = read_probe(borrowed, fail=True)
    remainder = borrowed.readline()

assert (owned_normal, owned_failure, borrowed_normal, borrowed_failure, remainder) == (
    ("heading\n", True, False),
    ("heading\n", True, True),
    ("row one\n", False, False),
    ("row two\n", False, True),
    "row three\n",
)
```

## Trade-offs and Limitations

Files opened here use UTF-8. A borrowed stream keeps its caller-selected
encoding and newline behavior, and reading starts at its current position.
The adapter does not rewind it or roll back consumption after a failed read.

The scope leaves errors visible to the caller. Finish any lazy reading
before leaving the block because a file opened from a path is then closed.
The caller remains responsible for eventually closing borrowed streams.

## Related Snippets

<!-- catalog:related:start -->
- [Read CSV from a Borrowed Binary Stream Without Closing It](../data-processing/read-csv-from-a-borrowed-binary-stream-without-closing-it.md)
- [Hand Off Bounded Bytes Through a Scoped Named Temporary Path](../storage-databases/hand-off-bounded-bytes-through-a-scoped-named-temporary-path.md)
<!-- catalog:related:end -->
