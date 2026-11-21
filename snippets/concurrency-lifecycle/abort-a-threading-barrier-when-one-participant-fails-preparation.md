---
title: "Abort a Threading Barrier When One Participant Fails Preparation"
snippet_type: pattern
use_cases:
  - concurrency-control
  - lifecycle-management
tested_python:
  - "3.14"
dependencies: []
related:
  - collect-thread-pool-results-and-errors-as-futures-complete.md
  - stop-a-polling-worker-cooperatively-with-an-event.md
---

# Abort a Threading Barrier When One Participant Fails Preparation

## Idea and Problem

Release the rest of a fixed group of threads when one participant cannot reach their shared barrier.

Each participant prepares a value and then waits for the others before the
next stage. If preparation raises, abort the barrier and re-raise the original
exception. Waiting peers and peers that arrive later get `BrokenBarrierError`
instead of waiting for the missing participant.

## When to Use

Use this for a small, fixed group with a separate barrier for one preparation
stage. Ensure each participant has a worker available and reaches the helper.
Allow the coordinator to inspect the initiating worker's exception separately
from peer failures caused by the broken barrier.

## Implementation

```python
from collections.abc import Callable
from threading import Barrier


def prepare_before_barrier[PreparedT](
    barrier: Barrier, prepare: Callable[[], PreparedT]
) -> PreparedT:
    try:
        prepared = prepare()
    except BaseException:
        barrier.abort()
        raise

    barrier.wait()
    return prepared
```

## Example

```python
from concurrent.futures import ThreadPoolExecutor
from threading import BrokenBarrierError


class MissingPart(ValueError):
    pass


def prepare_part(name: str) -> str:
    if not name:
        raise MissingPart("empty part name")
    return name.upper()


with ThreadPoolExecutor(max_workers=2) as pool:
    healthy = Barrier(2, timeout=2)
    left = pool.submit(prepare_before_barrier, healthy, lambda: prepare_part("left"))
    right = pool.submit(prepare_before_barrier, healthy, lambda: prepare_part("right"))
    prepared = (left.result(), right.result())

    stage = Barrier(2)
    peer = pool.submit(prepare_before_barrier, stage, lambda: prepare_part("left"))
    failed = pool.submit(prepare_before_barrier, stage, lambda: prepare_part(""))
    try:
        original_error = failed.exception(timeout=2)
        broken_by_failure = stage.broken
        peer_error = peer.exception(timeout=2)
    finally:
        stage.abort()

assert (
    prepared == ("LEFT", "RIGHT")
    and broken_by_failure
    and isinstance(original_error, MissingPart)
    and str(original_error) == "empty part name"
    and isinstance(peer_error, BrokenBarrierError)
)
```

## Trade-offs and Limitations

Aborting releases barrier waiters. Participants already inside a preparation
call still need to finish or stop cooperatively. Effects produced before the
barrier remain the responsibility of the task or its coordinator. Put work
that depends on the full group after the helper returns successfully.

If a worker can disappear before calling the helper, add a coordinator-driven
abort or a barrier timeout. Use a new barrier for a new stage once the failed
workers have finished. The thread pool must let all participants run together,
since a waiting participant occupies its worker.

## Related Snippets

<!-- catalog:related:start -->
- [Collect Thread-Pool Results and Errors as Futures Complete](collect-thread-pool-results-and-errors-as-futures-complete.md)
- [Stop a Polling Worker Cooperatively with an Event](stop-a-polling-worker-cooperatively-with-an-event.md)
<!-- catalog:related:end -->
