---
title: "Transfer Bounded Synchronous Cleanup Ownership with ExitStack Pop All"
snippet_type: pattern
use_cases:
  - lifecycle-management
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - run-one-async-operation-with-a-bounded-resource-stack.md
  - guard-an-async-resource-with-explicit-lifecycle-states.md
  - ../python-language/add-bounded-stage-context-without-replacing-an-exception.md
---

# Transfer Bounded Synchronous Cleanup Ownership with ExitStack Pop All

## Idea and Problem

Acquire a small declared sequence of synchronous context managers, then transfer their registered exit callbacks to the caller without running cleanup.

An `ExitStack` safely unwinds already entered managers when a later factory or
entry fails. On complete success, `pop_all()` moves the callbacks into a new
stack and empties the local one. The returned resources and caller-owned stack
therefore cross the function boundary together.

A saved failure sentinel handles an unusual edge case: an earlier manager may
return a truthy value from `__exit__`. Such cleanup may suppress an exception
inside the local `with`, but it must not turn incomplete acquisition into an
apparently successful result.

## When to Use

Use this pattern when one synchronous owner must acquire between one and 32
independent resources now and decide their lifetime later. It fits factories
known to return context managers when declaration-wide validation must happen
before the first side effect and partial acquisition must unwind in LIFO order.

Prefer an aggregate context manager when the resources have coupled lifecycle
rules. Use `AsyncExitStack` for asynchronous managers. Transferring callbacks
does not make resources safe to use or close from another thread.

## Implementation

```python
import re
from collections.abc import Callable
from contextlib import AbstractContextManager, ExitStack
from dataclasses import dataclass

_MAX_RESOURCE_SPECS = 32
_RESOURCE_NAME = re.compile(
    r"[a-z][a-z0-9_-]{0,31}",
    re.ASCII,
).fullmatch


@dataclass(frozen=True, slots=True)
class ResourceSpec:
    name: str
    factory: Callable[[], AbstractContextManager[object]]


@dataclass(frozen=True, slots=True)
class NamedResource:
    name: str
    value: object


@dataclass(frozen=True, slots=True)
class AcquiredResources:
    resources: tuple[NamedResource, ...]
    cleanup: ExitStack


def _validate_resource_specs(specs: tuple[ResourceSpec, ...]) -> None:
    if type(specs) is not tuple:
        raise TypeError("specs must be an exact tuple")
    if not 1 <= len(specs) <= _MAX_RESOURCE_SPECS:
        raise ValueError("specs count is outside the supported range")

    names: set[str] = set()
    for spec in specs:
        if type(spec) is not ResourceSpec:
            raise TypeError("every declaration must be an exact ResourceSpec")
        if type(spec.name) is not str:
            raise TypeError("resource names must be exact strings")
        if _RESOURCE_NAME(spec.name) is None:
            raise ValueError("a resource name is outside the supported grammar")
        if spec.name in names:
            raise ValueError("resource names must be unique")
        if not callable(spec.factory):
            raise TypeError("every resource factory must be callable")
        names.add(spec.name)


def acquire_resources(
    specs: tuple[ResourceSpec, ...],
) -> AcquiredResources:
    _validate_resource_specs(specs)

    initiating_error: BaseException | None = None
    entered: list[NamedResource] = []
    transferred: ExitStack

    with ExitStack() as pending:
        try:
            for spec in specs:
                manager = spec.factory()
                value = pending.enter_context(manager)
                entered.append(NamedResource(spec.name, value))
        except BaseException as error:
            initiating_error = error
            raise
        transferred = pending.pop_all()

    if initiating_error is not None:
        raise initiating_error
    return AcquiredResources(tuple(entered), transferred)
```

## Example

```python
class ProbeContext:
    def __init__(
        self,
        events: list[str],
        name: str,
        *,
        entry_error: BaseException | None = None,
        suppress: bool = False,
    ) -> None:
        self.events = events
        self.name = name
        self.entry_error = entry_error
        self.suppress = suppress

    def __enter__(self) -> str:
        self.events.append(f"enter:{self.name}")
        if self.entry_error is not None:
            raise self.entry_error
        return f"value:{self.name}"

    def __exit__(self, exc_type, exc, traceback) -> bool:
        error_name = "none" if exc_type is None else exc_type.__name__
        self.events.append(f"exit:{self.name}:{error_name}")
        return self.suppress


def probe_spec(
    events: list[str],
    name: str,
    *,
    entry_error: BaseException | None = None,
    suppress: bool = False,
) -> ResourceSpec:
    return ResourceSpec(
        name,
        lambda: ProbeContext(
            events,
            name,
            entry_error=entry_error,
            suppress=suppress,
        ),
    )


success_events: list[str] = []
acquired = acquire_resources(
    (
        probe_spec(success_events, "first"),
        probe_spec(success_events, "second"),
    )
)
assert success_events == ["enter:first", "enter:second"]
assert acquired.resources == (
    NamedResource("first", "value:first"),
    NamedResource("second", "value:second"),
)

acquired.cleanup.close()
acquired.cleanup.close()
assert success_events == [
    "enter:first",
    "enter:second",
    "exit:second:none",
    "exit:first:none",
]

failure_events: list[str] = []
entry_error = RuntimeError("second entry failed")
try:
    acquire_resources(
        (
            probe_spec(failure_events, "first", suppress=True),
            probe_spec(failure_events, "second", entry_error=entry_error),
        )
    )
except RuntimeError as caught:
    assert caught is entry_error
else:
    raise AssertionError("a truthy cleanup hid an acquisition failure")

assert failure_events == [
    "enter:first",
    "enter:second",
    "exit:first:RuntimeError",
]
```

## Trade-offs and Limitations

Validation and acquisition take `O(n)` work and state for at most 32 declared
resources. Entry is sequential. Partial failure invokes registered exits once
in reverse entry order. A manager whose own `__enter__` fails is responsible
for any state it created before it was successfully registered.

The frozen result record prevents rebinding its fields, but the returned
`ExitStack` is intentionally stateful. The caller must close it or use it as a
context manager. Cleanup runs at most once because exit callbacks are consumed;
calling `close()` again has no callbacks left to invoke.

Only exit callbacks are transferred. Resource objects, thread affinity, and
other ownership rules are not changed or verified. Cleanup callbacks remain
arbitrary code: they may suppress a caller-body exception, raise a replacement
exception, or fail partway through unwinding according to normal `ExitStack`
semantics. This pattern does not provide atomic, parallel, asynchronous,
cross-thread, or serializable acquisition.

## Related Snippets

<!-- catalog:related:start -->
- [Run One Async Operation with a Bounded Resource Stack](run-one-async-operation-with-a-bounded-resource-stack.md)
- [Guard an Async Resource with Explicit Lifecycle States](guard-an-async-resource-with-explicit-lifecycle-states.md)
- [Add Bounded Stage Context Without Replacing an Exception](../python-language/add-bounded-stage-context-without-replacing-an-exception.md)
<!-- catalog:related:end -->
