---
title: "Verify Calls to a Standalone Function with Autospec"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/check-a-bounded-abstract-call-shape-against-a-signature-without-invocation.md
  - verify-ordered-http-client-expectations-with-bounded-mismatch-reports.md
  - compare-a-bounded-text-capture-against-a-golden-fixture.md
---

# Verify Calls to a Standalone Function with Autospec

## Idea and Problem

Test that a small unit calls one injected standalone function with the exact supported arguments, order, and call count.

`create_autospec()` derives a mock signature from the real function. Invalid
positional arity or keyword names then fail at the call site instead of being
silently recorded by an unrestricted mock. Comparing the complete
`call_args_list` with an ordered tuple of `call(...)` records also detects
missing, reordered, and extra interactions.

The unit's returned state still needs its own assertion. The mock protects the
collaboration boundary; it does not prove that the real dependency behaves
correctly.

## When to Use

Use this technique for a local unit that receives one trusted plain Python
function by dependency injection and is expected to make a small closed set of
calls. It is useful when argument shape and order are part of the unit's
contract and calling the real dependency would perform unwanted I/O.

Prefer a small fake when collaborator state or realistic behavior matters.
Use an integration test when correctness depends on a filesystem, network,
database, or framework boundary. Avoid autospeccing live object graphs whose
properties or descriptors may execute during introspection.

## Implementation

```python
from collections.abc import Callable
from unittest.mock import call, create_autospec

_MAX_AUTOSPEC_CALLS = 64
_MAX_RECORD_ID_BYTES = 128
_MAX_RECORD_PAYLOAD_BYTES = 4_096

def store_record(
    record_id: str,
    payload: bytes,
    *,
    replace: bool,
) -> str:
    """Represent the standalone collaborator signature; never call it here."""
    raise NotImplementedError


def persist_record_batch(
    records: tuple[tuple[str, bytes], ...],
    writer: Callable[..., str],
) -> tuple[str, ...]:
    """Validate a bounded batch and delegate each record in declaration order."""
    if type(records) is not tuple:
        raise TypeError("records must be an exact tuple")
    if len(records) > _MAX_AUTOSPEC_CALLS:
        raise ValueError("record count exceeds the supported limit")

    results: list[str] = []
    for record_index, record in enumerate(records):
        if type(record) is not tuple:
            raise TypeError(f"records[{record_index}] must be an exact tuple")
        if len(record) != 2:
            raise ValueError(f"records[{record_index}] must contain id and payload")
        record_id, payload = record
        if type(record_id) is not str:
            raise TypeError(f"records[{record_index}].id must be an exact string")
        if not 1 <= len(record_id) <= _MAX_RECORD_ID_BYTES:
            raise ValueError(
                f"records[{record_index}].id cannot fit the UTF-8 byte limit"
            )
        try:
            record_id_size = len(record_id.encode("utf-8", errors="strict"))
        except UnicodeEncodeError as error:
            raise ValueError(
                f"records[{record_index}].id must be valid UTF-8 text"
            ) from error
        if not 1 <= record_id_size <= _MAX_RECORD_ID_BYTES:
            raise ValueError(f"records[{record_index}].id has an invalid byte size")
        if type(payload) is not bytes:
            raise TypeError(f"records[{record_index}].payload must be exact bytes")
        if len(payload) > _MAX_RECORD_PAYLOAD_BYTES:
            raise ValueError(f"records[{record_index}].payload is too large")
        results.append(writer(record_id, payload, replace=False))
    return tuple(results)
```

## Example

```python
def exact_calls(mock, expected: tuple[object, ...]) -> None:
    if tuple(mock.call_args_list) != expected:
        raise AssertionError(
            f"expected exact calls {expected!r}, got {mock.call_args_list!r}"
        )


def raises(error_type: type[Exception], operation) -> bool:
    try:
        operation()
    except error_type:
        return True
    return False


records = (("alpha", b"one"), ("beta", b"two"), ("gamma", b"three"))
expected_calls = tuple(
    call(record_id, payload, replace=False) for record_id, payload in records
)

writer = create_autospec(store_record, spec_set=True)
writer.side_effect = lambda record_id, payload, *, replace: (
    f"{record_id}:{len(payload)}:{replace}"
)
results = persist_record_batch(records, writer)
exact_calls(writer, expected_calls)

wrong_order_detected = False
try:
    exact_calls(writer, tuple(reversed(expected_calls)))
except AssertionError:
    wrong_order_detected = True

extra_writer = create_autospec(store_record, spec_set=True, return_value="ok")
persist_record_batch(records, extra_writer)
extra_writer("delta", b"four", replace=False)
extra_call_detected = False
try:
    exact_calls(extra_writer, expected_calls)
except AssertionError:
    extra_call_detected = True

assert (
    results == ("alpha:3:False", "beta:3:False", "gamma:5:False")
    and wrong_order_detected
    and extra_call_detected
    and raises(TypeError, lambda: writer("alpha", b"one"))
    and raises(
        TypeError,
        lambda: writer("alpha", b"one", replace=False, unknown=True),
    )
    and tuple(writer.call_args_list) == expected_calls
)
```

## Trade-offs and Limitations

Creating and comparing at most `C` recorded calls uses `O(C)` time and memory.
`create_autospec()` validates the standalone function's callable signature.
Here, `spec_set=True` constrains its backing mock, but the returned function-like
wrapper can still receive arbitrary Python attributes. Neither feature enforces
annotations, runtime value types, return correctness, business semantics, or
real collaborator effects.

The pattern deliberately targets one trusted plain Python function. It avoids
live instances, bound methods, descriptors, arbitrary callable objects,
patching, network access, and I/O. Autospec introspection is not a substitute
for understanding a complex object boundary.

Exact interaction assertions can make a test brittle when call order is an
implementation detail. Use them only when the collaboration sequence is part
of observable behavior, and always assert the unit's meaningful output or
state as well.

## Related Snippets

<!-- catalog:related:start -->
- [Check a Bounded Abstract Call Shape Against a Signature Without Invocation](../python-language/check-a-bounded-abstract-call-shape-against-a-signature-without-invocation.md)
- [Verify Ordered HTTP Client Expectations with Bounded Mismatch Reports](verify-ordered-http-client-expectations-with-bounded-mismatch-reports.md)
- [Compare a Bounded Text Capture Against a Golden Fixture](compare-a-bounded-text-capture-against-a-golden-fixture.md)
<!-- catalog:related:end -->
