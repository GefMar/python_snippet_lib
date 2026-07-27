---
title: "Stream Bounded stdout and stderr Lines from a POSIX Process"
snippet_type: integration
use_cases:
  - automation
  - concurrency-control
  - lifecycle-management
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - drain-bounded-deferred-writes-outside-the-queue-lock.md
  - stop-a-polling-worker-cooperatively-with-an-event.md
  - ../data-processing/limit-text-lines-across-arbitrary-chunks.md
---

# Stream Bounded stdout and stderr Lines from a POSIX Process

## Idea and Problem

Drain stdout and stderr from one shell-free POSIX child without pipe deadlock while bounding runtime, raw bytes, line length, and emitted records.

Two nonblocking parent pipes share a selector. The context-managed iterator
preserves order within each stream, labels every binary line, emits a final EOF
fragment explicitly, and owns termination and reaping when iteration stops
early or a limit fails.

## When to Use

Use this integration for a small POSIX command whose output must be processed
incrementally and whose stdout and stderr are both relevant. Pass a validated
argv sequence rather than a shell command, choose limits before spawning, and
decode each returned byte string under an explicit application encoding.

Prefer `subprocess.Popen` in normal application code, especially when
portability matters. This lower-level version is useful when the lifecycle and
selector invariants themselves are the subject. Use an async subprocess API
inside an event loop, and a dedicated process supervisor for process groups,
services, interactive terminals, or descendant management.

## Implementation

```python
import math
import os
import selectors
import signal
import time
from collections import deque
from dataclasses import dataclass
from types import TracebackType
from typing import Literal, Self


_MAX_ARGUMENTS = 64
_MAX_ARGUMENT_BYTES = 64 * 1024
_MAX_TIMEOUT_SECONDS = 86_400.0
_MAX_CHUNK_BYTES = 64 * 1024
_MAX_LINE_BYTES = 1024 * 1024
_MAX_TOTAL_BYTES = 64 * 1024 * 1024
_MAX_LINES = 1_000_000
_MAX_TERMINATION_GRACE = 5.0


@dataclass(frozen=True, slots=True)
class ProcessLine:
    stream: Literal["stdout", "stderr"]
    data: bytes
    ended_with_newline: bool


class PosixProcessError(RuntimeError):
    pass


class ProcessOutputLimitError(PosixProcessError):
    pass


class ProcessStreamTimeoutError(PosixProcessError, TimeoutError):
    pass


class ProcessExitError(PosixProcessError):
    def __init__(self, returncode: int) -> None:
        self.returncode = returncode
        super().__init__(f"the process exited with status {returncode}")


def _bounded_integer(value: int, *, name: str, lower: int, upper: int) -> int:
    if isinstance(value, bool) or not isinstance(value, int):
        raise TypeError(f"{name} must be an integer")
    if not lower <= value <= upper:
        raise ValueError(f"{name} is outside the supported range")
    return value


def _bounded_duration(value: float, *, name: str, maximum: float) -> float:
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError(f"{name} must be numeric")
    try:
        normalized = float(value)
    except OverflowError as error:
        raise ValueError(f"{name} must be finite") from error
    if not math.isfinite(normalized) or not 0.0 < normalized <= maximum:
        raise ValueError(f"{name} must be positive and bounded")
    return normalized


def _validated_argv(argv: tuple[str, ...]) -> tuple[str, ...]:
    if not isinstance(argv, tuple):
        raise TypeError("argv must be a tuple")
    if not 1 <= len(argv) <= _MAX_ARGUMENTS:
        raise ValueError("argv has an unsupported number of arguments")
    encoded_bytes = 0
    for index, argument in enumerate(argv):
        if not isinstance(argument, str):
            raise TypeError("argv entries must be strings")
        if "\x00" in argument or (index == 0 and not argument):
            raise ValueError("argv contains an invalid entry")
        encoded_bytes += len(os.fsencode(argument)) + 1
    if encoded_bytes > _MAX_ARGUMENT_BYTES:
        raise ValueError("argv is too large")
    return argv


class PosixProcessLines:
    def __init__(
        self,
        argv: tuple[str, ...],
        *,
        timeout: float,
        max_lines: int,
        max_line_bytes: int,
        max_total_bytes: int,
        chunk_size: int = 8192,
        termination_grace: float = 0.25,
    ) -> None:
        self._argv = _validated_argv(argv)
        self._timeout = _bounded_duration(
            timeout,
            name="timeout",
            maximum=_MAX_TIMEOUT_SECONDS,
        )
        self._max_lines = _bounded_integer(
            max_lines,
            name="max_lines",
            lower=1,
            upper=_MAX_LINES,
        )
        self._max_line_bytes = _bounded_integer(
            max_line_bytes,
            name="max_line_bytes",
            lower=1,
            upper=_MAX_LINE_BYTES,
        )
        self._max_total_bytes = _bounded_integer(
            max_total_bytes,
            name="max_total_bytes",
            lower=0,
            upper=_MAX_TOTAL_BYTES,
        )
        self._chunk_size = _bounded_integer(
            chunk_size,
            name="chunk_size",
            lower=1,
            upper=_MAX_CHUNK_BYTES,
        )
        self._termination_grace = _bounded_duration(
            termination_grace,
            name="termination_grace",
            maximum=_MAX_TERMINATION_GRACE,
        )
        self._selector: selectors.BaseSelector | None = None
        self._pid: int | None = None
        self._deadline = 0.0
        self._entered = False
        self._closed = False
        self._completed = False
        self._returncode: int | None = None
        self._open_fds: dict[Literal["stdout", "stderr"], int] = {}
        self._buffers = {
            "stdout": bytearray(),
            "stderr": bytearray(),
        }
        self._pending: deque[ProcessLine] = deque()
        self._total_bytes = 0
        self._line_count = 0

    @property
    def returncode(self) -> int | None:
        return self._returncode

    def _require_posix_spawn(self) -> None:
        required_names = (
            "POSIX_SPAWN_CLOSE",
            "POSIX_SPAWN_DUP2",
            "POSIX_SPAWN_OPEN",
            "posix_spawnp",
        )
        if os.name != "posix" or any(not hasattr(os, name) for name in required_names):
            raise OSError("this iterator requires POSIX posix_spawnp support")

    def __enter__(self) -> Self:
        if self._entered or self._closed:
            raise PosixProcessError("the process context is not reusable")
        self._entered = True
        self._require_posix_spawn()
        self._deadline = time.monotonic() + self._timeout

        owned_fds: set[int] = set()
        try:
            stdout_read, stdout_write = os.pipe()
            owned_fds.update((stdout_read, stdout_write))
            stderr_read, stderr_write = os.pipe()
            owned_fds.update((stderr_read, stderr_write))
            if any(descriptor <= 2 for descriptor in owned_fds):
                raise OSError("standard descriptors 0, 1, and 2 must be open")
            os.set_blocking(stdout_read, False)
            os.set_blocking(stderr_read, False)
            self._selector = selectors.DefaultSelector()
            self._selector.register(stdout_read, selectors.EVENT_READ, "stdout")
            self._open_fds["stdout"] = stdout_read
            owned_fds.remove(stdout_read)
            self._selector.register(stderr_read, selectors.EVENT_READ, "stderr")
            self._open_fds["stderr"] = stderr_read
            owned_fds.remove(stderr_read)
            file_actions = (
                (os.POSIX_SPAWN_OPEN, 0, os.devnull, os.O_RDONLY, 0),
                (os.POSIX_SPAWN_DUP2, stdout_write, 1),
                (os.POSIX_SPAWN_DUP2, stderr_write, 2),
                (os.POSIX_SPAWN_CLOSE, stdout_read),
                (os.POSIX_SPAWN_CLOSE, stderr_read),
                (os.POSIX_SPAWN_CLOSE, stdout_write),
                (os.POSIX_SPAWN_CLOSE, stderr_write),
            )
            self._pid = os.posix_spawnp(
                self._argv[0],
                self._argv,
                os.environ.copy(),
                file_actions=file_actions,
                setsigmask=(),
                setsigdef=(signal.SIGPIPE, signal.SIGTERM),
            )
            owned_fds.remove(stdout_write)
            os.close(stdout_write)
            owned_fds.remove(stderr_write)
            os.close(stderr_write)
        except BaseException:
            while owned_fds:
                descriptor = owned_fds.pop()
                try:
                    os.close(descriptor)
                except OSError:
                    pass
            self.close()
            raise
        return self

    def __iter__(self) -> Self:
        return self

    def _emit(
        self,
        stream: Literal["stdout", "stderr"],
        data: bytes,
        *,
        ended_with_newline: bool,
    ) -> None:
        if len(data) > self._max_line_bytes:
            raise ProcessOutputLimitError("a process line is too long")
        if self._line_count >= self._max_lines:
            raise ProcessOutputLimitError("the process emitted too many lines")
        self._line_count += 1
        self._pending.append(ProcessLine(stream, data, ended_with_newline))

    def _consume(
        self,
        stream: Literal["stdout", "stderr"],
        chunk: bytes,
    ) -> None:
        if len(chunk) > self._max_total_bytes - self._total_bytes:
            raise ProcessOutputLimitError("the process emitted too many bytes")
        self._total_bytes += len(chunk)
        buffer = self._buffers[stream]
        buffer.extend(chunk)
        while True:
            newline = buffer.find(b"\n")
            if newline < 0:
                if len(buffer) > self._max_line_bytes:
                    raise ProcessOutputLimitError("a process line is too long")
                return
            line = bytes(buffer[:newline])
            del buffer[: newline + 1]
            self._emit(stream, line, ended_with_newline=True)

    def _close_stream(self, stream: Literal["stdout", "stderr"]) -> None:
        descriptor = self._open_fds.pop(stream)
        if self._selector is not None:
            try:
                self._selector.unregister(descriptor)
            except (KeyError, ValueError):
                pass
        os.close(descriptor)
        buffer = self._buffers[stream]
        if buffer:
            self._emit(stream, bytes(buffer), ended_with_newline=False)
            buffer.clear()

    def _record_wait_status(self, status: int) -> None:
        self._returncode = os.waitstatus_to_exitcode(status)
        self._pid = None

    def _wait_until(self, deadline: float) -> bool:
        while self._pid is not None:
            try:
                waited_pid, status = os.waitpid(self._pid, os.WNOHANG)
            except InterruptedError:
                continue
            if waited_pid:
                self._record_wait_status(status)
                return True
            remaining = deadline - time.monotonic()
            if remaining <= 0.0:
                return False
            time.sleep(min(0.01, remaining))
        return True

    def _finish_after_eof(self) -> None:
        if not self._wait_until(self._deadline):
            raise ProcessStreamTimeoutError(
                "the process did not exit before the deadline"
            )
        self._completed = True
        self._close_selector()
        self._closed = True
        if self._returncode != 0:
            raise ProcessExitError(self._returncode or 0)

    def _close_selector(self) -> None:
        for descriptor in tuple(self._open_fds.values()):
            try:
                os.close(descriptor)
            except OSError:
                pass
        self._open_fds.clear()
        if self._selector is not None:
            self._selector.close()
            self._selector = None

    def _terminate_and_reap(self) -> None:
        pid = self._pid
        if pid is None:
            return
        if self._wait_until(time.monotonic()):
            return
        try:
            os.kill(pid, signal.SIGTERM)
        except ProcessLookupError:
            pass
        if self._wait_until(time.monotonic() + self._termination_grace):
            return
        try:
            os.kill(pid, signal.SIGKILL)
        except ProcessLookupError:
            pass
        while self._pid is not None:
            try:
                waited_pid, status = os.waitpid(self._pid, 0)
            except InterruptedError:
                continue
            if waited_pid:
                self._record_wait_status(status)

    def __next__(self) -> ProcessLine:
        if not self._entered:
            raise PosixProcessError("enter the process context before iterating")
        if self._closed:
            raise StopIteration
        try:
            while True:
                remaining = self._deadline - time.monotonic()
                if remaining <= 0.0:
                    raise ProcessStreamTimeoutError(
                        "the process stream deadline was reached"
                    )
                if self._pending:
                    return self._pending.popleft()
                if not self._open_fds:
                    self._finish_after_eof()
                    raise StopIteration
                if self._selector is None:
                    raise AssertionError("an open process must own a selector")
                events = self._selector.select(remaining)
                if not events:
                    continue
                for key, _mask in events:
                    stream = key.data
                    if stream not in ("stdout", "stderr"):
                        raise AssertionError("selector stream label is invalid")
                    try:
                        remaining_bytes = self._max_total_bytes - self._total_bytes
                        chunk = os.read(
                            key.fd,
                            min(self._chunk_size, remaining_bytes + 1),
                        )
                    except BlockingIOError:
                        continue
                    if chunk:
                        self._consume(stream, chunk)
                    else:
                        self._close_stream(stream)
        except StopIteration:
            raise
        except BaseException as primary_error:
            try:
                self.close()
            except BaseException as cleanup_error:
                primary_error.add_note(
                    f"process cleanup also failed: {cleanup_error!r}"
                )
            raise

    def close(self) -> None:
        if self._closed:
            return
        self._closed = True
        try:
            self._close_selector()
        finally:
            if not self._completed:
                self._terminate_and_reap()

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc: BaseException | None,
        traceback: TracebackType | None,
    ) -> None:
        try:
            self.close()
        except BaseException as cleanup_error:
            if exc_type is None:
                raise
            if exc is not None:
                exc.add_note(f"process cleanup also failed: {cleanup_error!r}")
```

## Example

```python
import sys


program = (
    "import os; "
    "os.write(1, b'out-1\\npartial'); "
    "os.write(2, b'err-1\\n')"
)
stream = PosixProcessLines(
    (sys.executable, "-I", "-S", "-c", program),
    timeout=2.0,
    max_lines=5,
    max_line_bytes=20,
    max_total_bytes=100,
)
with stream:
    lines = list(stream)

stdout_lines = [
    (line.data, line.ended_with_newline)
    for line in lines
    if line.stream == "stdout"
]
stderr_lines = [
    (line.data, line.ended_with_newline)
    for line in lines
    if line.stream == "stderr"
]

failing = PosixProcessLines(
    (sys.executable, "-I", "-S", "-c", "raise SystemExit(3)"),
    timeout=2.0,
    max_lines=1,
    max_line_bytes=10,
    max_total_bytes=10,
)
try:
    with failing:
        list(failing)
except ProcessExitError as error:
    failing_returncode = error.returncode
else:
    failing_returncode = 0

slow_program = (
    "import os, time; "
    "os.write(1, b'ready\\n'); "
    "time.sleep(5)"
)
early = PosixProcessLines(
    (sys.executable, "-I", "-S", "-c", slow_program),
    timeout=2.0,
    max_lines=2,
    max_line_bytes=10,
    max_total_bytes=20,
    termination_grace=0.1,
)
with early:
    first = next(early)
early.close()

assert (
    stdout_lines,
    stderr_lines,
    stream.returncode,
    failing_returncode,
    first,
    early.returncode is not None,
) == (
    [(b"out-1", True), (b"partial", False)],
    [(b"err-1", True)],
    0,
    3,
    ProcessLine("stdout", b"ready", True),
    True,
)
```

## Trade-offs and Limitations

This implementation is POSIX-only. Pipe readiness through `selectors` is not a
portable Windows subprocess contract. It inherits the current environment and
working directory, replaces stdin with the null device, and does not invoke a
shell. It preserves byte order within each pipe but makes no causal ordering
claim between stdout and stderr; selector readiness can interleave them either
way. A carriage return before a newline remains part of the binary line.

The executable, argv, and lookup environment must still be trusted: avoiding a
shell does not make an attacker-selected program safe. Prefer an absolute
executable path or a controlled `PATH`. The child receives the complete current
environment, including any credentials stored there. Process creation cannot
be interrupted by the deadline. The parent must keep descriptors 0, 1, and 2
open, own iteration from one thread, and remain the child's only reaper; a
SIGCHLD handler or other waiter breaks that ownership. Explicitly inheritable
unrelated descriptors may reach the child on platforms without a close-from
spawn action, so this is not a sandbox or privilege boundary.

The overall monotonic deadline includes consumer pauses, and limits count raw
pipe bytes including newlines. Reaching EOF on both pipes is followed by process
reaping, so a child that closes its output but keeps running can still time out.
Early close and failures close the read pipes, send `SIGTERM`, wait for a bounded
grace period, then send `SIGKILL` and perform a mandatory reap. The final wait
after `SIGKILL` can still be delayed by the operating system. Only the direct
child is owned; commands that daemonize or leave descendants, process groups,
signal forwarding, terminal emulation, encoding, and callbacks remain outside
this snippet.

## Related Snippets

<!-- catalog:related:start -->
- [Drain Bounded Deferred Writes Outside the Queue Lock](drain-bounded-deferred-writes-outside-the-queue-lock.md)
- [Stop a Polling Worker Cooperatively with an Event](stop-a-polling-worker-cooperatively-with-an-event.md)
- [Limit Text Lines Across Arbitrary Chunks](../data-processing/limit-text-lines-across-arbitrary-chunks.md)
<!-- catalog:related:end -->
