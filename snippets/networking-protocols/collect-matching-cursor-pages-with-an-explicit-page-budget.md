---
title: "Collect Matching Cursor Pages with an Explicit Page Budget"
snippet_type: pattern
use_cases:
  - networking
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - iterate-cursor-paginated-results-lazily.md
---

# Collect Matching Cursor Pages with an Explicit Page Budget

## Idea and Problem

Collect a requested number of matching items across cursor pages while making every successful stop condition explicit.

Filtering can require more pages than the requested result count, including
empty pages. A structured result distinguishes reaching the item limit, true
cursor exhaustion, and consuming the caller's page budget without treating a
falsey cursor as exhaustion or guessing that an empty page is terminal.

## When to Use

Use this pattern when a synchronous cursor API applies filtering on the client
and the caller needs a bounded materialized result rather than a lazy stream.
The fetch callback must return ordered, materialized pages, cursor values must
be hashable, and `None` must be the protocol's only exhaustion marker. Choose a
positive page budget that limits calls even when few items match.

Use the related lazy iterator when consumers can stop naturally while reading
items and do not need a structured filtered-count decision. Keep transport
timeouts, authentication, retries, and rate-limit policy in the API client.

## Implementation

```python
from collections.abc import Callable, Hashable
from dataclasses import dataclass
from enum import StrEnum
from typing import Generic, TypeVar

ItemT = TypeVar("ItemT")
CursorT = TypeVar("CursorT", bound=Hashable)


class CollectionStop(StrEnum):
    LIMIT = "limit"
    EXHAUSTED = "exhausted"
    BUDGET = "budget"


@dataclass(frozen=True, slots=True)
class CursorPage(Generic[ItemT, CursorT]):
    items: tuple[ItemT, ...]
    next_cursor: CursorT | None


@dataclass(frozen=True, slots=True)
class MatchingCollection(Generic[ItemT, CursorT]):
    items: tuple[ItemT, ...]
    stop: CollectionStop
    next_cursor: CursorT | None
    pages_fetched: int


class CursorCycleError(RuntimeError):
    pass


def _bounded_integer(value: int, *, name: str, minimum: int) -> int:
    if isinstance(value, bool) or not isinstance(value, int):
        raise TypeError(f"{name} must be an integer")
    if value < minimum:
        raise ValueError(f"{name} must be at least {minimum}")
    return value


def _remember_cursor(
    cursor: CursorT | None,
    seen: set[CursorT | None],
) -> None:
    try:
        repeated = cursor in seen
        seen.add(cursor)
    except TypeError as error:
        raise TypeError("cursors must be hashable") from error
    if repeated:
        raise CursorCycleError("cursor cycle detected")


def _reject_seen_cursor(
    cursor: CursorT,
    seen: set[CursorT | None],
) -> None:
    try:
        repeated = cursor in seen
    except TypeError as error:
        raise TypeError("cursors must be hashable") from error
    if repeated:
        raise CursorCycleError("cursor cycle detected")


def collect_matching_cursor_pages(
    fetch_page: Callable[[CursorT | None], CursorPage[ItemT, CursorT]],
    predicate: Callable[[ItemT], bool],
    *,
    limit: int,
    page_budget: int,
    initial_cursor: CursorT | None = None,
) -> MatchingCollection[ItemT, CursorT]:
    if not callable(fetch_page):
        raise TypeError("fetch_page must be callable")
    if not callable(predicate):
        raise TypeError("predicate must be callable")
    limit = _bounded_integer(limit, name="limit", minimum=0)
    page_budget = _bounded_integer(page_budget, name="page_budget", minimum=1)

    if limit == 0:
        return MatchingCollection((), CollectionStop.LIMIT, None, 0)

    matches: list[ItemT] = []
    cursor = initial_cursor
    seen: set[CursorT | None] = set()

    for pages_fetched in range(1, page_budget + 1):
        _remember_cursor(cursor, seen)
        page = fetch_page(cursor)
        if not isinstance(page, CursorPage):
            raise TypeError("fetch_page must return CursorPage")
        if not isinstance(page.items, tuple):
            raise TypeError("CursorPage.items must be a tuple")

        for item in page.items:
            if predicate(item):
                matches.append(item)
                if len(matches) == limit:
                    return MatchingCollection(
                        tuple(matches),
                        CollectionStop.LIMIT,
                        None,
                        pages_fetched,
                    )

        next_cursor = page.next_cursor
        if next_cursor is None:
            return MatchingCollection(
                tuple(matches),
                CollectionStop.EXHAUSTED,
                None,
                pages_fetched,
            )

        _reject_seen_cursor(next_cursor, seen)
        if pages_fetched == page_budget:
            return MatchingCollection(
                tuple(matches),
                CollectionStop.BUDGET,
                next_cursor,
                pages_fetched,
            )
        cursor = next_cursor

    raise AssertionError("positive page budget must stop inside the loop")
```

## Example

```python
pages = {
    None: CursorPage((1, 2), ""),
    "": CursorPage((), "tail"),
    "tail": CursorPage((3, 4, 5), "unused"),
}
fetch_calls: list[str | None] = []
predicate_calls: list[int] = []


def fetch(cursor: str | None) -> CursorPage[int, str]:
    fetch_calls.append(cursor)
    return pages[cursor]


def is_even(value: int) -> bool:
    predicate_calls.append(value)
    return value % 2 == 0


limited = collect_matching_cursor_pages(
    fetch,
    is_even,
    limit=2,
    page_budget=5,
)

budget_fetches: list[str | None] = []


def fetch_for_budget(cursor: str | None) -> CursorPage[int, str]:
    budget_fetches.append(cursor)
    return pages[cursor]


budgeted = collect_matching_cursor_pages(
    fetch_for_budget,
    lambda value: value > 10,
    limit=3,
    page_budget=1,
)


def fetch_exhausted(_cursor: str | None) -> CursorPage[str, str]:
    return CursorPage(("only",), None)


exhausted = collect_matching_cursor_pages(
    fetch_exhausted,
    lambda _item: True,
    limit=2,
    page_budget=2,
)

zero_fetches: list[str | None] = []
zero_predicates: list[int] = []


def should_not_fetch(cursor: str | None) -> CursorPage[int, str]:
    zero_fetches.append(cursor)
    raise AssertionError("zero limit must not fetch")


def should_not_filter(item: int) -> bool:
    zero_predicates.append(item)
    raise AssertionError("zero limit must not filter")


zero = collect_matching_cursor_pages(
    should_not_fetch,
    should_not_filter,
    limit=0,
    page_budget=1,
)

assert (
    limited,
    fetch_calls,
    predicate_calls,
    budgeted,
    budget_fetches,
    exhausted,
    zero,
    zero_fetches,
    zero_predicates,
) == (
    MatchingCollection((2, 4), CollectionStop.LIMIT, None, 3),
    [None, "", "tail"],
    [1, 2, 3, 4],
    MatchingCollection((), CollectionStop.BUDGET, "", 1),
    [None],
    MatchingCollection(("only",), CollectionStop.EXHAUSTED, None, 1),
    MatchingCollection((), CollectionStop.LIMIT, None, 0),
    [],
    [],
)
```

## Trade-offs and Limitations

Cycle detection retains one cursor per fetched page, using `O(p)` memory for
`p` pages. The page budget bounds callback count, not response size, request
duration, or predicate cost, and each page is already materialized as a tuple.
Items are preserved in encounter order without deduplication. Fetch and
predicate exceptions propagate immediately, so retries require an explicit
idempotency and transport policy outside this function.

Stopping at the match limit may leave unvisited items in the current page, so
that result intentionally has no resumable cursor. A next cursor is returned
only for `BUDGET`, after every item in the budget's final page was examined. An
API that requires item-level resume needs a more precise continuation token or
caller-owned buffering. Concurrent prefetch and offset pagination have
different ordering and consistency contracts.

## Related Snippets

<!-- catalog:related:start -->
- [Iterate Cursor-Paginated Results Lazily](iterate-cursor-paginated-results-lazily.md)
<!-- catalog:related:end -->
