---
title: "Run One Trusted Stateless Function in a Short-Lived Subinterpreter"
snippet_type: pattern
use_cases:
  - lifecycle-management
  - resource-management
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - submit-a-callable-with-a-snapshot-of-the-current-context.md
  - map-a-large-iterable-with-a-bounded-thread-pool-submission-buffer.md
  - run-bounded-weighted-jobs-under-shared-process-capacity.md
---

# Run One Trusted Stateless Function in a Short-Lived Subinterpreter

## Idea and Problem

Run one closed, trusted function with bounded immutable values in a fresh interpreter and destroy that interpreter on every exit path.

`concurrent.interpreters` can transfer a top-level function without a closure
and execute it against another interpreter's isolated Python state. The call is
synchronous in the current thread. A narrow allowlist keeps arbitrary
callables outside the boundary, while `finally` makes the expensive interpreter
lifecycle explicit even when child execution fails.

## When to Use

Use this pattern to probe or isolate Python interpreter state for one small,
trusted operation on CPython 3.14 or later. The example makes isolation visible
by adding a synthetic entry to the child interpreter's `sys.modules`; the
entry never appears in the parent. Keep inputs and results to a closed profile
of small immutable built-in values.

Use an `InterpreterPoolExecutor` when repeated calls justify a managed pool,
and add threads when actual concurrent execution is required. Use a process or
another security boundary for hostile code, crash isolation, enforceable
resource limits, or dependencies that are not safe in multiple interpreters.

## Implementation

```python
import sys
from collections.abc import Callable
from concurrent import interpreters

_MAX_ISOLATED_PAYLOAD_BYTES = 16 * 1_024


def _reverse_and_mark_child(payload: bytes) -> tuple[bytes, bool]:
    import sys

    marker = "_subinterpreter_local_marker_example"
    sys.modules[marker] = object()
    return payload[::-1], marker in sys.modules


def _raise_synthetic_child_failure(_: bytes) -> None:
    raise RuntimeError("synthetic child failure")


_TRUSTED_STATELESS_CALLS = (
    _reverse_and_mark_child,
    _raise_synthetic_child_failure,
)


def _call_in_fresh_interpreter(
    function: Callable[[bytes], object],
    payload: bytes,
) -> object:
    if not any(function is allowed for allowed in _TRUSTED_STATELESS_CALLS):
        raise TypeError("function is not in the trusted stateless allowlist")
    if type(payload) is not bytes:
        raise TypeError("payload must be exact bytes")
    if len(payload) > _MAX_ISOLATED_PAYLOAD_BYTES:
        raise ValueError("payload exceeds the supported byte limit")

    interpreter = interpreters.create()
    try:
        return interpreter.call(function, payload)
    finally:
        interpreter.close()


def run_isolated_reverse(payload: bytes) -> tuple[bytes, bool]:
    result = _call_in_fresh_interpreter(_reverse_and_mark_child, payload)
    if type(result) is not tuple or len(result) != 2:
        raise TypeError("child result must contain bytes and one Boolean")
    transformed, marker_was_local = result
    if type(transformed) is not bytes:
        raise TypeError("child payload result must be exact bytes")
    if len(transformed) > _MAX_ISOLATED_PAYLOAD_BYTES:
        raise ValueError("child payload result exceeds the byte limit")
    if type(marker_was_local) is not bool:
        raise TypeError("child marker result must be an exact Boolean")
    return transformed, marker_was_local
```

## Example

```python
def live_interpreter_ids() -> frozenset[int]:
    return frozenset(interpreter.id for interpreter in interpreters.list_all())


marker = "_subinterpreter_local_marker_example"
assert marker not in sys.modules

before = live_interpreter_ids()
transformed = run_isolated_reverse(b"state")
after_success = live_interpreter_ids()

failure_was_transferred = False
try:
    _call_in_fresh_interpreter(_raise_synthetic_child_failure, b"")
except interpreters.ExecutionFailed:
    failure_was_transferred = True
after_failure = live_interpreter_ids()

assert (
    transformed,
    marker in sys.modules,
    after_success,
    failure_was_transferred,
    after_failure,
) == (
    (b"etats", True),
    False,
    before,
    True,
    before,
)
```

## Trade-offs and Limitations

Input and result validation take `O(b)` time for at most 16 KiB of bytes. The
interpreter startup, trusted function work, result transfer, finalizers, and
teardown have opaque time and memory costs and are not bounded by that byte
limit. Output validation happens only after the child has completed. A child
failure is reported as `ExecutionFailed`; an unsupported transfer may raise
`NotShareableError`; and an interpreter lifecycle failure propagates.

The allowlist is deliberately closed, and its functions are top-level
non-closures that do not depend on module globals. The fixed bytes input and
small immutable tuple result avoid accepting mutable caller objects or
arbitrary serialization payloads. Adapt the allowlist only after reviewing the
function, its imports, its result types, and every extension module it uses.

A subinterpreter isolates Python state such as imports; it does not start a new
thread, provide parallelism by itself, enforce a timeout, cancel stuck work, or
form a sandbox. Interpreters share a process, so environment variables, file
descriptors, the working directory, native libraries, memory corruption, and
other process-wide effects can cross the apparent boundary. The API is not
available on WASI, and not every extension supports multiple interpreters.

## Related Snippets

<!-- catalog:related:start -->
- [Submit a Callable with a Snapshot of the Current Context](submit-a-callable-with-a-snapshot-of-the-current-context.md)
- [Map a Large Iterable with a Bounded Thread-Pool Submission Buffer](map-a-large-iterable-with-a-bounded-thread-pool-submission-buffer.md)
- [Run Bounded Weighted Jobs Under Shared Process Capacity](run-bounded-weighted-jobs-under-shared-process-capacity.md)
<!-- catalog:related:end -->
