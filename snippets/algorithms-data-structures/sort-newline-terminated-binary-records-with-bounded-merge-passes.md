---
title: "Sort Newline-Terminated Binary Records with Bounded Merge Passes"
snippet_type: algorithm
use_cases:
  - data-transformation
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - ../storage-databases/replace-a-file-atomically-with-a-sibling-temporary-file.md
  - ../data-processing/batch-items-by-estimated-byte-size.md
  - ../python-language/read-fixed-size-blocks-with-iter-sentinel.md
---

# Sort Newline-Terminated Binary Records with Bounded Merge Passes

## Idea and Problem

Sort a file larger than memory by creating byte-bounded sorted runs and merging them in passes with a fixed open-file fan-in.

Binary newline-terminated records make memory accounting and preservation
explicit. Each merge opens at most `fan_in` input runs, and the final run
replaces the destination only after every read, comparison, write, and close
has succeeded.

## When to Use

Use this algorithm for trusted local files containing independent records that
all end in `b"\n"`, when a full in-memory sort is too large but local scratch
space is available. Choose a byte budget larger than the longest permitted
record and a fan-in below the process file-descriptor limit. Use a database or
specialized external-sort tool for parallel execution, compression, schema
validation, or recovery across process restarts.

## Implementation

```python
import heapq
import os
import stat
import sys
import tempfile
from collections.abc import Callable, Iterator, Sequence
from contextlib import ExitStack
from itertools import batched
from pathlib import Path
from typing import BinaryIO


RecordKey = Callable[[bytes], object]


def _records(stream: BinaryIO, *, max_record_bytes: int) -> Iterator[bytes]:
    while record := stream.readline(max_record_bytes + 1):
        if len(record) > max_record_bytes:
            raise ValueError("one record exceeds max_run_bytes")
        if not record.endswith(b"\n"):
            raise ValueError("every input record must end with a newline")
        yield record


def _write_sorted_run(
    records: list[bytes],
    path: Path,
    *,
    key: RecordKey,
) -> None:
    records.sort(key=lambda record: key(record[:-1]))
    with path.open("wb") as target:
        target.writelines(records)


def _merge_runs(
    sources: Sequence[Path],
    target: Path,
    *,
    key: RecordKey,
    max_record_bytes: int,
) -> None:
    with ExitStack() as stack:
        streams = [stack.enter_context(path.open("rb")) for path in sources]
        inputs = [
            _records(stream, max_record_bytes=max_record_bytes)
            for stream in streams
        ]
        with target.open("wb") as output:
            output.writelines(
                heapq.merge(
                    *inputs,
                    key=lambda record: key(record[:-1]),
                )
            )


def _published_permissions(target: Path, new_permissions: int) -> int:
    if isinstance(new_permissions, bool) or not isinstance(new_permissions, int):
        raise TypeError("new_permissions must be an integer")
    if not 0 <= new_permissions <= 0o777:
        raise ValueError("new_permissions must be between 0o000 and 0o777")
    try:
        existing = target.lstat()
    except FileNotFoundError:
        return new_permissions
    if not stat.S_ISREG(existing.st_mode):
        raise ValueError("output must be absent or a regular file")
    return stat.S_IMODE(existing.st_mode) & 0o777


def external_sort_binary_records(
    input_path: str | os.PathLike[str],
    output_path: str | os.PathLike[str],
    *,
    max_run_bytes: int,
    fan_in: int = 32,
    key: RecordKey | None = None,
    new_permissions: int = 0o600,
) -> None:
    for name, value, minimum in (
        ("max_run_bytes", max_run_bytes, 1),
        ("fan_in", fan_in, 2),
    ):
        if isinstance(value, bool) or not isinstance(value, int):
            raise TypeError(f"{name} must be an integer")
        if value < minimum:
            raise ValueError(f"{name} must be at least {minimum}")
    if max_run_bytes >= sys.maxsize:
        raise ValueError("max_run_bytes must be smaller than sys.maxsize")
    if key is not None and not callable(key):
        raise TypeError("key must be callable or None")

    source_path = Path(input_path)
    destination = Path(output_path)
    if not destination.parent.is_dir():
        raise FileNotFoundError("output parent directory does not exist")
    permissions = _published_permissions(destination, new_permissions)
    record_key: RecordKey = key if key is not None else lambda record: record

    with tempfile.TemporaryDirectory(
        prefix=f".{destination.name}.sort-",
        dir=destination.parent,
    ) as temporary_name:
        temporary = Path(temporary_name)
        runs: list[Path] = []
        buffered: list[bytes] = []
        buffered_bytes = 0
        next_run = 0

        def flush_run() -> None:
            nonlocal buffered_bytes, next_run
            if not buffered:
                return
            run = temporary / f"run-{next_run:08d}.bin"
            next_run += 1
            _write_sorted_run(buffered, run, key=record_key)
            runs.append(run)
            buffered.clear()
            buffered_bytes = 0

        with source_path.open("rb") as source:
            for record in _records(source, max_record_bytes=max_run_bytes):
                if buffered and buffered_bytes + len(record) > max_run_bytes:
                    flush_run()
                buffered.append(record)
                buffered_bytes += len(record)
        flush_run()

        if not runs:
            empty_run = temporary / f"run-{next_run:08d}.bin"
            empty_run.write_bytes(b"")
            runs.append(empty_run)
            next_run += 1

        while len(runs) > 1:
            next_pass: list[Path] = []
            for group in batched(runs, fan_in):
                if len(group) == 1:
                    next_pass.append(group[0])
                    continue
                merged = temporary / f"run-{next_run:08d}.bin"
                next_run += 1
                _merge_runs(
                    group,
                    merged,
                    key=record_key,
                    max_record_bytes=max_run_bytes,
                )
                next_pass.append(merged)
                for run in group:
                    run.unlink()
            runs = next_pass

        final_run = runs[0]
        os.chmod(final_run, permissions)
        os.replace(final_run, destination)
```

## Example

```python
from pathlib import Path
from tempfile import TemporaryDirectory


with TemporaryDirectory() as directory:
    root = Path(directory)
    data = root / "records.bin"
    data.write_bytes(b"b:1\na:1\nb:2\na:2\nc:1\n")

    external_sort_binary_records(
        data,
        data,
        max_run_bytes=8,
        fan_in=2,
        key=lambda record: record.split(b":", 1)[0],
    )

    invalid = root / "invalid.bin"
    invalid.write_bytes(b"valid\nmissing-newline")
    protected = root / "protected.bin"
    protected.write_bytes(b"keep\n")
    try:
        external_sort_binary_records(
            invalid,
            protected,
            max_run_bytes=32,
            fan_in=2,
        )
    except ValueError:
        invalid_rejected = True
    else:
        invalid_rejected = False

    leftovers = list(root.glob(".*.sort-*"))
    observed = (
        data.read_bytes(),
        protected.read_bytes(),
        invalid_rejected,
        leftovers,
    )

assert observed == (
    b"a:1\na:2\nb:1\nb:2\nc:1\n",
    b"keep\n",
    True,
    [],
)
```

## Trade-offs and Limitations

The byte budget excludes Python container overhead and the merge phase still
holds one record per open run, but no accepted record exceeds the configured
budget. The run-path list grows with the number of runs, and a merge pass can
temporarily require nearly twice the input size in scratch space because old
runs coexist with their merged replacements; the separately retained source
and existing destination require additional space. Each extra pass adds
full-file I/O. The key sees
record bytes without the trailing `b"\n"`; its results must be deterministic
and mutually comparable. Equal-key records retain input order, but input with
missing final newlines is rejected and carriage returns remain part of the
payload. Final replacement is reader-atomic where supported, not crash-durable
without platform-specific directory synchronization. Special set-id/sticky
mode bits, ownership, extended attributes, sparse layout, and timestamps are
not preserved.

## Related Snippets

<!-- catalog:related:start -->
- [Replace a File Atomically with a Sibling Temporary File](../storage-databases/replace-a-file-atomically-with-a-sibling-temporary-file.md)
- [Batch Items by Estimated Byte Size](../data-processing/batch-items-by-estimated-byte-size.md)
- [Read Fixed-Size Blocks with iter() and a Sentinel](../python-language/read-fixed-size-blocks-with-iter-sentinel.md)
<!-- catalog:related:end -->
