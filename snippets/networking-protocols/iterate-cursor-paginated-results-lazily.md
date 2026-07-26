---
title: "Iterate Cursor-Paginated Results Lazily"
snippet_type: pattern
use_cases:
  - networking
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/batch-any-iterable-lazily-with-itertools-batched.md
  - ../data-processing/batch-items-by-estimated-byte-size.md
---

# Iterate Cursor-Paginated Results Lazily

## Idea and Problem

Hide cursor pagination behind an iterator while keeping page fetches lazy, ordered, and bounded by explicit safety checks.

The callback receives the current cursor and returns a typed page. `None` is
the only exhaustion marker, so falsey cursors such as an empty string remain
valid. The iterator remembers fetched cursors to reject cycles and can stop at
a caller-selected page limit before issuing another request.

## When to Use

Use this adapter for a finite cursor-based API when consumers benefit from an
ordinary iterator and should not materialize every page. It is especially
useful when consumers may stop early. Define cursor equality and page-limit
policy before calling, and let the client callback own transport concerns such
as authentication, timeouts, retries, and response decoding.

## Implementation

```python
from collections.abc import Callable, Hashable, Iterator
from dataclasses import dataclass
from typing import Generic, TypeVar


ItemT = TypeVar("ItemT")
CursorT = TypeVar("CursorT", bound=Hashable)


@dataclass(frozen=True, slots=True)
class CursorPage(Generic[ItemT, CursorT]):
    items: tuple[ItemT, ...]
    next_cursor: CursorT | None


class CursorCycleError(RuntimeError):
    pass


class PageLimitExceeded(RuntimeError):
    pass


def iterate_cursor_pages(
    fetch_page: Callable[[CursorT | None], CursorPage[ItemT, CursorT]],
    *,
    initial_cursor: CursorT | None = None,
    max_pages: int | None = None,
) -> Iterator[ItemT]:
    if not callable(fetch_page):
        raise TypeError("fetch_page must be callable")
    if max_pages is not None:
        if isinstance(max_pages, bool) or not isinstance(max_pages, int):
            raise TypeError("max_pages must be an integer or None")
        if max_pages <= 0:
            raise ValueError("max_pages must be positive")

    cursor = initial_cursor
    seen: set[CursorT | None] = set()
    pages_fetched = 0

    while True:
        try:
            repeated = cursor in seen
            seen.add(cursor)
        except TypeError as error:
            raise TypeError("cursors must be hashable") from error
        if repeated:
            raise CursorCycleError(f"cursor cycle detected at {cursor!r}")

        page = fetch_page(cursor)
        if not isinstance(page, CursorPage):
            raise TypeError("fetch_page must return CursorPage")
        pages_fetched += 1
        yield from page.items

        if page.next_cursor is None:
            return
        if max_pages is not None and pages_fetched >= max_pages:
            raise PageLimitExceeded(f"pagination exceeded {max_pages} pages")
        cursor = page.next_cursor
```

## Example

```python
pages = {
    None: CursorPage(("first",), ""),
    "": CursorPage((), "final"),
    "final": CursorPage(("last",), None),
}
fetch_calls: list[str | None] = []


def fetch(cursor: str | None) -> CursorPage[str, str]:
    fetch_calls.append(cursor)
    return pages[cursor]


items = iterate_cursor_pages(fetch)
calls_before_iteration = tuple(fetch_calls)
first_item = next(items)
calls_after_first_item = tuple(fetch_calls)
remaining_items = tuple(items)

cycle_calls: list[str | None] = []


def fetch_cycle(cursor: str | None) -> CursorPage[int, str]:
    cycle_calls.append(cursor)
    return CursorPage((len(cycle_calls),), "again")


try:
    tuple(iterate_cursor_pages(fetch_cycle))
except CursorCycleError:
    cycle_rejected = True
else:
    cycle_rejected = False

assert (
    calls_before_iteration,
    first_item,
    calls_after_first_item,
    remaining_items,
    fetch_calls,
    cycle_rejected,
    cycle_calls,
) == (
    (),
    "first",
    (None,),
    ("last",),
    [None, "", "final"],
    True,
    [None, "again"],
)
```

## Trade-offs and Limitations

The iterator retains one cursor per fetched page, so cycle detection uses
`O(p)` memory for `p` pages. It keeps only the current `CursorPage`, but that
page is already materialized as a tuple. An error discovered after a page is
yielded cannot retract its items. The page cap raises only when a continuation
would cross the cap. Closing or abandoning the iterator prevents later
fetches, but this helper does not close client sessions or cancel an in-flight
request. Offset pagination, concurrent prefetch, checkpoint persistence,
retries, and rate-limit handling need separate policies.

## Related Snippets

<!-- catalog:related:start -->
- [Batch Any Iterable Lazily with itertools.batched](../python-language/batch-any-iterable-lazily-with-itertools-batched.md)
- [Batch Items by Estimated Byte Size](../data-processing/batch-items-by-estimated-byte-size.md)
<!-- catalog:related:end -->
