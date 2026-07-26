---
title: "Compensate Completed Workflow Steps in Reverse Order"
snippet_type: pattern
use_cases:
  - retry-recovery
  - automation
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/keep-exception-handlers-narrow-with-try-else.md
  - ../storage-databases/replace-a-file-atomically-with-a-sibling-temporary-file.md
---

# Compensate Completed Workflow Steps in Reverse Order

## Idea and Problem

Register one compensation after each successful synchronous step so a later failure can unwind completed work without hiding the primary error.

An `ExitStack` owns the last-in, first-out callback order. Each callback is
wrapped so one compensation failure is recorded instead of preventing later
cleanup attempts, while the exception that interrupted forward execution
remains the explicit cause of the final workflow error.

## When to Use

Use this pattern for a short, process-local sequence whose completed effects
have well-defined inverse operations. Step names and callables are validated
before the first effect, and a step must return its compensation only after it
has completed successfully. The step that fails before returning is responsible
for cleaning up any partial effect it created.

Prefer a database transaction when one transactional store owns every effect.
Use ordinary context managers directly for resource acquisition, and use a
durable workflow or saga engine when compensation must survive a process crash,
restart, or distributed timeout.

## Implementation

```python
import inspect
from collections.abc import Callable, Iterable
from contextlib import ExitStack
from dataclasses import dataclass

Compensation = Callable[[], None]
StepOperation = Callable[[], Compensation | None]


@dataclass(frozen=True, slots=True)
class WorkflowStep:
    name: str
    execute: StepOperation


@dataclass(frozen=True, slots=True)
class CompensationFailure:
    step_name: str
    error: Exception


class WorkflowFailure(RuntimeError):
    def __init__(
        self,
        failed_step: str,
        compensation_failures: tuple[CompensationFailure, ...],
    ) -> None:
        self.failed_step = failed_step
        self.compensation_failures = compensation_failures
        count = len(compensation_failures)
        super().__init__(
            f"workflow failed at {failed_step!r} "
            f"with {count} compensation failure(s)"
        )


def _is_async_callable(callback: Callable[..., object]) -> bool:
    call_method = type(callback).__call__
    return (
        inspect.iscoroutinefunction(callback)
        or inspect.isasyncgenfunction(callback)
        or inspect.iscoroutinefunction(call_method)
        or inspect.isasyncgenfunction(call_method)
    )


def _validated_steps(steps: Iterable[WorkflowStep]) -> tuple[WorkflowStep, ...]:
    prepared = tuple(steps)
    names: set[str] = set()

    for step in prepared:
        if not isinstance(step, WorkflowStep):
            raise TypeError("every step must be a WorkflowStep")
        if not isinstance(step.name, str) or not step.name.strip():
            raise ValueError("step names must be non-empty text")
        if step.name in names:
            raise ValueError(f"duplicate step name: {step.name!r}")
        if not callable(step.execute) or _is_async_callable(step.execute):
            raise TypeError(f"step {step.name!r} must use a synchronous callable")
        names.add(step.name)

    return prepared


def _attempt_compensation(
    step_name: str,
    compensation: Compensation,
    failures: list[CompensationFailure],
) -> None:
    try:
        compensation()
    except Exception as error:
        failures.append(CompensationFailure(step_name, error))


def run_compensated_workflow(steps: Iterable[WorkflowStep]) -> None:
    prepared = _validated_steps(steps)
    compensation_failures: list[CompensationFailure] = []
    failed_step = ""

    try:
        with ExitStack() as compensations:
            for step in prepared:
                failed_step = step.name
                compensation = step.execute()
                if compensation is None:
                    continue
                if not callable(compensation) or _is_async_callable(compensation):
                    raise TypeError(
                        f"step {step.name!r} must return a synchronous compensation or None"
                    )
                compensations.callback(
                    _attempt_compensation,
                    step.name,
                    compensation,
                    compensation_failures,
                )

            compensations.pop_all()
    except Exception as primary_error:
        raise WorkflowFailure(
            failed_step,
            tuple(compensation_failures),
        ) from primary_error
```

## Example

```python
events: list[str] = []


def reserve_capacity() -> Compensation:
    events.append("reserve")

    def release_capacity() -> None:
        events.append("release")

    return release_capacity


def publish_summary() -> Compensation:
    events.append("publish")

    def withdraw_summary() -> None:
        events.append("withdraw")
        raise OSError("withdrawal was rejected")

    return withdraw_summary


def send_notice() -> None:
    events.append("notify")
    raise ValueError("recipient was unavailable")


steps = (
    WorkflowStep("reserve", reserve_capacity),
    WorkflowStep("publish", publish_summary),
    WorkflowStep("notify", send_notice),
)

try:
    run_compensated_workflow(steps)
except WorkflowFailure as error:
    failed_step = error.failed_step
    primary_type = type(error.__cause__).__name__
    rollback_errors = tuple(
        (failure.step_name, type(failure.error).__name__)
        for failure in error.compensation_failures
    )
else:
    failed_step = ""
    primary_type = ""
    rollback_errors = ()

assert (events, failed_step, primary_type, rollback_errors) == (
    ["reserve", "publish", "notify", "withdraw", "release"],
    "notify",
    "ValueError",
    (("publish", "OSError"),),
)
```

## Trade-offs and Limitations

Compensation is not atomic rollback. Another observer may see intermediate
effects, an inverse operation may be incomplete, and callbacks should be
idempotent if an outer caller might invoke the workflow again. The helper
catches ordinary `Exception` instances from compensation callbacks but does
not convert cancellation-like `BaseException` subclasses into business
failures. It neither retries nor persists progress, and it deliberately cannot
compensate the currently failing step because that step never registered a
completed inverse operation.

Materializing the input requires a finite iterable. Validation can reject
declared async callables, but a misleading synchronous wrapper can still return
an awaitable at runtime; use an async-specific design instead of adapting it
here. Structured failures retain their exception objects for local diagnosis,
so callers should extract safe fields before serialization or logging.

## Related Snippets

<!-- catalog:related:start -->
- [Keep Exception Handlers Narrow with try/else](../python-language/keep-exception-handlers-narrow-with-try-else.md)
- [Replace a File Atomically with a Sibling Temporary File](../storage-databases/replace-a-file-atomically-with-a-sibling-temporary-file.md)
<!-- catalog:related:end -->
