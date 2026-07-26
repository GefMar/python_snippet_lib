---
title: "Handle Search Exhaustion with for/else"
snippet_type: idiom
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - keep-exception-handlers-narrow-with-try-else.md
  - batch-any-iterable-lazily-with-itertools-batched.md
---

# Handle Search Exhaustion with for/else

## Idea and Problem

Use a loop else clause to handle search failure only after an iterable ends without a matching break.

The matched item remains available after `break`, while the `else` block covers
both an empty iterable and a fully consumed one. This removes a separate
`found` flag and keeps the not-found branch next to the loop whose exhaustion
defines it.

## When to Use

Use this idiom when a search loop performs enough work that a generator passed
to `next()` would be less clear, and when the team recognizes Python's loop
`else` semantics. It fits a first-match search that should stop consuming a
lazy source immediately. Use `next((item for item in items if ...), sentinel)`
for a simple one-expression lookup.

## Implementation

```python
from collections.abc import Callable, Iterable
from typing import TypeVar


T = TypeVar("T")


class MatchNotFound(LookupError):
    pass


def require_first_match(
    items: Iterable[T],
    predicate: Callable[[T], object],
) -> T:
    if not callable(predicate):
        raise TypeError("predicate must be callable")
    for item in items:
        if predicate(item):
            break
    else:
        raise MatchNotFound("iterable contains no matching item")
    return item
```

## Example

```python
visited: list[int] = []


def candidates():
    for value in (1, 4, 9):
        visited.append(value)
        yield value


match = require_first_match(candidates(), lambda value: value % 2 == 0)

try:
    require_first_match([], lambda _value: True)
except MatchNotFound:
    empty_rejected = True
else:
    empty_rejected = False


class PredicateFailed(Exception):
    pass


def fail_predicate(_value: int) -> bool:
    raise PredicateFailed


try:
    require_first_match([1], fail_predicate)
except PredicateFailed:
    predicate_error_propagated = True
else:
    predicate_error_propagated = False

assert (
    match,
    visited,
    empty_rejected,
    predicate_error_propagated,
) == (4, [1, 4], True, True)
```

## Trade-offs and Limitations

Loop `else` is unfamiliar to some readers and becomes hard to follow in a long
body with several `break` statements or nested loops. The clause runs only
after normal exhaustion, including zero iterations; a `break` skips it, while
an exception or `return` leaves the construct entirely. The predicate may
return any truthy value and its exceptions propagate. This helper finds only
the first match and provides no partial result when the iterable is exhausted;
an infinite source without a match never returns.

## Related Snippets

<!-- catalog:related:start -->
- [Keep Exception Handlers Narrow with try/else](keep-exception-handlers-narrow-with-try-else.md)
- [Batch Any Iterable Lazily with itertools.batched](batch-any-iterable-lazily-with-itertools-batched.md)
<!-- catalog:related:end -->
