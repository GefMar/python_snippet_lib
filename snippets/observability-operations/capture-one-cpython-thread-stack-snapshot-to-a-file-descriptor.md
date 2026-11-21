---
title: "Capture One CPython Thread-Stack Snapshot to a File Descriptor"
snippet_type: recipe
use_cases:
  - observability
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - aggregate-bounded-sampled-stacks-into-deterministic-collapsed-counts.md
  - project-a-bounded-asyncio-call-graph-without-retaining-live-frames.md
---

# Capture One CPython Thread-Stack Snapshot to a File Descriptor

## Idea and Problem

Capture where Python threads are executing at one diagnostic checkpoint using a temporary file and the standard-library fault handler.

`faulthandler.dump_traceback` writes synchronously to an open file descriptor.
An owned temporary file gives the capture a destination that stays open until
the dump finishes. The caller receives bytes and can decide whether to keep
them, display them locally, or discard them.

## When to Use

Use this at a chosen checkpoint in a GIL-enabled CPython process that can still
execute diagnostic code. It fits a controlled test or an on-demand local
investigation of what worker threads are doing. Choose `all_threads=False`
when only the calling thread matters.

## Implementation

```python
import faulthandler
from tempfile import TemporaryFile


def snapshot_python_stacks(*, all_threads: bool = True) -> bytes:
    with TemporaryFile(mode="w+b") as output:
        faulthandler.dump_traceback(output.fileno(), all_threads=all_threads)
        output.seek(0)
        return output.read()
```

## Example

```python
from threading import Event, Thread


def wait_for_release(ready: Event, release: Event) -> None:
    ready.set()
    release.wait()


ready = Event()
release = Event()
worker = Thread(target=wait_for_release, args=(ready, release))
worker.start()
try:
    if not ready.wait(timeout=2):
        raise RuntimeError("worker did not reach the diagnostic checkpoint")
    caller_only = snapshot_python_stacks(all_threads=False)
    all_stacks = snapshot_python_stacks()
finally:
    release.set()
    worker.join()

assert (
    b"in snapshot_python_stacks" in caller_only
    and b"in wait_for_release" not in caller_only
    and b"in wait_for_release" in all_stacks
    and not worker.is_alive()
)
```

## Trade-offs and Limitations

The capture is one observation of running threads. It does not establish a
deadlock or describe which locks each thread owns. CPython caps the diagnostic
to 100 threads and 100 frames per thread. Use the output for human inspection,
since frame locations and thread identifiers vary between runs. The tested
all-thread behavior is for GIL-enabled CPython.

The dump can contain source paths and function names. Keep it within the
diagnostic trust boundary. The destination needs a real file descriptor, so
an in-memory `StringIO` or `BytesIO` cannot receive this output. The call
requires a responsive path that can request the capture. No signal handler
or watchdog is installed by the recipe.

## Related Snippets

<!-- catalog:related:start -->
- [Aggregate Bounded Sampled Stacks into Deterministic Collapsed Counts](aggregate-bounded-sampled-stacks-into-deterministic-collapsed-counts.md)
- [Project a Bounded asyncio Call Graph Without Retaining Live Frames](project-a-bounded-asyncio-call-graph-without-retaining-live-frames.md)
<!-- catalog:related:end -->
