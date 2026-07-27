---
title: "Read a Bounded Log Delta Across One Rename-Based Rotation"
snippet_type: recipe
use_cases:
  - observability
  - resource-management
  - retry-recovery
tested_python:
  - "3.14"
dependencies: []
related:
  - ../storage-databases/read-the-last-bounded-binary-lines-with-a-read-only-mmap.md
  - report-partition-offsets-behind-a-fixed-checkpoint.md
---

# Read a Bounded Log Delta Across One Rename-Based Rotation

## Idea and Problem

Resume a binary log from a device, inode, and offset checkpoint while recovering unread bytes across at most one rename-based rotation.

If the current file still has the checkpoint identity, the reader returns its
new suffix. If the identity changed, an explicitly supplied rotated file must
match the checkpoint; its remaining suffix is returned before bytes from the
new current file. Every opened path is verified with `fstat`, and one byte
budget covers the complete result.

## When to Use

Use this recipe for a regular local log whose rotation policy renames the
current file to one known path before creating a new current file. Persist the
returned checkpoint only after the returned bytes have been delivered
durably. Keep calls frequent enough that no second rotation can occur between
successful checkpoints, and choose a byte limit above the expected delta. The
filesystem must expose stable, meaningful `st_dev` and `st_ino` values plus
compatible rename behavior; verify those assumptions before using a network,
overlay, or unusual platform filesystem.

Use a logging agent with its own journal when you need compression,
copy-truncate support, several retained generations, acknowledgements, or
recovery after an arbitrary outage.

## Implementation

```python
import os
import stat
from dataclasses import dataclass
from pathlib import Path
from typing import BinaryIO


_MAX_DELTA_BYTES = 64 * 1024 * 1024
_READ_CHUNK_BYTES = 64 * 1024


class LogGapError(RuntimeError):
    pass


class LogDeltaLimitError(RuntimeError):
    pass


@dataclass(frozen=True, slots=True)
class LogCheckpoint:
    device: int
    inode: int
    offset: int


@dataclass(frozen=True, slots=True)
class LogDelta:
    data: bytes
    next_checkpoint: LogCheckpoint
    crossed_rotation: bool


def _validate_checkpoint(checkpoint: LogCheckpoint) -> None:
    if not isinstance(checkpoint, LogCheckpoint):
        raise TypeError("checkpoint must be LogCheckpoint")
    for name, value in (
        ("device", checkpoint.device),
        ("inode", checkpoint.inode),
        ("offset", checkpoint.offset),
    ):
        if isinstance(value, bool) or not isinstance(value, int):
            raise TypeError(f"checkpoint {name} must be an integer")
        if value < 0:
            raise ValueError(f"checkpoint {name} must be non-negative")


def _open_regular(path: Path) -> tuple[BinaryIO, os.stat_result]:
    discovered = os.stat(path, follow_symlinks=False)
    if not stat.S_ISREG(discovered.st_mode):
        raise LogGapError("a log path is not a regular file")
    stream = Path(path).open("rb")
    try:
        metadata = os.fstat(stream.fileno())
        if (
            not stat.S_ISREG(metadata.st_mode)
            or _identity(metadata) != _identity(discovered)
        ):
            raise LogGapError("a log path changed while it was being opened")
        return stream, metadata
    except BaseException:
        stream.close()
        raise


def _identity(metadata: os.stat_result) -> tuple[int, int]:
    return metadata.st_dev, metadata.st_ino


def _read_span(
    stream: BinaryIO,
    metadata: os.stat_result,
    *,
    start: int,
    end: int,
    remaining_budget: int,
) -> bytes:
    if start > end:
        raise LogGapError("the checkpoint is beyond the file end")
    length = end - start
    if length > remaining_budget:
        raise LogDeltaLimitError("max_bytes was exceeded")

    stream.seek(start)
    chunks: list[bytes] = []
    remaining = length
    while remaining:
        chunk = stream.read(min(remaining, _READ_CHUNK_BYTES))
        if not chunk:
            raise LogGapError("the log changed while it was being read")
        chunks.append(chunk)
        remaining -= len(chunk)

    after = os.fstat(stream.fileno())
    if _identity(after) != _identity(metadata) or after.st_size < end:
        raise LogGapError("the log changed while it was being read")
    return b"".join(chunks)


def read_log_delta(
    current_path: Path,
    *,
    checkpoint: LogCheckpoint | None = None,
    rotated_path: Path | None = None,
    max_bytes: int = 8 * 1024 * 1024,
) -> LogDelta:
    if isinstance(max_bytes, bool) or not isinstance(max_bytes, int):
        raise TypeError("max_bytes must be an integer")
    if not 0 <= max_bytes <= _MAX_DELTA_BYTES:
        raise ValueError("max_bytes is outside the supported range")
    if checkpoint is not None:
        _validate_checkpoint(checkpoint)

    current, current_metadata = _open_regular(Path(current_path))
    try:
        current_identity = _identity(current_metadata)
        current_end = current_metadata.st_size

        if checkpoint is None:
            data = _read_span(
                current,
                current_metadata,
                start=0,
                end=current_end,
                remaining_budget=max_bytes,
            )
            crossed_rotation = False
        elif current_identity == (checkpoint.device, checkpoint.inode):
            data = _read_span(
                current,
                current_metadata,
                start=checkpoint.offset,
                end=current_end,
                remaining_budget=max_bytes,
            )
            crossed_rotation = False
        else:
            if rotated_path is None:
                raise LogGapError("the checkpoint file is no longer available")
            try:
                rotated, rotated_metadata = _open_regular(Path(rotated_path))
            except OSError as error:
                raise LogGapError(
                    "the expected rotated file is unavailable"
                ) from error
            try:
                if _identity(rotated_metadata) != (
                    checkpoint.device,
                    checkpoint.inode,
                ):
                    raise LogGapError(
                        "the rotated file does not match the checkpoint"
                    )
                previous = _read_span(
                    rotated,
                    rotated_metadata,
                    start=checkpoint.offset,
                    end=rotated_metadata.st_size,
                    remaining_budget=max_bytes,
                )
            finally:
                rotated.close()

            current_data = _read_span(
                current,
                current_metadata,
                start=0,
                end=current_end,
                remaining_budget=max_bytes - len(previous),
            )
            data = previous + current_data
            crossed_rotation = True

        return LogDelta(
            data=data,
            next_checkpoint=LogCheckpoint(
                device=current_identity[0],
                inode=current_identity[1],
                offset=current_end,
            ),
            crossed_rotation=crossed_rotation,
        )
    finally:
        current.close()
```

## Example

```python
from tempfile import TemporaryDirectory


with TemporaryDirectory() as temporary_directory:
    directory = Path(temporary_directory)
    current = directory / "service.log"
    rotated = directory / "service.log.1"
    current.write_bytes(b"alpha\n")

    initial = read_log_delta(current, max_bytes=32)
    with current.open("ab") as stream:
        stream.write(b"beta\n")
    current.rename(rotated)
    current.write_bytes(b"gamma\n")

    resumed = read_log_delta(
        current,
        checkpoint=initial.next_checkpoint,
        rotated_path=rotated,
        max_bytes=32,
    )

    current.write_bytes(b"x")
    try:
        read_log_delta(current, checkpoint=resumed.next_checkpoint)
    except LogGapError:
        truncation_rejected = True
    else:
        truncation_rejected = False

assert (
    initial.data,
    resumed.data,
    resumed.crossed_rotation,
    resumed.next_checkpoint.offset,
    truncation_rejected,
) == (b"alpha\n", b"beta\ngamma\n", True, 6, True)
```

## Trade-offs and Limitations

The recipe supports one rename generation only. It deliberately reports a gap
instead of guessing after a missing or mismatched rotated file, same-inode
truncation, copy-truncate rotation, or a second rotation. It reads a byte
snapshot and does not align records; callers must retain an incomplete final
line if their format requires complete records.

Opening a path and then using `fstat` prevents a path replacement from changing
the already opened descriptor, but the function is not a filesystem snapshot.
Concurrent content rewrites can still produce mixed data, and appends after the
initial size are left for the next call. One regular-file read may block. A
returned checkpoint represents bytes read, not bytes durably processed, so
persisting it before downstream acknowledgement can lose data; persisting it
after processing can intentionally produce at-least-once replay after a crash.

## Related Snippets

<!-- catalog:related:start -->
- [Read the Last Bounded Binary Lines with a Read-Only mmap](../storage-databases/read-the-last-bounded-binary-lines-with-a-read-only-mmap.md)
- [Report Partition Offsets Behind a Fixed Checkpoint](report-partition-offsets-behind-a-fixed-checkpoint.md)
<!-- catalog:related:end -->
