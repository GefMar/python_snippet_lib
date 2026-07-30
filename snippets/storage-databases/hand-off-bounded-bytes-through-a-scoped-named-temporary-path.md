---
title: "Hand Off Bounded Bytes Through a Scoped Named Temporary Path"
snippet_type: recipe
use_cases:
  - interoperability
  - persistence
  - resource-management
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - spool-a-bounded-byte-iterable-into-a-scoped-seekable-temporary-file.md
  - replace-a-file-atomically-with-a-sibling-temporary-file.md
  - publish-a-new-posix-file-without-replacement-and-sync-directory-entries.md
---

# Hand Off Bounded Bytes Through a Scoped Named Temporary Path

## Idea and Problem

Give a path-only local consumer bounded bytes while retaining deterministic cleanup ownership in the surrounding context manager.

`NamedTemporaryFile(delete=True, delete_on_close=False)` separates closing the
primary handle from deleting its name. Writing and closing before `yield`
makes ordinary reopening portable to Windows, while the still-active outer
context removes the file after the consumer finishes or raises.

## When to Use

Use this recipe when a trusted synchronous library accepts only a filesystem
path and the data is needed for one local call. Keep the payload between 1 and
1,048,576 bytes, use only a small lowercase extension when the consumer needs
one, and close every secondary handle before leaving the `with` block.

Pass an open handle when the consumer supports file objects. Use a durable
caller-owned path for later reuse, and use a same-directory atomic publication
pattern when readers must observe a permanent all-or-nothing replacement.

## Implementation

```python
import re
from collections.abc import Iterator
from contextlib import contextmanager
from pathlib import Path
from tempfile import NamedTemporaryFile

_MAX_PAYLOAD_BYTES = 1024 * 1024
_SUFFIX_PATTERN = re.compile(r"(?:\.[a-z0-9]{1,8})?", flags=re.ASCII)


@contextmanager
def scoped_named_payload(
    payload: bytes,
    *,
    suffix: str = "",
) -> Iterator[Path]:
    if type(payload) is not bytes:
        raise TypeError("payload must be exact bytes")
    if not 1 <= len(payload) <= _MAX_PAYLOAD_BYTES:
        raise ValueError("payload is outside the supported byte range")
    if type(suffix) is not str:
        raise TypeError("suffix must be an exact string")
    if _SUFFIX_PATTERN.fullmatch(suffix) is None:
        raise ValueError("suffix must be empty or a short lowercase extension")

    with NamedTemporaryFile(
        mode="w+b",
        suffix=suffix,
        delete=True,
        delete_on_close=False,
    ) as temporary:
        written = temporary.write(payload)
        if written != len(payload):
            raise OSError("temporary-file write was incomplete")
        temporary.flush()
        path = Path(temporary.name)
        temporary.close()
        yield path
```

## Example

```python
class ConsumerStopped(RuntimeError):
    pass


scoped_path: Path | None = None
try:
    with scoped_named_payload(b"header\nbody\n", suffix=".bin") as path:
        scoped_path = path
        with path.open("rb") as reopened:
            observed = reopened.read()
        assert path.exists()
        raise ConsumerStopped("exercise exceptional cleanup")
except ConsumerStopped:
    pass

assert scoped_path is not None
assert observed == b"header\nbody\n"
assert not scoped_path.exists()
```

## Trade-offs and Limitations

Closing the primary handle before `yield` is important on Windows. Every
handle opened by the consumer must also be closed before outer context exit;
otherwise deletion can fail with `PermissionError`. Write, consumer, close,
and cleanup failures propagate. Normal context exit is not a promise of
cleanup after a process crash, forced termination, or host failure.

The visible name introduces a time-of-check/time-of-use window after the
primary handle closes. Another actor able to modify the temporary directory
could unlink or replace that name before the consumer reopens it. Use this
only with a trusted local consumer and an appropriately protected temporary
directory, not as a security-sensitive handoff to another trust domain.

The payload is copied to temporary storage in `O(B)` time and space. `flush()`
makes bytes visible to a reopening handle but does not request durable storage;
there is no `fsync`. The recipe is neither encrypted storage, atomic
publication, a stable path after scope exit, nor a guarantee that the
temporary directory has a particular filesystem or capacity.

## Related Snippets

<!-- catalog:related:start -->
- [Spool a Bounded Byte Iterable into a Scoped Seekable Temporary File](spool-a-bounded-byte-iterable-into-a-scoped-seekable-temporary-file.md)
- [Replace a File Atomically with a Sibling Temporary File](replace-a-file-atomically-with-a-sibling-temporary-file.md)
- [Publish a New POSIX File Without Replacement and Sync Directory Entries](publish-a-new-posix-file-without-replacement-and-sync-directory-entries.md)
<!-- catalog:related:end -->
