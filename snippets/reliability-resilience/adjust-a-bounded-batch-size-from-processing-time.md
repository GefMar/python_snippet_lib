---
title: "Adjust a Bounded Batch Size from Processing Time"
snippet_type: algorithm
use_cases:
  - performance-optimization
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - flush-fixed-size-batches-through-a-scoped-sink.md
  - ../data-processing/batch-items-by-estimated-byte-size.md
---

# Adjust a Bounded Batch Size from Processing Time

## Idea and Problem

Apply one deterministic batch-size adjustment when a completed full batch falls outside an explicit processing-time deadband.

The policy compares elapsed time with half and twice a target duration. A full
batch below the lower threshold doubles once; one above the upper threshold
halves once. The configured size limits clamp either change. Exact threshold
values remain stable, and an incomplete batch never changes the size because
its processing time does not describe the cost of a full batch.

## When to Use

Use this transition when a caller processes repeated bounded batches, measures
each completed attempt, and wants a deliberately coarse response to sustained
under- or over-sized work units. Keep the returned size with the caller's
state, then feed it into the next batch acquisition step.

The target must describe comparable work. If item costs vary greatly, use byte
or cost estimates instead of item count. Add an independently specified moving
average or hysteresis policy when one measurement is too noisy; this helper
intentionally has no hidden history.

## Implementation

```python
import math
from dataclasses import dataclass
from enum import StrEnum


_MAX_BATCH_SIZE = 1_000_000
_MAX_TARGET_SECONDS = 86_400.0


class BatchAdjustmentAction(StrEnum):
    INCOMPLETE = "incomplete-batch"
    STABLE = "inside-deadband"
    INCREASED = "increased"
    DECREASED = "decreased"
    AT_MINIMUM = "at-minimum"
    AT_MAXIMUM = "at-maximum"


@dataclass(frozen=True, slots=True)
class BatchSizePolicy:
    target_seconds: float
    minimum_size: int
    maximum_size: int


@dataclass(frozen=True, slots=True)
class BatchSizeAdjustment:
    previous_size: int
    next_size: int
    action: BatchAdjustmentAction


def _finite_seconds(value: object, *, field: str, allow_zero: bool) -> float:
    if type(value) not in (int, float):
        raise TypeError(f"{field} must be an integer or float")
    try:
        result = float(value)
    except OverflowError as error:
        raise ValueError(f"{field} must fit in a finite float") from error
    if (
        not math.isfinite(result)
        or result < 0.0
        or (not allow_zero and result == 0.0)
    ):
        qualifier = "non-negative" if allow_zero else "positive"
        raise ValueError(f"{field} must be finite and {qualifier}")
    return result


def adjust_batch_size(
    policy: BatchSizePolicy,
    current_size: int,
    elapsed_seconds: float,
    *,
    batch_was_full: bool,
) -> BatchSizeAdjustment:
    if type(policy) is not BatchSizePolicy:
        raise TypeError("policy must be a BatchSizePolicy")
    for field, value in (
        ("minimum_size", policy.minimum_size),
        ("maximum_size", policy.maximum_size),
    ):
        if type(value) is not int:
            raise TypeError(f"{field} must be an integer")
    if not 1 <= policy.minimum_size <= policy.maximum_size <= _MAX_BATCH_SIZE:
        raise ValueError("batch-size limits are outside the supported range")
    target = _finite_seconds(
        policy.target_seconds,
        field="target_seconds",
        allow_zero=False,
    )
    if target > _MAX_TARGET_SECONDS:
        raise ValueError("target_seconds exceeds the supported limit")
    if type(current_size) is not int:
        raise TypeError("current_size must be an integer")
    if not policy.minimum_size <= current_size <= policy.maximum_size:
        raise ValueError("current_size must be within the policy limits")
    elapsed = _finite_seconds(
        elapsed_seconds,
        field="elapsed_seconds",
        allow_zero=True,
    )
    if type(batch_was_full) is not bool:
        raise TypeError("batch_was_full must be a bool")

    if not batch_was_full:
        return BatchSizeAdjustment(
            current_size,
            current_size,
            BatchAdjustmentAction.INCOMPLETE,
        )

    if elapsed * 2.0 < target:
        next_size = min(policy.maximum_size, current_size * 2)
        action = (
            BatchAdjustmentAction.INCREASED
            if next_size > current_size
            else BatchAdjustmentAction.AT_MAXIMUM
        )
    elif elapsed > target * 2.0:
        next_size = max(policy.minimum_size, current_size // 2)
        action = (
            BatchAdjustmentAction.DECREASED
            if next_size < current_size
            else BatchAdjustmentAction.AT_MINIMUM
        )
    else:
        next_size = current_size
        action = BatchAdjustmentAction.STABLE

    return BatchSizeAdjustment(current_size, next_size, action)
```

## Example

```python
policy = BatchSizePolicy(
    target_seconds=4.0,
    minimum_size=4,
    maximum_size=64,
)

fast = adjust_batch_size(policy, 16, 1.9, batch_was_full=True)
lower_boundary = adjust_batch_size(policy, 16, 2.0, batch_was_full=True)
slow = adjust_batch_size(policy, 15, 8.1, batch_was_full=True)
partial = adjust_batch_size(policy, 16, 0.2, batch_was_full=False)

assert (fast, lower_boundary, slow, partial) == (
    BatchSizeAdjustment(16, 32, BatchAdjustmentAction.INCREASED),
    BatchSizeAdjustment(16, 16, BatchAdjustmentAction.STABLE),
    BatchSizeAdjustment(15, 7, BatchAdjustmentAction.DECREASED),
    BatchSizeAdjustment(16, 16, BatchAdjustmentAction.INCOMPLETE),
)
```

## Trade-offs and Limitations

This rule reacts to a single duration, so alternating fast and slow batches
can make the size oscillate. The wide deadband and one-step changes limit the
effect but do not provide statistical smoothing. Integer halving rounds down,
and clamping can report a boundary action without changing the current size.

Elapsed processing time can also reflect unrelated load, retries, or external
latency. The function does not read a clock, acquire work, perform I/O, retain
history, coordinate concurrent workers, or claim that the chosen size is
optimal. Keep those responsibilities and any cross-worker synchronization in
the surrounding system.

## Related Snippets

<!-- catalog:related:start -->
- [Flush Fixed-Size Batches Through a Scoped Sink](flush-fixed-size-batches-through-a-scoped-sink.md)
- [Batch Items by Estimated Byte Size](../data-processing/batch-items-by-estimated-byte-size.md)
<!-- catalog:related:end -->
