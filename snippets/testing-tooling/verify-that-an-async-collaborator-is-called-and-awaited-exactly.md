---
title: "Verify That an Async Collaborator Is Called and Awaited Exactly"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - verify-calls-to-a-standalone-function-with-create-autospec.md
  - wait-for-a-cross-thread-callback-with-threadingmock-instead-of-sleeping.md
  - ../concurrency-lifecycle/run-one-async-operation-with-a-bounded-resource-stack.md
---

# Verify That an Async Collaborator Is Called and Awaited Exactly

## Idea and Problem

Verify that a local unit both calls and awaits one asynchronous collaborator with one exact bounded interaction sequence.

Calling an `AsyncMock` records an ordinary mock call and returns a coroutine.
Awaiting that coroutine creates a separate await record. Checking only
`call_args_list` can therefore let a missing `await` pass unnoticed. Comparing
the complete ordered call and await histories against the same expected records
proves that every expected invocation reached both lifecycle stages.

## When to Use

Use this technique for one directly constructed `AsyncMock` that replaces a
trusted asynchronous collaborator in a small unit test. Keep 1 through 32
expected `call(...)` records, and assert the unit's meaningful returned state in
addition to its collaboration sequence.

Use a fake or integration test when the collaborator's state, scheduling,
cancellation, I/O, or protocol behavior matters. Use autospec separately when
signature enforcement is part of the test; this focused assertion deliberately
makes no signature claim.

## Implementation

```python
import asyncio
from collections.abc import Awaitable
from typing import Protocol
from unittest.mock import AsyncMock, call

_MAX_EXPECTED_ASYNC_CALLS = 32
_CALL_RECORD_TYPE = type(call())


def assert_called_and_awaited_exactly(
    collaborator: AsyncMock,
    expected_calls: tuple[object, ...],
) -> None:
    if not isinstance(collaborator, AsyncMock):
        raise TypeError("collaborator must be a direct AsyncMock")
    if type(expected_calls) is not tuple:
        raise TypeError("expected_calls must be an exact tuple")
    if not 1 <= len(expected_calls) <= _MAX_EXPECTED_ASYNC_CALLS:
        raise ValueError("expected call count is outside the supported range")
    for index, expected_call in enumerate(expected_calls):
        if type(expected_call) is not _CALL_RECORD_TYPE:
            raise TypeError(f"expected_calls[{index}] must be a call record")

    recorded_call_count = len(collaborator.call_args_list)
    recorded_await_count = len(collaborator.await_args_list)
    if (
        recorded_call_count > _MAX_EXPECTED_ASYNC_CALLS
        or recorded_await_count > _MAX_EXPECTED_ASYNC_CALLS
    ):
        raise AssertionError("the async interaction history exceeds the supported limit")

    recorded_calls = tuple(collaborator.call_args_list)
    recorded_awaits = tuple(collaborator.await_args_list)
    if recorded_calls != expected_calls:
        raise AssertionError("the complete async call history does not match")
    if recorded_awaits != expected_calls:
        raise AssertionError("the complete async await history does not match")
```

## Example

```python
class LabelFetcher(Protocol):
    def __call__(self, label: str, *, fresh: bool) -> Awaitable[str]: ...


async def fetch_labels(
    labels: tuple[str, ...],
    fetch: LabelFetcher,
) -> tuple[str, ...]:
    results: list[str] = []
    for label in labels:
        results.append(await fetch(label, fresh=False))
    return tuple(results)


async def run_example() -> tuple[tuple[str, ...], bool]:
    fetch = AsyncMock(side_effect=("ALPHA", "BETA"))
    expected = (
        call("alpha", fresh=False),
        call("beta", fresh=False),
    )

    results = await fetch_labels(("alpha", "beta"), fetch)
    assert_called_and_awaited_exactly(fetch, expected)

    orphaned_fetch = AsyncMock(return_value="unused")
    orphan = orphaned_fetch("gamma", fresh=False)
    try:
        assert_called_and_awaited_exactly(
            orphaned_fetch,
            (call("gamma", fresh=False),),
        )
    except AssertionError:
        missing_await_detected = True
    else:
        missing_await_detected = False
    finally:
        orphan.close()

    return results, missing_await_detected


assert asyncio.run(run_example()) == (("ALPHA", "BETA"), True)
```

## Trade-offs and Limitations

The assertion performs `O(C)` comparison work and retains `O(C)` temporary
references for at most 32 expected calls. Argument equality must be trusted and
finite. The mock has already retained both histories before the assertion runs;
this helper only refuses to copy an unexpectedly larger history.

An `AsyncMock` records a call before the returned coroutine is awaited, while
its side effect and return behavior occur during the await. Matching both lists
proves invocation and awaiting in the same declared order, but it does not
prove when the coroutine ran, whether another task awaited it, or whether a
real collaborator would complete successfully. It also does not validate a
callable signature, annotations, argument value types, or external effects.

The negative example closes its deliberately orphaned coroutine so the test
does not leak it or emit an unawaited-coroutine warning. Production code should
await or otherwise explicitly own every created coroutine rather than closing
one merely to satisfy a test. Its unit-facing parameter uses a narrow async
protocol instead of coupling production code to the mock type. Exact interaction
assertions are brittle when order is an implementation detail, so use this
technique only when the complete sequence is part of the unit's contract.

## Related Snippets

<!-- catalog:related:start -->
- [Verify Calls to a Standalone Function with Autospec](verify-calls-to-a-standalone-function-with-create-autospec.md)
- [Wait for a Cross-Thread Callback with ThreadingMock Instead of Sleeping](wait-for-a-cross-thread-callback-with-threadingmock-instead-of-sleeping.md)
- [Run One Async Operation with a Bounded Resource Stack](../concurrency-lifecycle/run-one-async-operation-with-a-bounded-resource-stack.md)
<!-- catalog:related:end -->
