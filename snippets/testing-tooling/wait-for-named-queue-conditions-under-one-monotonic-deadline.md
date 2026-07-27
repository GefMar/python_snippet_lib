---
title: "Wait for Named Queue Conditions Under One Monotonic Deadline"
snippet_type: testing-technique
use_cases:
  - concurrency-control
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../concurrency-lifecycle/collect-thread-pool-results-and-errors-as-futures-complete.md
  - ../concurrency-lifecycle/stop-a-polling-worker-cooperatively-with-an-event.md
  - ../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md
---

# Wait for Named Queue Conditions Under One Monotonic Deadline

## Idea and Problem

Wait for several order-independent message conditions while sharing one monotonic timeout and a finite consumption budget.

Conditions keep stable names and input order rather than being placed in a
set. One dequeued message may satisfy several still-pending conditions, and
the first matching message is retained for each. Every consumed queue item is
paired with `task_done()`, including when a predicate raises.

## When to Use

Use this testing helper when a component emits a finite stream of in-process
messages in nondeterministic order and a test needs several observable states.
Give it exclusive ownership of the queue during the wait, and keep predicates
pure and quick. Use callbacks, condition variables, or an async primitive in
production code instead of consuming and discarding unrelated messages here.

## Implementation

```python
import math
import re
import time
from collections.abc import Callable, Iterable
from dataclasses import dataclass
from queue import Empty, Queue
from typing import Generic, TypeVar


MessageT = TypeVar("MessageT")
_CONDITION_NAME = re.compile(r"[a-z][a-z0-9_-]{0,31}", re.ASCII)
_MAX_TIMEOUT_SECONDS = 86_400.0


@dataclass(frozen=True, slots=True)
class QueueCondition(Generic[MessageT]):
    name: str
    matches: Callable[[MessageT], bool]


@dataclass(frozen=True, slots=True)
class ConditionMatch(Generic[MessageT]):
    name: str
    message: MessageT


class QueueConditionsTimeout(TimeoutError):
    def __init__(self, pending_names: tuple[str, ...]) -> None:
        self.pending_names = pending_names
        super().__init__("queue conditions were not met before the deadline")


class QueueMessageBudgetExceeded(RuntimeError):
    def __init__(self, pending_names: tuple[str, ...]) -> None:
        self.pending_names = pending_names
        super().__init__("queue message budget was exhausted")


def wait_for_queue_conditions(
    messages: Queue[MessageT],
    conditions: Iterable[QueueCondition[MessageT]],
    *,
    timeout: int | float,
    max_messages: int = 1000,
) -> tuple[ConditionMatch[MessageT], ...]:
    if not isinstance(messages, Queue):
        raise TypeError("messages must be a queue.Queue")
    if isinstance(timeout, bool) or not isinstance(timeout, (int, float)):
        raise TypeError("timeout must be numeric")
    try:
        timeout_value = float(timeout)
    except OverflowError:
        raise ValueError("timeout must be representable as a float") from None
    if not math.isfinite(timeout_value) or timeout_value <= 0:
        raise ValueError("timeout must be finite and positive")
    if timeout_value > _MAX_TIMEOUT_SECONDS:
        raise ValueError("timeout exceeds the supported duration")
    if isinstance(max_messages, bool) or not isinstance(max_messages, int):
        raise TypeError("max_messages must be an integer")
    if not 1 <= max_messages <= 100_000:
        raise ValueError("max_messages is outside the supported range")

    ordered: list[QueueCondition[MessageT]] = []
    names: set[str] = set()
    for condition in conditions:
        if len(ordered) >= 32:
            raise ValueError("conditions exceed the supported count")
        if not isinstance(condition, QueueCondition):
            raise TypeError("conditions must contain QueueCondition values")
        if _CONDITION_NAME.fullmatch(condition.name) is None:
            raise ValueError("condition name is invalid")
        if condition.name in names:
            raise ValueError("condition names must be unique")
        if not callable(condition.matches):
            raise TypeError("condition matches must be callable")
        names.add(condition.name)
        ordered.append(condition)

    if not ordered:
        return ()

    deadline = time.monotonic() + timeout_value
    if not math.isfinite(deadline):
        raise ValueError("timeout produces an invalid deadline")

    pending = list(range(len(ordered)))
    matches: list[ConditionMatch[MessageT] | None] = [None] * len(ordered)
    consumed = 0
    while pending:
        pending_names = tuple(ordered[index].name for index in pending)
        if consumed >= max_messages:
            raise QueueMessageBudgetExceeded(pending_names)

        remaining = deadline - time.monotonic()
        if remaining <= 0:
            raise QueueConditionsTimeout(pending_names)
        try:
            message = messages.get(timeout=remaining)
        except Empty:
            raise QueueConditionsTimeout(pending_names) from None

        try:
            matched_now = [
                index
                for index in pending
                if ordered[index].matches(message)
            ]
        finally:
            messages.task_done()
        consumed += 1
        if time.monotonic() >= deadline:
            raise QueueConditionsTimeout(pending_names)

        for index in matched_now:
            matches[index] = ConditionMatch(ordered[index].name, message)
        if matched_now:
            matched_indexes = set(matched_now)
            pending = [index for index in pending if index not in matched_indexes]

    result: list[ConditionMatch[MessageT]] = []
    for match in matches:
        assert match is not None
        result.append(match)
    return tuple(result)
```

## Example

```python
messages: Queue[dict[str, object]] = Queue()
messages.put({"kind": "noise", "value": 0})
messages.put({"kind": "complete", "value": 3})
messages.put({"kind": "ready", "value": 10})


def has_kind(expected: str) -> Callable[..., bool]:
    return lambda message: message.get("kind") == expected


def has_positive_value(message: dict[str, object]) -> bool:
    value = message.get("value")
    return isinstance(value, int) and value > 0


matched = wait_for_queue_conditions(
    messages,
    (
        QueueCondition("ready", has_kind("ready")),
        QueueCondition("complete", has_kind("complete")),
        QueueCondition("positive", has_positive_value),
    ),
    timeout=1,
    max_messages=3,
)
messages.join()

assert tuple(
    (match.name, match.message["kind"])
    for match in matched
) == (
    ("ready", "ready"),
    ("complete", "complete"),
    ("positive", "complete"),
)
```

## Trade-offs and Limitations

Every dequeued item is consumed permanently, including unmatched messages, so
another consumer must not share this queue. `task_done()` is always called;
the helper therefore owns that part of the queue's accounting contract.
Partial matches are lost when a timeout, message-budget error, or predicate
exception escapes. Predicates run sequentially and cannot be interrupted, so a
slow or hung predicate can cross the deadline; a timeout is raised only after
that call returns. The function retains one full message per condition, uses
`O(messages * conditions)` predicate calls, and caps one wait at 86,400 seconds
to avoid platform timeout overflows. It is not an event router, delivery
guarantee, cancellation mechanism, or async-queue adapter.

## Related Snippets

<!-- catalog:related:start -->
- [Collect Thread-Pool Results and Errors as Futures Complete](../concurrency-lifecycle/collect-thread-pool-results-and-errors-as-futures-complete.md)
- [Stop a Polling Worker Cooperatively with an Event](../concurrency-lifecycle/stop-a-polling-worker-cooperatively-with-an-event.md)
- [Wait for a Predicate Until a Monotonic Deadline](../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md)
<!-- catalog:related:end -->
