---
title: "Verify Call Order Across Attached Mocks"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - verify-calls-to-a-standalone-function-with-create-autospec.md
  - capture-mutable-mock-arguments-at-call-time.md
---

# Verify Call Order Across Attached Mocks

## Idea and Problem

Verify that calls to separate dependencies happen in one expected order, such as recording a message before acknowledging it.

Separate call histories cannot show which mock was called first. Attach the
mocks to a common parent with `attach_mock()` before running the operation.
The parent's `mock_calls` list then records calls under their attached
names, so one exact sequence can detect missing, reordered, or extra calls.

## When to Use

Use this in a synchronous unit test when order across two or more injected
dependencies is part of the behavior. Test only interactions whose order
matters. Attach fresh mocks so every recorded call belongs to the operation
under test.

## Implementation

```python
from collections.abc import Callable
from unittest.mock import Mock, call


def deliver_record(
    payload: bytes,
    *,
    persist: Callable[[bytes], str],
    confirm: Callable[[str], None],
) -> None:
    receipt = persist(payload)
    confirm(receipt)


timeline = Mock()
persist = Mock(name="storage", return_value="receipt-42")
confirm = Mock(name="transport")
timeline.attach_mock(persist, "persist")
timeline.attach_mock(confirm, "confirm")
```

## Example

```python
payload = b"daily report"
deliver_record(payload, persist=persist, confirm=confirm)
assert timeline.mock_calls == [call.persist(payload), call.confirm("receipt-42")]

timeline.reset_mock()
persist.side_effect = OSError("storage unavailable")
try:
    deliver_record(payload, persist=persist, confirm=confirm)
except OSError:
    pass
else:
    raise AssertionError("the persistence error must propagate")

assert timeline.mock_calls == [call.persist(payload)]
```

## Trade-offs and Limitations

The timeline records invocations, including calls whose side effect raises.
It cannot establish that real storage committed data, that a background task
finished, or that concurrent operations completed in order. Use an
integration check or record completion events when those outcomes matter.

An exact interaction sequence can make tests sensitive to incidental calls.
Keep the mocked boundary narrow. A mock also retains argument references,
so capture values at invocation time if later mutations could change what
the assertion observes.

## Related Snippets

<!-- catalog:related:start -->
- [Verify Calls to a Standalone Function with Autospec](verify-calls-to-a-standalone-function-with-create-autospec.md)
- [Capture Mutable Mock Arguments at Call Time](capture-mutable-mock-arguments-at-call-time.md)
<!-- catalog:related:end -->
