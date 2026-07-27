---
title: "Run One Async Operation with a Bounded Resource Stack"
snippet_type: pattern
use_cases:
  - concurrency-control
  - lifecycle-management
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - guard-an-async-resource-with-explicit-lifecycle-states.md
  - elect-one-final-releaser-from-bounded-named-leases.md
  - ../python-language/add-bounded-stage-context-without-replacing-an-exception.md
---

# Run One Async Operation with a Bounded Resource Stack

## Idea and Problem

Enter a small declared set of asynchronous resources for exactly one operation, then release every entered resource in reverse order.

Validating the complete declaration before invoking any factory prevents an
early side effect from preceding a later configuration error. An
`AsyncExitStack` handles success, failure, cancellation, and partial entry,
while a separate failure sentinel prevents a truthy `__aexit__` result from
turning an entry or operation exception into apparent success.

## When to Use

Use this pattern when one asynchronous operation temporarily owns between one
and eight independently constructed resources. Resource factories and context
managers must be trusted local callables: validation checks their declarations,
not what arbitrary callback code will do. Each factory must return an async
context manager, and the operation must accept the frozen resource records in
declaration order.

Prefer a purpose-built aggregate context manager when resources have coupled
startup rules or need to outlive one operation. Use a task group separately if
the operation itself needs concurrency; this function creates no tasks and
keeps no state between invocations.

## Implementation

```python
import re
from collections.abc import Awaitable, Callable
from contextlib import AbstractAsyncContextManager, AsyncExitStack
from dataclasses import dataclass
from typing import TypeVar


ResultT = TypeVar("ResultT")


@dataclass(frozen=True, slots=True)
class ResourceSpec:
    name: str
    factory: Callable[[], AbstractAsyncContextManager[object]]


@dataclass(frozen=True, slots=True)
class EnteredResource:
    name: str
    value: object


def _validate_declarations(
    resource_specs: tuple[ResourceSpec, ...],
    operation: object,
) -> None:
    if type(resource_specs) is not tuple:
        raise TypeError("resource_specs must be an exact tuple")
    if not 1 <= len(resource_specs) <= 8:
        raise ValueError("resource_specs must contain between 1 and 8 items")
    if not callable(operation):
        raise TypeError("operation must be callable")

    names: set[str] = set()
    for spec in resource_specs:
        if type(spec) is not ResourceSpec:
            raise TypeError("every resource declaration must be a ResourceSpec")
        if type(spec.name) is not str:
            raise TypeError("resource names must be exact strings")
        if (
            not 1 <= len(spec.name) <= 32
            or re.fullmatch(r"[a-z][a-z0-9_-]{0,31}", spec.name) is None
        ):
            raise ValueError(
                "resource names must be conservative ASCII identifiers"
            )
        if spec.name in names:
            raise ValueError("resource names must be unique")
        if not callable(spec.factory):
            raise TypeError("resource factories must be callable")
        names.add(spec.name)


async def run_one_with_resources(
    resource_specs: tuple[ResourceSpec, ...],
    operation: Callable[
        [tuple[EnteredResource, ...]],
        Awaitable[ResultT],
    ],
) -> ResultT:
    _validate_declarations(resource_specs, operation)

    original_error: BaseException | None = None
    result: ResultT

    async with AsyncExitStack() as stack:
        try:
            entered: list[EnteredResource] = []
            for spec in resource_specs:
                manager = spec.factory()
                value = await stack.enter_async_context(manager)
                entered.append(EnteredResource(spec.name, value))

            result = await operation(tuple(entered))
        except BaseException as error:
            original_error = error
            raise

    if original_error is not None:
        raise original_error
    return result
```

## Example

```python
import asyncio


class ProbeContext:
    def __init__(self, events: list[str], name: str, *, suppress: bool) -> None:
        self.events = events
        self.name = name
        self.suppress = suppress

    async def __aenter__(self) -> str:
        self.events.append(f"enter:{self.name}")
        return f"value:{self.name}"

    async def __aexit__(self, exc_type, exc, traceback) -> bool:
        self.events.append(f"exit:{self.name}")
        return self.suppress


def probe_spec(
    events: list[str],
    name: str,
    *,
    suppress: bool = False,
) -> ResourceSpec:
    return ResourceSpec(
        name,
        lambda: ProbeContext(events, name, suppress=suppress),
    )


async def run_example():
    success_events: list[str] = []
    result_token = object()

    async def use(resources: tuple[EnteredResource, ...]) -> object:
        success_events.append(
            "action:"
            + ",".join(
                f"{resource.name}={resource.value}"
                for resource in resources
            )
        )
        return result_token

    returned = await run_one_with_resources(
        (
            probe_spec(success_events, "first"),
            probe_spec(success_events, "second"),
        ),
        use,
    )

    cancellation_events: list[str] = []
    operation_started = asyncio.Event()
    operation_cancellations: list[asyncio.CancelledError] = []

    async def wait_for_cancel(_resources: tuple[EnteredResource, ...]) -> None:
        cancellation_events.append("action")
        operation_started.set()
        try:
            await asyncio.Future()
        except asyncio.CancelledError as error:
            operation_cancellations.append(error)
            raise

    task = asyncio.create_task(
        run_one_with_resources(
            (probe_spec(cancellation_events, "shield", suppress=True),),
            wait_for_cancel,
        )
    )
    await operation_started.wait()
    task.cancel()
    try:
        await task
    except asyncio.CancelledError as caught:
        cancellation_raised = True
        identity_preserved = caught is operation_cancellations[0]
    else:
        cancellation_raised = False
        identity_preserved = False

    return (
        tuple(success_events),
        returned is result_token,
        tuple(cancellation_events),
        cancellation_raised,
        identity_preserved,
        task.cancelled(),
    )


assert asyncio.run(run_example()) == (
    (
        "enter:first",
        "enter:second",
        "action:first=value:first,second=value:second",
        "exit:second",
        "exit:first",
    ),
    True,
    (
        "enter:shield",
        "action",
        "exit:shield",
    ),
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Factories and entries run sequentially, so their latencies add. A context
manager whose own `__aenter__` fails is responsible for releasing anything it
allocated before raising; `AsyncExitStack` can close only managers that entered
successfully. Cleanup is awaited in reverse order, but a cleanup method that
ignores cancellation or never returns can still prevent completion.

A non-suppressing cleanup error follows normal context-manager rules and may
replace an active entry or operation error, leaving the earlier error as
context. If cleanup instead suppresses an entry, factory, operation, or
cancellation failure, the function re-raises the same exception object after
the stack is closed. Its type and identity are preserved, and a real task
cancellation still leaves the task cancelled instead of producing a result.

The returned object is unchanged, so the operation must not return a handle
that becomes unusable when the stack closes. The pattern provides no timeout,
retry, logging, persistence, transport behavior, or cross-invocation sharing.

## Related Snippets

<!-- catalog:related:start -->
- [Guard an Async Resource with Explicit Lifecycle States](guard-an-async-resource-with-explicit-lifecycle-states.md)
- [Elect One Final Releaser from Bounded Named Leases](elect-one-final-releaser-from-bounded-named-leases.md)
- [Add Bounded Stage Context Without Replacing an Exception](../python-language/add-bounded-stage-context-without-replacing-an-exception.md)
<!-- catalog:related:end -->
