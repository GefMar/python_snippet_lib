---
title: "Isolate Independently Failing Items by Bisecting a Bounded Batch"
snippet_type: algorithm
use_cases:
  - data-transformation
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - collect-expected-parse-failures-without-stopping-a-batch.md
  - batch-items-by-estimated-byte-size.md
  - ../reliability-resilience/retry-only-eligible-items-in-a-bounded-batch.md
---

# Isolate Independently Failing Items by Bisecting a Bounded Batch

## Idea and Problem

Find item-specific failures behind an all-or-error batch operation by recursively probing two smaller halves.

Some batch APIs report only that a group was rejected, without identifying the
responsible item. If failure is deterministic and hereditary—every group that
contains a bad item fails—bisection can classify passing items and failing
singletons while preserving input order. A finite input and explicit probe
budget keep the diagnostic work bounded.

## When to Use

Use this algorithm only when the probe is pure or safely repeatable, either
accepts the whole supplied tuple or raises one declared item-rejection
exception, and each failure belongs to an individual item. Items need stable,
unique identifiers so the result can be reconciled without relying on object
identity.

Do not use it for transient outages, timeouts, cancellation, capacity limits,
order-sensitive behavior, duplicate-sensitive behavior, or failures caused by
a combination of otherwise valid items. Unexpected exceptions propagate. If a
probe can partially apply side effects before rejecting, use an isolated dry
run or a transactional validation endpoint instead.

## Implementation

```python
from collections.abc import Callable, Hashable, Sequence
from dataclasses import dataclass
from itertools import islice
from typing import Generic, TypeVar


T = TypeVar("T")
_MAX_ITEMS = 128


class BatchRejected(Exception):
    """Expected signal that at least one supplied item is invalid."""


class ProbeBudgetExceeded(RuntimeError):
    pass


class NonIndependentFailure(RuntimeError):
    pass


@dataclass(frozen=True, slots=True)
class IsolationResult(Generic[T]):
    passing: tuple[T, ...]
    failing: tuple[T, ...]
    probe_count: int


def isolate_failing_items(
    values: Sequence[T],
    probe: Callable[..., None],
    *,
    item_id: Callable[..., Hashable],
    rejection: type[BatchRejected] = BatchRejected,
    max_probes: int,
) -> IsolationResult[T]:
    if not isinstance(values, Sequence) or isinstance(values, (str, bytes)):
        raise TypeError("values must be a non-text sequence")
    items = tuple(islice(values, _MAX_ITEMS + 1))
    if not 1 <= len(items) <= _MAX_ITEMS:
        raise ValueError("item count is outside the supported range")
    if not callable(probe) or not callable(item_id):
        raise TypeError("probe and item_id must be callable")
    if (
        not isinstance(rejection, type)
        or not issubclass(rejection, BatchRejected)
    ):
        raise TypeError("rejection must be a BatchRejected subclass")

    worst_case = 2 * len(items) - 1
    if isinstance(max_probes, bool) or not isinstance(max_probes, int):
        raise TypeError("max_probes must be an integer")
    if not 1 <= max_probes <= worst_case:
        raise ValueError("max_probes must be between one and 2n - 1")

    identifiers: set[Hashable] = set()
    for item in items:
        identifier = item_id(item)
        try:
            duplicate = identifier in identifiers
            identifiers.add(identifier)
        except TypeError as error:
            raise TypeError("item IDs must be hashable") from error
        if duplicate:
            raise ValueError("item IDs must be unique")

    probe_count = 0

    def visit(start: int, stop: int) -> frozenset[int]:
        nonlocal probe_count
        if probe_count >= max_probes:
            raise ProbeBudgetExceeded("the probe budget was exhausted")

        group = items[start:stop]
        probe_count += 1
        try:
            returned = probe(group)
        except rejection:
            if stop - start == 1:
                return frozenset((start,))

            middle = start + (stop - start) // 2
            left_failures = visit(start, middle)
            right_failures = visit(middle, stop)
            if not left_failures and not right_failures:
                raise NonIndependentFailure(
                    "a rejected group split into two passing groups"
                )
            return left_failures | right_failures

        if returned is not None:
            raise TypeError("a successful probe must return None")
        return frozenset()

    failing_indices = visit(0, len(items))
    passing = tuple(
        item for index, item in enumerate(items) if index not in failing_indices
    )
    failing = tuple(
        item for index, item in enumerate(items) if index in failing_indices
    )
    return IsolationResult(passing, failing, probe_count)
```

## Example

```python
records = ("alpha", "broken-a", "beta", "broken-b", "gamma")


def validate_group(group: tuple[str, ...]) -> None:
    if any(record.startswith("broken-") for record in group):
        raise BatchRejected("the group contains an invalid record")


result = isolate_failing_items(
    records,
    validate_group,
    item_id=lambda record: record,
    max_probes=2 * len(records) - 1,
)

assert (result.passing, result.failing, result.probe_count) == (
    ("alpha", "beta", "gamma"),
    ("broken-a", "broken-b"),
    9,
)
```

## Trade-offs and Limitations

The worst case visits a full binary tree and performs `2n - 1` probes, so this
is a bounded diagnostic technique rather than a cheap replacement for precise
per-item errors. The recursion depth is logarithmic, but every successful
probe may still repeat expensive work. Set a smaller budget only when failure
with no partial result is acceptable.

The algorithm cannot prove its independence assumption. It detects the simple
contradiction where a rejected group splits into two passing groups, but
stateful or intermittent probes can still produce plausible yet incorrect
classifications. Preserve the original failure evidence and fix the batch API
to report per-item outcomes when that protocol is under your control.

## Related Snippets

<!-- catalog:related:start -->
- [Collect Expected Parse Failures Without Stopping a Batch](collect-expected-parse-failures-without-stopping-a-batch.md)
- [Batch Items by Estimated Byte Size](batch-items-by-estimated-byte-size.md)
- [Retry Only Eligible Items in a Bounded Batch](../reliability-resilience/retry-only-eligible-items-in-a-bounded-batch.md)
<!-- catalog:related:end -->
