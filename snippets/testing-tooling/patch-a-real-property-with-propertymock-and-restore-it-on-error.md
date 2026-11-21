---
title: "Patch a Real Property with PropertyMock and Restore It on Error"
snippet_type: testing-technique
use_cases:
  - lifecycle-management
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - capture-mutable-mock-arguments-at-call-time.md
  - clean-up-unittest-fixtures-when-setup-fails-with-entercontext.md
---

# Patch a Real Property with PropertyMock and Restore It on Error

## Idea and Problem

Control what code reads from a real property without changing the instance or leaving the class patched.

Python looks up a property descriptor on the class. Use a scoped
`patch.object()` with `PropertyMock` on the class that owns the descriptor.
Reading the replacement calls the mock without arguments; exiting the patch
restores the original descriptor even when the test path raises.

## When to Use

Use this to test how a caller responds to a property value or failure,
without exercising the real getter in that test. Use a real receiver and a
short, isolated patch scope. The example keeps the ordinary class behavior
apart from the property under test.

## Implementation

```python
from unittest.mock import PropertyMock, patch


class Batch:
    def __init__(self, pending: int) -> None:
        self.pending = pending

    @property
    def ready(self) -> bool:
        return self.pending == 0


def next_action(batch: Batch) -> str:
    return "publish" if batch.ready else "wait"
```

## Example

```python
batch = Batch(pending=2)
original = vars(Batch)["ready"]
before = next_action(batch)
failure = RuntimeError("status unavailable")

try:
    with patch.object(Batch, "ready", new_callable=PropertyMock, return_value=True) as ready:
        during = next_action(batch)
        ready.assert_called_once_with()
        ready.side_effect = failure
        next_action(batch)
except RuntimeError as error:
    failed_as_expected = error is failure
else:
    failed_as_expected = False

assert (
    (before, during, next_action(batch)) == ("wait", "publish", "wait")
    and failed_as_expected
    and ready.call_count == 2
    and vars(Batch)["ready"] is original
)
```

## Trade-offs and Limitations

The patch affects every instance that resolves this class attribute during
the scope. Avoid concurrent tests that share the class. This tests the
caller, not the original property's implementation.

Use an explicit failure such as `RuntimeError` for this test. A descriptor
raising `AttributeError` can trigger `__getattr__` instead. On a mock
receiver that fallback can return a child mock.

## Related Snippets

<!-- catalog:related:start -->
- [Capture Mutable Mock Arguments at Call Time](capture-mutable-mock-arguments-at-call-time.md)
- [Clean Up unittest Fixtures When setUp Fails with enterContext](clean-up-unittest-fixtures-when-setup-fails-with-entercontext.md)
<!-- catalog:related:end -->
