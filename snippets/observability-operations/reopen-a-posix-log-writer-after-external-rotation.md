---
title: "Reopen a POSIX Log Writer After External Rotation"
snippet_type: recipe
use_cases:
  - observability
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - process-log-records-in-a-background-thread-with-queuelistener.md
  - read-a-bounded-log-delta-across-one-rename-based-rotation.md
---

# Reopen a POSIX Log Writer After External Rotation

## Idea and Problem

Send subsequent log records to the current path after an external rotator replaces the file.

An open file can keep receiving writes after its name moves. Before emitting a
record, `WatchedFileHandler` checks whether the path names a different file
by comparing device and inode. On replacement, it closes the old stream and
opens the path again in append mode.

## When to Use

Use this for a regular local POSIX log managed by a separate rename-and-create
rotator. The caller supplies a logger already configured to accept the
desired levels. Let this scope own the attached handler, and finish the
emitters before leaving it.

## Implementation

```python
import logging
import os
from collections.abc import Iterator
from contextlib import closing, contextmanager
from logging.handlers import WatchedFileHandler
from pathlib import Path


@contextmanager
def watched_log(logger: logging.Logger, path: Path) -> Iterator[None]:
    if os.name != "posix":
        raise NotImplementedError("this rotation recipe requires POSIX")

    with closing(WatchedFileHandler(path, mode="a", encoding="utf-8")) as handler:
        logger.addHandler(handler)
        try:
            yield
        finally:
            logger.removeHandler(handler)
```

## Example

```python
from tempfile import TemporaryDirectory

logger = logging.Logger("batch-status", level=logging.INFO)

with TemporaryDirectory() as directory:
    current = Path(directory) / "batch.log"
    rotated = current.with_name("batch.log.1")

    with watched_log(logger, current):
        logger.info("batch accepted")
        current.rename(rotated)
        current.touch(exist_ok=False)
        logger.info("batch completed")

    old_text = rotated.read_text(encoding="utf-8")
    new_text = current.read_text(encoding="utf-8")

assert (old_text, new_text, logger.handlers) == (
    "batch accepted\n",
    "batch completed\n",
    [],
)
```

## Trade-offs and Limitations

The check adds a filesystem lookup to each emit. It does not rotate files,
coordinate rotation with writers, or guarantee lossless delivery. Use a
trusted writable directory and coordinate creation and ownership with the
rotator: a write during the rename-and-create gap can create the path itself.

Copy-and-truncate keeps the same file identity, so it is not detected as a
replacement. Do not use this recipe on Windows or on filesystems without
reliable device and inode identities.

## Related Snippets

<!-- catalog:related:start -->
- [Process Log Records in a Background Thread with QueueListener](process-log-records-in-a-background-thread-with-queuelistener.md)
- [Read a Bounded Log Delta Across One Rename-Based Rotation](read-a-bounded-log-delta-across-one-rename-based-rotation.md)
<!-- catalog:related:end -->
