---
title: "Race a Preferred Async Read Against Bounded Alternatives"
snippet_type: pattern
use_cases:
  - concurrency-control
  - lifecycle-management
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - reuse-one-pending-future-across-non-cancelling-poll-timeouts.md
  - gather-async-results-with-bounded-concurrency.md
  - collect-a-bounded-thread-pool-batch-under-one-deadline.md
---

# Race a Preferred Async Read Against Bounded Alternatives

## Idea and Problem

Give one preferred asynchronous read a short head start, then race it against bounded alternatives without abandoning the still-pending preferred read.

The helper starts the preferred target once. An ordinary preferred failure or
the grace deadline starts every alternative, while an overall deadline bounds
result observation. Failures do not stop the remaining attempts, and successes
that are ready in the same scheduler batch are ranked by declared target order.
Success, exhaustion, overall timeout, and caller cancellation all cancel and
await every owned losing task before the helper finishes.

## When to Use

Use this pattern only when the caller has certified that every target performs
the same idempotent read and concurrent attempts cannot create, mutate, or
consume remote state. The callback must be trusted, event-loop-local, and
cancellation-cooperative. Its successful value must remain usable after the
losing tasks have finished their `finally` blocks.

Pass one exact tuple containing 1-31 unique alternatives. All target IDs,
timeouts, and the callback are validated before any callback task is created.
The overall timeout must be greater than the grace timeout so alternatives
receive a non-zero observation window.

## Implementation

```python
import asyncio
import math
import re
from collections.abc import Awaitable, Callable
from dataclasses import dataclass

_MAX_TARGETS = 32
_MAX_TARGET_ID_BYTES = 64
_MAX_TIMEOUT_SECONDS = 300.0
_TARGET_ID = re.compile(r"[a-z][a-z0-9._-]{0,63}", re.ASCII)


class AllReadsFailedError(RuntimeError):
    def __init__(self, failed_targets: tuple[str, ...]) -> None:
        self.failed_targets = tuple(failed_targets)
        super().__init__("all read attempts failed")


class ReadRaceTimeoutError(TimeoutError):
    def __init__(
        self,
        *,
        failed_targets: tuple[str, ...],
        pending_targets: tuple[str, ...],
    ) -> None:
        self.failed_targets = tuple(failed_targets)
        self.pending_targets = tuple(pending_targets)
        super().__init__("the overall read-race timeout expired")


@dataclass(frozen=True, slots=True)
class ReadRaceResult[ValueT]:
    target_id: str
    value: ValueT


def _validated_target(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not 1 <= len(value) <= _MAX_TARGET_ID_BYTES:
        raise ValueError(f"{field} length is outside the supported range")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError as error:
        raise ValueError(f"{field} must be valid UTF-8 text") from error
    if len(encoded) > _MAX_TARGET_ID_BYTES:
        raise ValueError(f"{field} exceeds the UTF-8 byte limit")
    if _TARGET_ID.fullmatch(value) is None:
        raise ValueError(f"{field} must be a conservative ASCII identifier")
    return value


def _positive_seconds(value: object, *, field: str) -> float:
    if type(value) not in (int, float):
        raise TypeError(f"{field} must be an integer or float")
    try:
        seconds = float(value)
    except OverflowError as error:
        raise ValueError(f"{field} must be representable as a float") from error
    if not math.isfinite(seconds) or seconds <= 0:
        raise ValueError(f"{field} must be finite and positive")
    if seconds > _MAX_TIMEOUT_SECONDS:
        raise ValueError(f"{field} exceeds the supported timeout")
    return seconds


async def _cancel_and_drain(
    tasks: tuple[asyncio.Task[object], ...],
) -> None:
    for task in tasks:
        if not task.done():
            task.cancel()
    drain = asyncio.gather(*tasks, return_exceptions=True)
    interrupted = False
    while not drain.done():
        try:
            await asyncio.shield(drain)
        except asyncio.CancelledError:
            interrupted = True
    if interrupted:
        raise asyncio.CancelledError


async def race_preferred_read[ValueT](
    preferred_target: str,
    alternative_targets: tuple[str, ...],
    *,
    grace_timeout: int | float,
    overall_timeout: int | float,
    read: Callable[[str], Awaitable[ValueT]],
) -> ReadRaceResult[ValueT]:
    preferred = _validated_target(preferred_target, field="preferred_target")
    if type(alternative_targets) is not tuple:
        raise TypeError("alternative_targets must be an exact tuple")
    if not 1 <= len(alternative_targets) < _MAX_TARGETS:
        raise ValueError("alternative target count is outside the supported range")

    alternatives: list[str] = []
    known = {preferred}
    for position, raw_target in enumerate(alternative_targets):
        target = _validated_target(
            raw_target,
            field=f"alternative_targets[{position}]",
        )
        if target in known:
            raise ValueError("target IDs must be unique")
        known.add(target)
        alternatives.append(target)

    grace_seconds = _positive_seconds(grace_timeout, field="grace_timeout")
    overall_seconds = _positive_seconds(overall_timeout, field="overall_timeout")
    if grace_seconds >= overall_seconds:
        raise ValueError("grace_timeout must be less than overall_timeout")
    if not callable(read):
        raise TypeError("read must be callable")

    ordered_targets = (preferred, *alternatives)
    order = {target: position for position, target in enumerate(ordered_targets)}
    loop = asyncio.get_running_loop()
    started_at = loop.time()
    grace_deadline = started_at + grace_seconds
    overall_deadline = started_at + overall_seconds

    task_by_target: dict[str, asyncio.Task[ValueT]] = {}
    target_by_task: dict[asyncio.Task[ValueT], str] = {}
    pending: set[asyncio.Task[ValueT]] = set()
    failed: set[str] = set()

    async def invoke(target: str) -> ValueT:
        return await read(target)

    def start(target: str) -> None:
        task = asyncio.create_task(invoke(target), name=f"read-race:{target}")
        task_by_target[target] = task
        target_by_task[task] = target
        pending.add(task)

    def timeout_error() -> ReadRaceTimeoutError:
        return ReadRaceTimeoutError(
            failed_targets=tuple(target for target in ordered_targets if target in failed),
            pending_targets=tuple(
                target
                for target in ordered_targets
                if (task := task_by_target.get(target)) is not None
                and task in pending
                and not task.done()
            ),
        )

    start(preferred)
    alternatives_started = False
    try:
        while pending:
            done = {task for task in pending if task.done()}
            if not done:
                now = loop.time()
                if now >= overall_deadline:
                    raise timeout_error()
                phase_deadline = overall_deadline if alternatives_started else grace_deadline
                done, _ = await asyncio.wait(
                    pending,
                    timeout=max(0.0, phase_deadline - now),
                    return_when=asyncio.FIRST_COMPLETED,
                )
                if not done:
                    if not alternatives_started and loop.time() < overall_deadline:
                        for target in alternatives:
                            start(target)
                        alternatives_started = True
                        continue
                    raise timeout_error()
                done = {task for task in pending if task.done()}

            if loop.time() >= overall_deadline:
                raise timeout_error()
            pending.difference_update(done)
            batch_successes: list[tuple[str, ValueT]] = []
            for task in sorted(
                done,
                key=lambda completed_task: order[target_by_task[completed_task]],
            ):
                target = target_by_task[task]
                try:
                    value = task.result()
                except asyncio.CancelledError:
                    failed.add(target)
                except Exception:
                    failed.add(target)
                else:
                    batch_successes.append((target, value))

            if loop.time() >= overall_deadline:
                raise timeout_error()
            if batch_successes:
                target, value = batch_successes[0]
                return ReadRaceResult(target, value)

            if not alternatives_started and preferred in failed:
                if loop.time() >= overall_deadline:
                    raise timeout_error()
                for target in alternatives:
                    start(target)
                alternatives_started = True

        raise AllReadsFailedError(tuple(target for target in ordered_targets if target in failed))
    finally:
        await _cancel_and_drain(tuple(task_by_target.values()))
```

## Example

```python
import time


class ReadUnavailableError(RuntimeError):
    pass


async def exercise_read_race() -> tuple[object, ...]:
    calls: list[str] = []
    finished: set[str] = set()
    alternatives_ready = asyncio.Event()
    alternative_arrivals = 0

    async def tied_read(target: str) -> str:
        nonlocal alternative_arrivals
        calls.append(target)
        try:
            if target == "preferred":
                await asyncio.Event().wait()
            alternative_arrivals += 1
            if alternative_arrivals == 2:
                alternatives_ready.set()
            await alternatives_ready.wait()
            await asyncio.sleep(0)
            return f"value-from-{target}"
        finally:
            finished.add(target)

    selected = await race_preferred_read(
        "preferred",
        ("alternative-a", "alternative-b"),
        grace_timeout=0.02,
        overall_timeout=0.5,
        read=tied_read,
    )

    rejected_calls: list[str] = []

    async def reject_read(target: str) -> str:
        rejected_calls.append(target)
        raise ReadUnavailableError("detail is deliberately not summarized")

    try:
        await race_preferred_read(
            "first",
            ("second", "third"),
            grace_timeout=0.2,
            overall_timeout=0.5,
            read=reject_read,
        )
    except AllReadsFailedError as error:
        exhaustion = (error.failed_targets, str(error))
    else:
        exhaustion = None

    cancelled_after_timeout: set[str] = set()

    async def held_read(target: str) -> str:
        try:
            await asyncio.Event().wait()
        finally:
            cancelled_after_timeout.add(target)
        return target

    try:
        await race_preferred_read(
            "north",
            ("east", "west"),
            grace_timeout=0.02,
            overall_timeout=0.15,
            read=held_read,
        )
    except ReadRaceTimeoutError as error:
        timeout_state = (error.failed_targets, error.pending_targets)
    else:
        timeout_state = None

    late_calls: list[str] = []

    async def immediate_read(target: str) -> str:
        late_calls.append(target)
        return target

    # Simulate an unrelated callback starving the loop before the read runs.
    asyncio.get_running_loop().call_soon(time.sleep, 0.08)
    try:
        await race_preferred_read(
            "late",
            ("unused",),
            grace_timeout=0.02,
            overall_timeout=0.04,
            read=immediate_read,
        )
    except ReadRaceTimeoutError as error:
        late_timeout_state = (error.failed_targets, error.pending_targets)
    else:
        late_timeout_state = None

    entered = asyncio.Event()
    cancelled_by_caller: list[str] = []

    async def caller_held_read(target: str) -> str:
        entered.set()
        try:
            await asyncio.Event().wait()
        finally:
            cancelled_by_caller.append(target)
        return target

    operation = asyncio.create_task(
        race_preferred_read(
            "solo",
            ("reserve",),
            grace_timeout=0.2,
            overall_timeout=0.5,
            read=caller_held_read,
        )
    )
    await entered.wait()
    operation.cancel()
    try:
        await operation
    except asyncio.CancelledError:
        caller_cancellation_propagated = True
    else:
        caller_cancellation_propagated = False

    return (
        selected,
        calls,
        finished,
        exhaustion,
        rejected_calls,
        timeout_state,
        cancelled_after_timeout,
        late_timeout_state,
        late_calls,
        caller_cancellation_propagated,
        cancelled_by_caller,
    )


assert asyncio.run(exercise_read_race()) == (
    ReadRaceResult("alternative-a", "value-from-alternative-a"),
    ["preferred", "alternative-a", "alternative-b"],
    {"preferred", "alternative-a", "alternative-b"},
    (("first", "second", "third"), "all read attempts failed"),
    ["first", "second", "third"],
    ((), ("north", "east", "west")),
    {"north", "east", "west"},
    ((), ()),
    ["late"],
    True,
    ["solo"],
)
```

## Trade-offs and Limitations

At most 32 tasks and 32 bounded target IDs are retained. Validation and state
use `O(n)` space; ordering completed batches costs at most `O(n log n)` total
for `n` started targets. The deadline uses the event loop's monotonic clock and
bounds result observation, not callback cleanup. A callback that suppresses
cancellation can therefore delay the function while owned tasks are drained.
Even a successful task is rejected when its result is first observed at or
after the overall deadline.
The drain shields its child-collection future across repeated caller
cancellation deliveries, then propagates cancellation after every child ends.

Ordinary `Exception` failures and child-task cancellation are summarized only
by bounded target ID. Their exception messages and objects are not embedded in
`AllReadsFailedError` or `ReadRaceTimeoutError`. Process-control exceptions are
not ordinary attempt failures and still propagate after owned tasks are handled.

Cancellation is local and best effort: cancelling an asyncio task does not
undo a request already accepted remotely. This is why only caller-certified
idempotent reads are suitable. The helper owns no DNS, discovery, health-check,
HTTP, authentication, retries, backoff, connections, or resource cleanup
policy; all of those concerns remain outside the callback race.

## Related Snippets

<!-- catalog:related:start -->
- [Reuse One Pending Future Across Non-Cancelling Poll Timeouts](reuse-one-pending-future-across-non-cancelling-poll-timeouts.md)
- [Gather Async Results with Bounded Concurrency](gather-async-results-with-bounded-concurrency.md)
- [Collect a Bounded Thread-Pool Batch Under One Deadline](collect-a-bounded-thread-pool-batch-under-one-deadline.md)
<!-- catalog:related:end -->
