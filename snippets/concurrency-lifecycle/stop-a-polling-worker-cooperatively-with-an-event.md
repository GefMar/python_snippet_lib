---
title: "Stop a Polling Worker Cooperatively with an Event"
snippet_type: pattern
use_cases:
  - concurrency-control
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - collect-thread-pool-results-and-errors-as-futures-complete.md
  - ../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md
---

# Stop a Polling Worker Cooperatively with an Event

## Idea and Problem

Run a bounded polling operation until another thread requests shutdown through a shared event that also interrupts idle waiting.

Checking the event before each operation avoids starting when shutdown was
already requested. Calling `Event.wait(interval)` instead of `sleep(interval)`
lets a later request wake the worker immediately rather than waiting for the
whole polling interval.

## When to Use

Use this loop as the target of an owned, non-daemon thread whose `work_once`
call is finite and leaves resources consistent before returning. Share one
`Event` with the lifecycle owner, set it during shutdown, and join the thread.
Use a queue, executor, or async task group when work submission, draining,
results, or cancellation need a richer ownership policy.

## Implementation

```python
import math
from collections.abc import Callable
from threading import Event, TIMEOUT_MAX


def run_polling_worker(
    stop_event: Event,
    work_once: Callable[[], None],
    *,
    interval: int | float,
) -> int:
    if not isinstance(stop_event, Event):
        raise TypeError("stop_event must be a threading.Event")
    if not callable(work_once):
        raise TypeError("work_once must be callable")
    if isinstance(interval, bool) or not isinstance(interval, (int, float)):
        raise TypeError("interval must be an integer or float")
    try:
        interval_value = float(interval)
    except OverflowError as error:
        raise ValueError("interval must be representable as a float") from error
    if not math.isfinite(interval_value) or interval_value <= 0:
        raise ValueError("interval must be finite and positive")
    if interval_value > TIMEOUT_MAX:
        raise ValueError("interval exceeds the platform timeout limit")

    completed_calls = 0
    while not stop_event.is_set():
        work_once()
        completed_calls += 1
        if stop_event.wait(interval_value):
            break
    return completed_calls
```

## Example

```python
from threading import Thread


stop_event = Event()
started = Event()
calls = []
completed = []


def work_once() -> None:
    calls.append("polled")
    started.set()


def run() -> None:
    completed.append(
        run_polling_worker(stop_event, work_once, interval=60)
    )


worker = Thread(target=run, name="example-poller", daemon=False)
worker.start()
assert started.wait(1)
stop_event.set()
worker.join(1)

already_stopped = Event()
already_stopped.set()
skipped = run_polling_worker(already_stopped, work_once, interval=1)


class WorkFailed(Exception):
    pass


def fail() -> None:
    raise WorkFailed


try:
    run_polling_worker(Event(), fail, interval=1)
except WorkFailed:
    failure_propagated = True
else:
    failure_propagated = False

assert (
    worker.is_alive(),
    calls,
    completed,
    skipped,
    failure_propagated,
) == (False, ["polled"], [1], 0, True)
```

## Trade-offs and Limitations

Shutdown is cooperative and cannot interrupt an in-flight or hung
`work_once`; that operation needs its own timeouts and cancellation policy. A
request racing with the pre-call check can allow one final operation to start.
The first call runs immediately, later starts are interval-based rather than a
fixed-rate schedule, and ordinary exceptions propagate out of the loop. When
used as a thread target, arrange an error-reporting channel or a suitable
`threading.excepthook`. Queue draining, sentinels, signal installation, daemon
policy, retries, and resource cleanup remain lifecycle-owner responsibilities.

## Related Snippets

<!-- catalog:related:start -->
- [Collect Thread-Pool Results and Errors as Futures Complete](collect-thread-pool-results-and-errors-as-futures-complete.md)
- [Wait for a Predicate Until a Monotonic Deadline](../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md)
<!-- catalog:related:end -->
