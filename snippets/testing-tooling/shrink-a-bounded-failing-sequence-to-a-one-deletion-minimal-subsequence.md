---
title: "Shrink a Bounded Failing Sequence to a One-Deletion-Minimal Subsequence"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/isolate-independently-failing-items-by-bisecting-a-bounded-batch.md
  - compare-a-bounded-text-capture-against-a-golden-fixture.md
  - extract-bounded-native-test-failure-highlights.md
---

# Shrink a Bounded Failing Sequence to a One-Deletion-Minimal Subsequence

## Idea and Problem

Remove steps from a reproducibly failing sequence until no single retained step can be removed without making the failure disappear.

Starting again at position zero after every accepted deletion gives one
deterministic result for a fixed pure predicate. The retained original indexes
distinguish duplicate labels and let diagnostics project the minimized witness
back onto the supplied sequence.

## When to Use

Use this reducer for small, immutable sequences whose failure depends on an
interaction among steps, such as a state-machine trace or a generated command
sequence. The predicate must be pure, deterministic, and safe to call repeatedly
with candidate tuples, including the empty tuple.

Preserve the complete original failure separately. This routine is a bounded
diagnostic aid, not a replacement for a precise test oracle, and its result is
one-deletion-minimal rather than globally shortest.

## Implementation

```python
from collections.abc import Callable
from dataclasses import dataclass

StepSequence = tuple[str, ...]

_MAX_STEPS = 64
_MAX_STEP_CHARACTERS = 256
_MAX_STEP_BYTES = 1_024
_MAX_TOTAL_BYTES = 65_536


@dataclass(frozen=True, slots=True)
class ShrinkResult:
    retained_steps: tuple[str, ...]
    original_indexes: tuple[int, ...]
    evaluation_count: int


def shrink_failing_sequence(
    steps: StepSequence,
    failure_predicate: Callable[[StepSequence], bool],
) -> ShrinkResult:
    if type(steps) is not tuple:
        raise TypeError("steps must be an exact tuple")
    if not 1 <= len(steps) <= _MAX_STEPS:
        raise ValueError("step count is outside the supported range")
    if not callable(failure_predicate):
        raise TypeError("failure_predicate must be callable")

    total_bytes = 0
    for step in steps:
        if type(step) is not str:
            raise TypeError("each step must be an exact string")
        if not step:
            raise ValueError("steps must not be empty strings")
        if len(step) > _MAX_STEP_CHARACTERS:
            raise ValueError("a step exceeds the character limit")
        try:
            encoded = step.encode("utf-8")
        except UnicodeEncodeError as error:
            raise ValueError("each step must be valid UTF-8 text") from error
        if len(encoded) > _MAX_STEP_BYTES:
            raise ValueError("a step exceeds the UTF-8 byte limit")
        total_bytes += len(encoded)
    if total_bytes > _MAX_TOTAL_BYTES:
        raise ValueError("steps exceed the aggregate UTF-8 byte limit")

    evaluation_limit = 1 + len(steps) * (len(steps) + 1) // 2
    evaluation_count = 0

    def still_fails(candidate: tuple[str, ...]) -> bool:
        nonlocal evaluation_count
        if evaluation_count >= evaluation_limit:
            raise RuntimeError("failure predicate evaluation limit exceeded")
        evaluation_count += 1
        outcome = failure_predicate(candidate)
        if type(outcome) is not bool:
            raise TypeError("failure_predicate must return an exact bool")
        return outcome

    if not still_fails(steps):
        raise ValueError("the complete sequence must fail")

    retained_steps = steps
    original_indexes = tuple(range(len(steps)))
    position = 0
    while position < len(retained_steps):
        candidate = retained_steps[:position] + retained_steps[position + 1 :]
        if still_fails(candidate):
            retained_steps = candidate
            original_indexes = original_indexes[:position] + original_indexes[position + 1 :]
            position = 0
        else:
            position += 1

    return ShrinkResult(retained_steps, original_indexes, evaluation_count)
```

## Example

```python
sequence = ("setup", "trigger", "trigger", "cleanup")


def duplicate_trigger_fails(candidate: tuple[str, ...]) -> bool:
    return candidate.count("trigger") >= 2


minimal = shrink_failing_sequence(sequence, duplicate_trigger_fails)
empty = shrink_failing_sequence(("only",), lambda candidate: True)

assert minimal == ShrinkResult(
    retained_steps=("trigger", "trigger"),
    original_indexes=(1, 2),
    evaluation_count=7,
)
assert empty == ShrinkResult((), (), 2)
assert duplicate_trigger_fails(minimal.retained_steps)
assert all(
    not duplicate_trigger_fails(
        minimal.retained_steps[:index] + minimal.retained_steps[index + 1 :]
    )
    for index in range(len(minimal.retained_steps))
)
```

## Trade-offs and Limitations

For `n` input steps, the initial check plus deletion attempts make at most
`1 + n(n + 1) / 2` predicate calls: no more than 2,081 at the fixed limit.
There are `O(n^2)` predicate calls, `O(n^3)` local tuple-reference copying, and
`O(n)` auxiliary memory, excluding the predicate's own time and memory.

Ascending deletion order is deterministic but may miss a shorter failing
subsequence. Predicate exceptions and non-boolean returns abort without a
partial result. Stateful, intermittent, effectful, or mutating predicates,
timeouts, parallel evaluation, and arbitrary nested input structures are
outside this contract.

## Related Snippets

<!-- catalog:related:start -->
- [Isolate Independently Failing Items by Bisecting a Bounded Batch](../data-processing/isolate-independently-failing-items-by-bisecting-a-bounded-batch.md)
- [Compare a Bounded Text Capture Against a Golden Fixture](compare-a-bounded-text-capture-against-a-golden-fixture.md)
- [Extract Bounded Native-Test Failure Highlights](extract-bounded-native-test-failure-highlights.md)
<!-- catalog:related:end -->
