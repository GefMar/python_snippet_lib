---
title: "Capture Mutable Mock Arguments at Call Time"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - verify-calls-to-a-standalone-function-with-create-autospec.md
  - ../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md
---

# Capture Mutable Mock Arguments at Call Time

## Idea and Problem

Record the observable values of a reused mutable argument before the caller changes it between mock calls.

A mock's `call_args_list` retains argument references. If two calls receive the
same list and the caller clears it later, both records show the empty list.
Capture an immutable projection inside `side_effect` while each value is still
available. Returning `DEFAULT` preserves the configured mock return value.

## When to Use

Use this in a synchronous unit test when the code under test reuses a mutable
buffer and the values passed to a dependency are part of the contract. Pick the
fields that matter to the assertion and express them as immutable values. An
ordinary call assertion is sufficient for arguments that remain unchanged for
the life of the test.

## Implementation

```python
from dataclasses import dataclass
from unittest.mock import DEFAULT, create_autospec


@dataclass
class DispatchBatch:
    record_ids: list[int]


def dispatch_batch(batch: DispatchBatch, *, queue: str) -> str:
    """Supply the signature of the dependency that the test replaces."""
    raise NotImplementedError


recorded_calls: list[tuple[str, tuple[int, ...]]] = []


def record_dispatch(batch: DispatchBatch, *, queue: str) -> object:
    recorded_calls.append((queue, tuple(batch.record_ids)))
    return DEFAULT


dispatch = create_autospec(
    dispatch_batch,
    return_value="accepted",
    side_effect=record_dispatch,
)
```

## Example

```python
batch = DispatchBatch([101, 102])
first_response = dispatch(batch, queue="updates")
batch.record_ids[:] = [203]
second_response = dispatch(batch, queue="updates")
batch.record_ids.clear()

assert dispatch.call_count == 2
assert all(call.args[0] is batch for call in dispatch.call_args_list)
assert [call.args[0].record_ids for call in dispatch.call_args_list] == [[], []]
assert (first_response, second_response) == ("accepted", "accepted")
assert recorded_calls == [("updates", (101, 102)), ("updates", (203,))]
```

## Trade-offs and Limitations

`create_autospec` enforces the dependency's call signature. It does not validate
type annotations. The snapshot relies on `record_ids` containing integers. A
tuple of mutable elements would still refer to those elements. Converting the
list takes time and memory proportional to its length on each call. Keep the
projection small and initialize a fresh recorder and mock for each test.

This uses the mock's `side_effect` slot. If the dependency already needs a
custom action, combine the capture with that action or use a small fake. The
mock still retains the original arguments in its own history, so the projection
does not prevent retention of the original object. Concurrent mutations require
separate synchronization.

## Related Snippets

<!-- catalog:related:start -->
- [Verify Calls to a Standalone Function with Autospec](verify-calls-to-a-standalone-function-with-create-autospec.md)
- [Make a Defensive Copy at a Mutable Input Boundary](../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md)
<!-- catalog:related:end -->
