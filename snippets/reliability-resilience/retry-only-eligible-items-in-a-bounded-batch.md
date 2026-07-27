---
title: "Retry Only Eligible Items in a Bounded Batch"
snippet_type: pattern
use_cases:
  - retry-recovery
  - networking
tested_python:
  - "3.14"
dependencies: []
related:
  - poll-a-remote-operation-within-deadline-and-failure-budgets.md
  - ../data-processing/batch-items-by-estimated-byte-size.md
  - ../concurrency-lifecycle/collect-thread-pool-results-and-errors-as-futures-complete.md
---

# Retry Only Eligible Items in a Bounded Batch

## Idea and Problem

Retry only idempotent items with explicitly retryable responses while retaining every item that has already reached a final outcome.

A batch transport can succeed even when some embedded operations fail. Stable
request IDs let the client reconcile each response exactly instead of relying
on position, and separate attempt and deadline budgets prevent a retryable item
from keeping the batch alive indefinitely.

## When to Use

Use this pattern when one finite transport call carries several independent
operations and returns one response for each request ID. The caller must decide
which operations are safe to repeat, and the service must preserve request IDs
in its responses. Successful and non-retryable items are never sent again.

Keep whole-batch transport recovery outside this helper: if `send_batch`
raises, the exception propagates because the client cannot know which
operations the remote side accepted. Use a protocol-specific idempotency key
and recovery policy before retrying an ambiguous transport failure.

## Implementation

```python
import math
import re
import time
from collections.abc import Callable, Iterable, Sequence
from dataclasses import dataclass
from itertools import islice


_MAX_BATCH_ITEMS = 64
_MAX_PAYLOAD_BYTES = 256 * 1024
_MAX_ATTEMPTS = 10
_MAX_TIMEOUT_SECONDS = 3_600
_MAX_DELAY_SECONDS = 60
_REQUEST_ID = re.compile(r"[A-Za-z0-9][A-Za-z0-9._-]{0,63}", re.ASCII)


class BatchProtocolError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class BatchRequest:
    request_id: str
    payload: bytes
    retry_safe: bool


@dataclass(frozen=True, slots=True)
class BatchResponse:
    request_id: str
    status: int
    body: bytes


@dataclass(frozen=True, slots=True)
class BatchOutcome:
    request: BatchRequest
    response: BatchResponse
    attempts: int
    retry_exhausted: bool


def _read_clock(
    clock: Callable[[], float],
    *,
    previous: float | None = None,
) -> float:
    value = clock()
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError("clock must return a number")
    current = float(value)
    if not math.isfinite(current):
        raise ValueError("clock must return a finite value")
    if previous is not None and current < previous:
        raise ValueError("clock moved backwards")
    return current


def _validate_request(request: object) -> BatchRequest:
    if not isinstance(request, BatchRequest):
        raise TypeError("requests must contain BatchRequest values")
    if _REQUEST_ID.fullmatch(request.request_id) is None:
        raise ValueError("request_id has an invalid format")
    if not isinstance(request.payload, bytes):
        raise TypeError("request payloads must be immutable bytes")
    if len(request.payload) > _MAX_PAYLOAD_BYTES:
        raise ValueError("a request payload exceeds the supported size")
    if not isinstance(request.retry_safe, bool):
        raise TypeError("retry_safe must be a boolean")
    return request


def _validate_response(response: object) -> BatchResponse:
    if not isinstance(response, BatchResponse):
        raise BatchProtocolError("responses must contain BatchResponse values")
    if _REQUEST_ID.fullmatch(response.request_id) is None:
        raise BatchProtocolError("a response has an invalid request_id")
    if isinstance(response.status, bool) or not isinstance(response.status, int):
        raise BatchProtocolError("response status must be an integer")
    if not 100 <= response.status <= 599:
        raise BatchProtocolError("response status is outside the supported range")
    if not isinstance(response.body, bytes):
        raise BatchProtocolError("response bodies must be immutable bytes")
    if len(response.body) > _MAX_PAYLOAD_BYTES:
        raise BatchProtocolError("a response body exceeds the supported size")
    return response


def retry_batch_items(
    requests: Sequence[BatchRequest],
    send_batch: Callable[
        [tuple[BatchRequest, ...], float],
        Iterable[BatchResponse],
    ],
    *,
    retryable_statuses: frozenset[int],
    max_attempts: int,
    timeout_seconds: float,
    initial_delay_seconds: float = 0.1,
    max_delay_seconds: float = 2.0,
    clock: Callable[[], float] = time.monotonic,
    sleeper: Callable[[float], None] = time.sleep,
) -> tuple[BatchOutcome, ...]:
    if not isinstance(requests, Sequence) or isinstance(requests, (str, bytes)):
        raise TypeError("requests must be a sequence")
    if not 1 <= len(requests) <= _MAX_BATCH_ITEMS:
        raise ValueError("request count is outside the supported range")
    if not callable(send_batch) or not callable(clock) or not callable(sleeper):
        raise TypeError("send_batch, clock, and sleeper must be callable")
    if isinstance(max_attempts, bool) or not isinstance(max_attempts, int):
        raise TypeError("max_attempts must be an integer")
    if not 1 <= max_attempts <= _MAX_ATTEMPTS:
        raise ValueError("max_attempts is outside the supported range")
    for name, value, upper in (
        ("timeout_seconds", timeout_seconds, _MAX_TIMEOUT_SECONDS),
        ("initial_delay_seconds", initial_delay_seconds, _MAX_DELAY_SECONDS),
        ("max_delay_seconds", max_delay_seconds, _MAX_DELAY_SECONDS),
    ):
        if isinstance(value, bool) or not isinstance(value, (int, float)):
            raise TypeError(f"{name} must be a number")
        if not math.isfinite(float(value)) or not 0 < value <= upper:
            raise ValueError(f"{name} is outside the supported range")
    if initial_delay_seconds > max_delay_seconds:
        raise ValueError("initial delay must not exceed maximum delay")
    if not isinstance(retryable_statuses, frozenset):
        raise TypeError("retryable_statuses must be a frozenset")
    if any(
        isinstance(status, bool)
        or not isinstance(status, int)
        or not 400 <= status <= 599
        for status in retryable_statuses
    ):
        raise ValueError("retryable_statuses contains an invalid status")

    ordered = tuple(_validate_request(request) for request in requests)
    by_id = {request.request_id: request for request in ordered}
    if len(by_id) != len(ordered):
        raise ValueError("request IDs must be unique")

    started = _read_clock(clock)
    deadline = started + float(timeout_seconds)
    if not math.isfinite(deadline) or deadline <= started:
        raise ValueError("timeout cannot form a representable future deadline")
    pending = dict(by_id)
    attempts = {request_id: 0 for request_id in by_id}
    last_response: dict[str, BatchResponse] = {}
    outcomes: dict[str, BatchOutcome] = {}
    delay = float(initial_delay_seconds)
    observed = started

    while pending:
        remaining = deadline - observed
        if remaining <= 0:
            for request_id, request in pending.items():
                outcomes[request_id] = BatchOutcome(
                    request,
                    last_response[request_id],
                    attempts[request_id],
                    retry_exhausted=True,
                )
            break
        batch = tuple(pending.values())
        for request in batch:
            attempts[request.request_id] += 1

        raw_responses = send_batch(batch, remaining)
        responses = tuple(islice(raw_responses, len(batch) + 1))
        validated = tuple(_validate_response(response) for response in responses)
        response_by_id = {response.request_id: response for response in validated}
        expected_ids = set(pending)
        if len(response_by_id) != len(validated):
            raise BatchProtocolError("response IDs must be unique")
        if set(response_by_id) != expected_ids:
            raise BatchProtocolError("response IDs do not match pending requests")

        retry: dict[str, BatchRequest] = {}
        for request_id, request in pending.items():
            response = response_by_id[request_id]
            last_response[request_id] = response
            eligible = request.retry_safe and response.status in retryable_statuses
            if eligible and attempts[request_id] < max_attempts:
                retry[request_id] = request
            else:
                outcomes[request_id] = BatchOutcome(
                    request,
                    response,
                    attempts[request_id],
                    retry_exhausted=(
                        eligible and attempts[request_id] >= max_attempts
                    ),
                )

        if not retry:
            break

        observed = _read_clock(clock, previous=observed)
        remaining = deadline - observed
        if remaining <= 0:
            for request_id, request in retry.items():
                outcomes[request_id] = BatchOutcome(
                    request,
                    last_response[request_id],
                    attempts[request_id],
                    retry_exhausted=True,
                )
            break

        sleeper(min(delay, remaining))
        observed = _read_clock(clock, previous=observed)
        if observed >= deadline:
            for request_id, request in retry.items():
                outcomes[request_id] = BatchOutcome(
                    request,
                    last_response[request_id],
                    attempts[request_id],
                    retry_exhausted=True,
                )
            break

        pending = retry
        delay = min(delay * 2, float(max_delay_seconds))

    return tuple(outcomes[request.request_id] for request in ordered)
```

## Example

```python
class ManualClock:
    def __init__(self) -> None:
        self.now = 10.0

    def read(self) -> float:
        return self.now

    def sleep(self, seconds: float) -> None:
        self.now += seconds


manual_clock = ManualClock()
sent_ids = []


def send(
    requests: tuple[BatchRequest, ...],
    remaining_seconds: float,
) -> tuple[BatchResponse, ...]:
    assert remaining_seconds > 0
    sent_ids.append(tuple(request.request_id for request in requests))
    if len(sent_ids) == 1:
        return (
            BatchResponse("gamma", 503, b"not repeated"),
            BatchResponse("alpha", 200, b"ready"),
            BatchResponse("beta", 503, b"retry later"),
        )
    return (BatchResponse("beta", 200, b"ready after retry"),)


report = retry_batch_items(
    (
        BatchRequest("alpha", b"first", True),
        BatchRequest("beta", b"second", True),
        BatchRequest("gamma", b"third", False),
    ),
    send,
    retryable_statuses=frozenset({429, 503}),
    max_attempts=3,
    timeout_seconds=5,
    clock=manual_clock.read,
    sleeper=manual_clock.sleep,
)

exhaustion_clock = ManualClock()
exhaustion_calls = []


def remain_unavailable(
    requests: tuple[BatchRequest, ...],
    remaining_seconds: float,
) -> tuple[BatchResponse, ...]:
    exhaustion_calls.append((requests[0].request_id, remaining_seconds))
    return (BatchResponse(requests[0].request_id, 503, b"retry later"),)


exhausted = retry_batch_items(
    (BatchRequest("delta", b"fourth", True),),
    remain_unavailable,
    retryable_statuses=frozenset({503}),
    max_attempts=2,
    timeout_seconds=5,
    clock=exhaustion_clock.read,
    sleeper=exhaustion_clock.sleep,
)[0]

assert (
    tuple((item.request.request_id, item.response.status) for item in report),
    tuple(item.attempts for item in report),
    tuple(item.retry_exhausted for item in report),
    tuple(sent_ids),
    (exhausted.attempts, exhausted.retry_exhausted, len(exhaustion_calls)),
) == (
    (("alpha", 200), ("beta", 200), ("gamma", 503)),
    (1, 2, 1),
    (False, False, False),
    (("alpha", "beta", "gamma"), ("beta",)),
    (2, True, 2),
)
```

## Trade-offs and Limitations

The helper eagerly retains at most 64 requests and outcomes, bounds attempts,
delays and payload sizes, and reads at most one response beyond the expected
count. A response with a missing, extra, or duplicate ID invalidates the
complete round because safe correlation is impossible. Backoff is
deterministic here; production clients often add bounded jitter and honor a
validated server retry hint.

`retry_exhausted` distinguishes a final retryable response from an ordinary
final response, but interpreting status codes and bodies remains protocol
policy. The attempt budget guarantees termination even if the injected clock
does not advance. Each transport call receives the positive time remaining,
but the helper cannot itself interrupt a sender or sleeper that ignores that
bound. Successful transport does not prove an operation is safe to repeat, so
`retry_safe` must follow an
explicit idempotency contract rather than an HTTP method guess.

## Related Snippets

<!-- catalog:related:start -->
- [Poll a Remote Operation Within Deadline and Failure Budgets](poll-a-remote-operation-within-deadline-and-failure-budgets.md)
- [Batch Items by Estimated Byte Size](../data-processing/batch-items-by-estimated-byte-size.md)
- [Collect Thread-Pool Results and Errors as Futures Complete](../concurrency-lifecycle/collect-thread-pool-results-and-errors-as-futures-complete.md)
<!-- catalog:related:end -->
