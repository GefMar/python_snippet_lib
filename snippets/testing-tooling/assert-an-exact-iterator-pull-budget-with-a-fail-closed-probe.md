---
title: "Assert an Exact Iterator Pull Budget with a Fail-Closed Probe"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/handle-search-exhaustion-with-for-else.md
  - shrink-a-bounded-failing-sequence-to-a-one-deletion-minimal-subsequence.md
---

# Assert an Exact Iterator Pull Budget with a Fail-Closed Probe

## Idea and Problem

Wrap a bounded tuple in a test iterator that counts source pulls and raises an assertion before a consumer can exceed the declared budget.

Every permitted call that reaches the tuple iterator consumes one pull,
including the first call that discovers `StopIteration`. Once exhaustion has
actually been observed, later calls return cached `StopIteration` without
changing the count or consulting the budget.

## When to Use

Use this probe in a focused unit test when laziness is observable behavior: a
first-match search should stop early, a bounded consumer must not read ahead,
or complete exhaustion must occur within a known number of pulls. Read the
`pulls` property after the operation to assert the exact interaction.

Set the budget from the behavior being tested, not from an arbitrary timeout.
Fully exhausting `n` tuple items generally needs `n + 1` pulls because one
additional call must discover the end. Use ordinary result assertions when
the number of iterator interactions is not part of the contract.

## Implementation

```python
from collections.abc import Iterator
from typing import TypeVar

ItemT = TypeVar("ItemT")
_MAX_PROBE_ITEMS = 1_024


class IteratorPullBudgetExceeded(AssertionError):
    def __init__(self, *, pulls: int, pull_budget: int) -> None:
        self.pulls = pulls
        self.pull_budget = pull_budget
        super().__init__(
            f"iterator pull budget exceeded after {pulls} of {pull_budget} pulls"
        )


class IteratorPullBudgetProbe(Iterator[ItemT]):
    def __init__(
        self,
        values: tuple[ItemT, ...],
        *,
        pull_budget: int,
    ) -> None:
        if type(values) is not tuple:
            raise TypeError("values must be an exact tuple")
        if len(values) > _MAX_PROBE_ITEMS:
            raise ValueError("values exceed the supported item limit")
        if type(pull_budget) is not int:
            raise TypeError("pull_budget must be an exact non-boolean integer")
        if not 0 <= pull_budget <= len(values) + 1:
            raise ValueError("pull_budget is outside the supported range")

        self._source = iter(values)
        self._pull_budget = pull_budget
        self._pulls = 0
        self._exhausted = False

    @property
    def pulls(self) -> int:
        """Return the number of calls forwarded to the tuple iterator."""
        return self._pulls

    def __iter__(self) -> Iterator[ItemT]:
        return self

    def __next__(self) -> ItemT:
        if self._exhausted:
            raise StopIteration
        if self._pulls >= self._pull_budget:
            raise IteratorPullBudgetExceeded(
                pulls=self._pulls,
                pull_budget=self._pull_budget,
            )

        self._pulls += 1
        try:
            return next(self._source)
        except StopIteration:
            self._exhausted = True
            raise
```

## Example

```python
early = IteratorPullBudgetProbe(
    ("skip", "match", "unread"),
    pull_budget=2,
)
matched = next(value for value in early if value == "match")
assert (matched, early.pulls) == ("match", 2)

try:
    next(early)
except IteratorPullBudgetExceeded as error:
    assert (error.pulls, error.pull_budget, early.pulls) == (2, 2, 2)
else:
    raise AssertionError("the probe allowed a pull beyond its budget")

complete = IteratorPullBudgetProbe((10, 20), pull_budget=3)
assert tuple(complete) == (10, 20)
assert complete.pulls == 3

for _ in range(2):
    try:
        next(complete)
    except StopIteration:
        pass
    else:
        raise AssertionError("an exhausted probe produced another item")
assert complete.pulls == 3

blocked = IteratorPullBudgetProbe(("untouched",), pull_budget=0)
for _ in range(2):
    try:
        next(blocked)
    except IteratorPullBudgetExceeded:
        pass
    else:
        raise AssertionError("a zero-budget probe touched its source")
assert blocked.pulls == 0
```

## Trade-offs and Limitations

Each pull adds `O(1)` work. The probe retains the exact input tuple, so memory
is `O(n)` in the supplied fixture and is capped at 1,024 item references. The
budget counts source `next()` attempts, not yielded values: the first observed
exhaustion counts, while repeated cached exhaustion does not.

This is a fail-closed unit-test assertion helper for one synchronous consumer.
It is not thread-safe and does not support arbitrary iterables, async
iteration, timing limits, fault injection, retries, prefetching protocols, or
production rate limiting. A budget failure deliberately prevents all later
source access; create a new probe instead of trying to resume the test.

## Related Snippets

<!-- catalog:related:start -->
- [Handle Search Exhaustion with for/else](../python-language/handle-search-exhaustion-with-for-else.md)
- [Shrink a Bounded Failing Sequence to a One-Deletion-Minimal Subsequence](shrink-a-bounded-failing-sequence-to-a-one-deletion-minimal-subsequence.md)
<!-- catalog:related:end -->
