---
title: "Wait for Every Named Connector Under One Shared Deadline"
snippet_type: testing-technique
use_cases:
  - concurrency-control
  - resource-management
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../concurrency-lifecycle/collect-a-bounded-thread-pool-batch-under-one-deadline.md
  - wait-for-named-queue-conditions-under-one-monotonic-deadline.md
  - ../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md
---

# Wait for Every Named Connector Under One Shared Deadline

## Idea and Problem

Wait concurrently for one predicate match from every owned connector while all participants share the same absolute monotonic deadline.

The group gives each connector one worker and the same deadline, then returns
frozen matches in connector order only when every wait succeeds. Otherwise it
raises a frozen report that separates completed connector failures from names
still unfinished at the deadline. Unlike a generic thread batch, this helper
also owns connector cleanup for the surrounding context.

## When to Use

Use this testing technique when 1-32 independent adapters expose a blocking
predicate wait and a test needs every participant to reach the same observable
state. Each adapter must accept an absolute monotonic deadline, return one
matching value or raise, and make `close()` safe to call while its wait is
finishing.

Use an async task group when the adapters are asynchronous. Use a process
boundary when work must be forcibly terminated, and use a production
coordination protocol when the observed state itself needs consensus or
durability.

## Implementation

```python
import math
import re
import time
from collections.abc import Callable
from concurrent.futures import Future, ThreadPoolExecutor, wait
from dataclasses import dataclass
from threading import Lock
from types import TracebackType
from typing import Generic, NoReturn, Protocol, TypeVar


MessageT = TypeVar("MessageT")
_CONNECTOR_NAME = re.compile(r"[a-z][a-z0-9_-]{0,31}", re.ASCII)
_MAX_CONNECTORS = 32
_MAX_TIMEOUT_SECONDS = 86_400.0


class PredicateConnector(Protocol[MessageT]):
    def wait_for(
        self,
        predicate: Callable[[MessageT], bool],
        *,
        deadline: float,
    ) -> MessageT: ...

    def close(self) -> None: ...


@dataclass(frozen=True, slots=True)
class NamedConnector(Generic[MessageT]):
    name: str
    adapter: PredicateConnector[MessageT]


@dataclass(frozen=True, slots=True)
class ConnectorMatch(Generic[MessageT]):
    name: str
    value: MessageT


@dataclass(frozen=True, slots=True)
class ConnectorFailure:
    name: str
    error: Exception


@dataclass(frozen=True, slots=True)
class ConnectorCloseFailure:
    name: str
    error: BaseException


@dataclass(frozen=True, slots=True)
class ConnectorWaitReport:
    failures: tuple[ConnectorFailure, ...]
    unfinished_names: tuple[str, ...]


class ConnectorWaitError(RuntimeError):
    def __init__(self, report: ConnectorWaitReport) -> None:
        self.report = report
        super().__init__(
            "connector wait failed: "
            f"{len(report.failures)} failed, "
            f"{len(report.unfinished_names)} unfinished"
        )


class ConnectorCleanupError(RuntimeError):
    def __init__(self, failures: tuple[ConnectorCloseFailure, ...]) -> None:
        self.failures = failures
        super().__init__(f"{len(failures)} connector closes failed")


def _raise_cleanup_failures(
    failures: tuple[ConnectorCloseFailure, ...],
    body_error: BaseException | None,
) -> NoReturn:
    cleanup_error = ConnectorCleanupError(failures)
    process_control_errors = tuple(
        failure.error
        for failure in failures
        if not isinstance(failure.error, Exception)
    )
    if body_error is None and not process_control_errors:
        raise cleanup_error

    grouped: tuple[BaseException, ...] = (
        (() if body_error is None else (body_error,))
        + (cleanup_error,)
        + process_control_errors
    )
    message = (
        "connector cleanup failed"
        if body_error is None
        else "connector body and cleanup failed"
    )
    if all(isinstance(error, Exception) for error in grouped):
        raise ExceptionGroup(message, grouped) from None
    raise BaseExceptionGroup(message, grouped) from None


def _shared_deadline(timeout: int | float) -> float:
    if isinstance(timeout, bool) or not isinstance(timeout, (int, float)):
        raise TypeError("timeout must be numeric")
    try:
        seconds = float(timeout)
    except OverflowError:
        raise ValueError("timeout must be representable as a float") from None
    if not math.isfinite(seconds) or not 0 < seconds <= _MAX_TIMEOUT_SECONDS:
        raise ValueError("timeout must be finite and between zero and one day")
    deadline = time.monotonic() + seconds
    if not math.isfinite(deadline):
        raise ValueError("timeout produces an invalid deadline")
    return deadline


class ConnectorWaitGroup(Generic[MessageT]):
    def __init__(self, connectors: tuple[NamedConnector[MessageT], ...]) -> None:
        if type(connectors) is not tuple:
            raise TypeError("connectors must be an exact tuple")
        if not 1 <= len(connectors) <= _MAX_CONNECTORS:
            raise ValueError("connector count is outside the supported range")

        names: set[str] = set()
        adapter_ids: set[int] = set()
        for connector in connectors:
            if type(connector) is not NamedConnector:
                raise TypeError("connectors must contain exact NamedConnector values")
            if (
                type(connector.name) is not str
                or _CONNECTOR_NAME.fullmatch(connector.name) is None
            ):
                raise ValueError("a connector name is invalid")
            if connector.name in names:
                raise ValueError("connector names must be unique")
            if id(connector.adapter) in adapter_ids:
                raise ValueError("connector adapters must be distinct objects")
            if not callable(getattr(connector.adapter, "wait_for", None)):
                raise TypeError("every connector must provide wait_for")
            if not callable(getattr(connector.adapter, "close", None)):
                raise TypeError("every connector must provide close")
            names.add(connector.name)
            adapter_ids.add(id(connector.adapter))

        self._connectors = connectors
        self._state_lock = Lock()
        self._entered = False
        self._wait_started = False
        self._waiting = False
        self._closed = False

    def __enter__(self) -> "ConnectorWaitGroup[MessageT]":
        with self._state_lock:
            if self._closed:
                raise RuntimeError("connector group is closed")
            if self._entered:
                raise RuntimeError("connector group is already entered")
            self._entered = True
        return self

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc: BaseException | None,
        traceback: TracebackType | None,
    ) -> None:
        del exc_type, traceback
        with self._state_lock:
            if self._closed:
                return
            self._closed = True
            self._entered = False

        failures: list[ConnectorCloseFailure] = []
        for connector in self._connectors:
            try:
                connector.adapter.close()
            except BaseException as error:
                failures.append(ConnectorCloseFailure(connector.name, error))

        if failures:
            _raise_cleanup_failures(tuple(failures), exc)

    def wait_for_all(
        self,
        predicate: Callable[[MessageT], bool],
        *,
        timeout: int | float,
    ) -> tuple[ConnectorMatch[MessageT], ...]:
        if not callable(predicate):
            raise TypeError("predicate must be callable")
        deadline = _shared_deadline(timeout)

        with self._state_lock:
            if self._closed:
                raise RuntimeError("connector group is closed")
            if not self._entered:
                raise RuntimeError("connector group must be entered")
            if self._waiting:
                raise RuntimeError("a connector wait is already in progress")
            if self._wait_started:
                raise RuntimeError("connector group wait is one-shot")
            self._waiting = True
            self._wait_started = True

        executor: ThreadPoolExecutor | None = None
        futures: list[Future[MessageT]] = []
        try:
            executor = ThreadPoolExecutor(
                max_workers=len(self._connectors),
                thread_name_prefix="connector-wait",
            )

            def wait_one(connector: NamedConnector[MessageT]) -> MessageT:
                value = connector.adapter.wait_for(predicate, deadline=deadline)
                if time.monotonic() >= deadline:
                    raise TimeoutError("connector returned after the shared deadline")
                return value

            futures.extend(
                executor.submit(wait_one, connector)
                for connector in self._connectors
            )
            remaining = max(0.0, deadline - time.monotonic())
            _, unfinished = wait(futures, timeout=remaining)
            unfinished_set = set(unfinished)
            for future in unfinished:
                future.cancel()

            matches: list[ConnectorMatch[MessageT]] = []
            failures: list[ConnectorFailure] = []
            unfinished_names: list[str] = []
            for connector, future in zip(
                self._connectors,
                futures,
                strict=True,
            ):
                if future in unfinished_set:
                    unfinished_names.append(connector.name)
                    continue
                try:
                    value = future.result()
                except Exception as error:
                    failures.append(ConnectorFailure(connector.name, error))
                else:
                    matches.append(ConnectorMatch(connector.name, value))

            if failures or unfinished_names:
                raise ConnectorWaitError(
                    ConnectorWaitReport(tuple(failures), tuple(unfinished_names))
                )
            return tuple(matches)
        finally:
            if executor is not None:
                executor.shutdown(wait=False, cancel_futures=True)
            with self._state_lock:
                self._waiting = False
```

## Example

```python
class FakeConnector:
    def __init__(
        self,
        values: tuple[int, ...],
        *,
        wait_error: Exception | None = None,
        close_error: Exception | None = None,
    ) -> None:
        self.values = values
        self.wait_error = wait_error
        self.close_error = close_error
        self.close_calls = 0

    def wait_for(
        self,
        predicate: Callable[[int], bool],
        *,
        deadline: float,
    ) -> int:
        if time.monotonic() >= deadline:
            raise TimeoutError("deadline reached")
        if self.wait_error is not None:
            raise self.wait_error
        for value in self.values:
            if predicate(value):
                return value
        raise LookupError("no matching value")

    def close(self) -> None:
        self.close_calls += 1
        if self.close_error is not None:
            raise self.close_error


alpha = FakeConnector((1, 4))
beta = FakeConnector((2, 5))
group = ConnectorWaitGroup(
    (
        NamedConnector("alpha", alpha),
        NamedConnector("beta", beta),
    )
)
with group:
    matches = group.wait_for_all(lambda value: value % 2 == 0, timeout=1)
    try:
        group.wait_for_all(lambda value: True, timeout=1)
    except RuntimeError:
        second_wait_rejected = True
    else:
        second_wait_rejected = False

try:
    group.wait_for_all(lambda value: True, timeout=1)
except RuntimeError:
    closed_use_rejected = True
else:
    closed_use_rejected = False

assert (
    matches,
    alpha.close_calls,
    beta.close_calls,
    second_wait_rejected,
    closed_use_rejected,
) == (
    (ConnectorMatch("alpha", 4), ConnectorMatch("beta", 2)),
    1,
    1,
    True,
    True,
)

failing = FakeConnector((), wait_error=LookupError("unavailable"))
healthy = FakeConnector((6,))
with ConnectorWaitGroup(
    (
        NamedConnector("failing", failing),
        NamedConnector("healthy", healthy),
    )
) as failing_group:
    try:
        failing_group.wait_for_all(lambda value: value > 0, timeout=1)
    except ConnectorWaitError as error:
        report = error.report

assert (
    tuple(item.name for item in report.failures),
    report.unfinished_names,
    failing.close_calls,
    healthy.close_calls,
) == (("failing",), (), 1, 1)

bad_close = FakeConnector((8,), close_error=OSError("close failed"))
good_close = FakeConnector((10,))
try:
    with ConnectorWaitGroup(
        (
            NamedConnector("bad_close", bad_close),
            NamedConnector("good_close", good_close),
        )
    ) as closing_group:
        closing_group.wait_for_all(lambda value: value > 0, timeout=1)
except ConnectorCleanupError as error:
    cleanup_failure_names = tuple(item.name for item in error.failures)

assert (
    cleanup_failure_names,
    bad_close.close_calls,
    good_close.close_calls,
) == (("bad_close",), 1, 1)
```

## Trade-offs and Limitations

The deadline bounds how long `wait_for_all()` collects results; it does not
forcibly stop Python threads, bound interpreter shutdown, or prove that an
adapter stopped at the deadline. Every adapter must honor the supplied
absolute deadline and make `close()` unblock or finish any remaining wait.
`shutdown(wait=False)` deliberately avoids an executor context manager's
unbounded exit wait, but a non-cooperative adapter can still leave a thread
running after the report is raised.

The group permits exactly one wait. This prevents a late worker from one timed
out attempt overlapping another attempt on the same connector. The state lock
rejects concurrent waits, but context exit itself must not race with a caller
that is still invoking the group.

Connector and cleanup failures retain their exception objects, and returned
matches retain their values, so the frozen wrappers provide only shallow
immutability. Cleanup attempts every connector even after ordinary exceptions;
it also finishes the remaining close attempts after a process-control
exception. Ordinary cleanup failures use `ConnectorCleanupError`. A body
failure or process-control cleanup failure is preserved in an appropriately
typed `ExceptionGroup` or `BaseExceptionGroup`, with every named close failure
also available through the cleanup error.

The implementation eagerly creates one thread per connector and is therefore
limited to 32 distinct adapter objects. It provides no retries, predicate
composition, event history, consensus, live networking, or forced
cancellation.

## Related Snippets

<!-- catalog:related:start -->
- [Collect a Bounded Thread-Pool Batch Under One Deadline](../concurrency-lifecycle/collect-a-bounded-thread-pool-batch-under-one-deadline.md)
- [Wait for Named Queue Conditions Under One Monotonic Deadline](wait-for-named-queue-conditions-under-one-monotonic-deadline.md)
- [Wait for a Predicate Until a Monotonic Deadline](../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md)
<!-- catalog:related:end -->
