---
title: "Seal Test Mocks Against Unconfigured Collaborator Calls"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - verify-calls-to-a-standalone-function-with-create-autospec.md
  - verify-call-order-across-multiple-mocks-with-attach-mock.md
---

# Seal Test Mocks Against Unconfigured Collaborator Calls

## Idea and Problem

Reject unexpected collaborator calls even when the method exists on the real interface.

`spec_set` limits the attribute names a mock accepts. A permitted method can
still create its own child mock on first access, letting an unplanned call
succeed by returning another mock. Configure the methods the test expects,
then call `seal()` to stop automatic child creation.

## When to Use

Use this for a narrow unit test that supplies an interface-based mock and
expects only a few of its methods to participate. Set explicit return values
or side effects before sealing. The recipe fits tests where calling another
valid method is already a failure rather than harmless setup detail.

## Implementation

```python
from typing import Protocol
from unittest.mock import Mock, call, seal


class ReportStore(Protocol):
    def save(self, report: bytes) -> str: ...

    def remove(self, receipt: str) -> None: ...


def store_for_save_test() -> Mock:
    store = Mock(spec_set=ReportStore)
    store.save.return_value = "receipt-42"
    seal(store)
    return store
```

## Example

```python
report = b"summary report"
store = store_for_save_test()
receipt = store.save(report)

unsealed = Mock(spec_set=ReportStore)
assert isinstance(unsealed.remove(receipt), Mock)

try:
    store.remove(receipt)
except AttributeError:
    remove_rejected = True
else:
    remove_rejected = False

assert (receipt, remove_rejected, store.mock_calls) == (
    "receipt-42",
    True,
    [call.save(report)],
)
```

## Trade-offs and Limitations

Sealing stops automatic creation. Children you accessed before sealing still
exist, even if you did not mean to configure them. An assigned child mock
with its own name or spec can be outside recursive sealing. Seal that child
separately if its boundary matters.

Sealing does not verify signatures or call counts. Use autospec for signature
checks and explicit assertions for calls that must occur. Use a fake instead
when the collaborator needs meaningful state or a broad range of realistic
responses.

## Related Snippets

<!-- catalog:related:start -->
- [Verify Calls to a Standalone Function with Autospec](verify-calls-to-a-standalone-function-with-create-autospec.md)
- [Verify Call Order Across Attached Mocks](verify-call-order-across-multiple-mocks-with-attach-mock.md)
<!-- catalog:related:end -->
