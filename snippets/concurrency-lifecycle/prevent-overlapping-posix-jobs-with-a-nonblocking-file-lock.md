---
title: "Prevent Overlapping POSIX Jobs with a Nonblocking File Lock"
snippet_type: recipe
use_cases:
  - concurrency-control
  - automation
tested_python:
  - "3.14"
dependencies: []
related:
  - guard-an-async-resource-with-explicit-lifecycle-states.md
  - stop-a-polling-worker-cooperatively-with-an-event.md
  - ../storage-databases/replace-a-file-atomically-with-a-sibling-temporary-file.md
---

# Prevent Overlapping POSIX Jobs with a Nonblocking File Lock

## Idea and Problem

Hold a nonblocking advisory lock for one job run so a concurrent local invocation can fail immediately instead of overlapping it.

The lock belongs to an open file descriptor, not to the presence or contents
of the file. Opening without truncation preserves an existing lock file, and
keeping the descriptor open for the complete critical section lets process
exit release the kernel lock even after an unhandled failure.

## When to Use

Use this context manager for cooperative scheduled jobs on one POSIX host when
every participant agrees to lock the same file. Select a trusted,
pre-existing directory with appropriate ownership and permissions before the
job starts. Treat `JobAlreadyRunning` as a normal decision to skip or report
the overlapping invocation.

Use a scheduler lease, database lock, or other distributed coordination
mechanism across hosts. A file lock is not a PID-file protocol, a privilege
boundary, or a substitute for authorization of the job's protected resources.

## Implementation

```python
import errno
import fcntl
import os
import stat
from collections.abc import Iterator
from contextlib import contextmanager
from pathlib import Path


class JobAlreadyRunning(RuntimeError):
    pass


@contextmanager
def exclusive_posix_job_lock(lock_path: str | os.PathLike[str]) -> Iterator[None]:
    path = Path(lock_path)
    if path.name in {"", ".", ".."}:
        raise ValueError("lock_path must name a file")
    if not path.parent.is_dir():
        raise FileNotFoundError("the trusted lock directory must already exist")
    if path.is_symlink():
        raise ValueError("lock_path must not be a symbolic link")

    flags = os.O_RDWR | os.O_CREAT | getattr(os, "O_CLOEXEC", 0)
    if hasattr(os, "O_NOFOLLOW"):
        flags |= os.O_NOFOLLOW
    descriptor = os.open(path, flags, 0o600)
    acquired = False
    try:
        if not stat.S_ISREG(os.fstat(descriptor).st_mode):
            raise ValueError("lock_path must refer to a regular file")
        try:
            fcntl.flock(descriptor, fcntl.LOCK_EX | fcntl.LOCK_NB)
        except OSError as error:
            if error.errno in {errno.EACCES, errno.EAGAIN}:
                raise JobAlreadyRunning("another invocation holds the lock") from error
            raise
        acquired = True
        yield
    finally:
        try:
            if acquired:
                fcntl.flock(descriptor, fcntl.LOCK_UN)
        finally:
            os.close(descriptor)
```

## Example

```python
from tempfile import TemporaryDirectory


with TemporaryDirectory() as temporary_directory:
    lock_file = Path(temporary_directory) / "scheduled-job.lock"
    lock_file.write_text("do not truncate\n", encoding="utf-8")

    with exclusive_posix_job_lock(lock_file):
        try:
            with exclusive_posix_job_lock(lock_file):
                pass
        except JobAlreadyRunning:
            overlap_rejected = True
        else:
            overlap_rejected = False

    with exclusive_posix_job_lock(lock_file):
        acquired_after_release = True

    observed = (
        overlap_rejected,
        acquired_after_release,
        lock_file.read_text(encoding="utf-8"),
        lock_file.exists(),
    )

assert observed == (True, True, "do not truncate\n", True)
```

## Trade-offs and Limitations

`fcntl.flock` is available on Unix platforms, not Windows or WASI, and lock
semantics on network and userspace filesystems vary. The mechanism is advisory:
a process that ignores the convention can still run or modify protected data.
All contenders must resolve the same persistent file identity, so do not unlink
or replace the lock file while jobs may be active.

The pre-checks are not a complete defense against directory-entry races. On
platforms with `O_NOFOLLOW`, the open also rejects a final-component symlink;
without it, the trusted-directory precondition is essential. File creation
uses mode `0o600` subject to the process umask, but an existing file keeps its
current ownership and permissions. Only `EACCES` and `EAGAIN` are translated
as contention; other open, lock, unlock, and close failures propagate.

## Related Snippets

<!-- catalog:related:start -->
- [Guard an Async Resource with Explicit Lifecycle States](guard-an-async-resource-with-explicit-lifecycle-states.md)
- [Stop a Polling Worker Cooperatively with an Event](stop-a-polling-worker-cooperatively-with-an-event.md)
- [Replace a File Atomically with a Sibling Temporary File](../storage-databases/replace-a-file-atomically-with-a-sibling-temporary-file.md)
<!-- catalog:related:end -->
