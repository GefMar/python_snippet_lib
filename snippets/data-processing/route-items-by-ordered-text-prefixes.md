---
title: "Route Items by Ordered Text Prefixes"
snippet_type: recipe
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - collect-expected-parse-failures-without-stopping-a-batch.md
  - project-nested-records-with-explicit-field-paths.md
---

# Route Items by Ordered Text Prefixes

## Idea and Problem

Partition items by the first matching text prefix while preserving priority, input order, and otherwise-unmatched data.

Prefixes are validated before the source is consumed. Each item's extractor is
called once and must return text or `None`; the first matching prefix wins.
Results use immutable tuples and retain unmatched items instead of silently
dropping missing or unexpected keys.

## When to Use

Use ordered-prefix routing when a normalized text key encodes a small,
intentional hierarchy and overlapping prefixes need explicit priority. Put
more specific routes first when they should override broader routes. Prefer an
exact mapping when keys are discrete, or a trie when the route set is large
enough that testing every prefix per item is too expensive.

## Implementation

```python
from collections.abc import Callable, Iterable, Sequence
from dataclasses import dataclass
from typing import Generic, TypeVar


ItemT = TypeVar("ItemT")


@dataclass(frozen=True, slots=True)
class PrefixBucket(Generic[ItemT]):
    prefix: str
    items: tuple[ItemT, ...]


@dataclass(frozen=True, slots=True)
class PrefixRouting(Generic[ItemT]):
    buckets: tuple[PrefixBucket[ItemT], ...]
    unmatched: tuple[ItemT, ...]


def route_by_ordered_prefixes(
    items: Iterable[ItemT],
    *,
    prefixes: Sequence[str],
    text: Callable[[ItemT], str | None],
) -> PrefixRouting[ItemT]:
    if isinstance(prefixes, (str, bytes)) or not isinstance(prefixes, Sequence):
        raise TypeError("prefixes must be a sequence of text values")
    if not callable(text):
        raise TypeError("text must be callable")

    ordered_prefixes = tuple(prefixes)
    if not ordered_prefixes:
        raise ValueError("at least one prefix is required")
    if any(not isinstance(prefix, str) for prefix in ordered_prefixes):
        raise TypeError("every prefix must be text")
    if any(not prefix for prefix in ordered_prefixes):
        raise ValueError("prefixes must not be empty")
    if len(set(ordered_prefixes)) != len(ordered_prefixes):
        raise ValueError("prefixes must be unique")

    routed: list[list[ItemT]] = [[] for _prefix in ordered_prefixes]
    unmatched: list[ItemT] = []
    for item in items:
        value = text(item)
        if value is None:
            unmatched.append(item)
            continue
        if not isinstance(value, str):
            raise TypeError("text must return str or None")

        for route_index, prefix in enumerate(ordered_prefixes):
            if value.startswith(prefix):
                routed[route_index].append(item)
                break
        else:
            unmatched.append(item)

    return PrefixRouting(
        buckets=tuple(
            PrefixBucket(prefix, tuple(bucket))
            for prefix, bucket in zip(ordered_prefixes, routed, strict=True)
        ),
        unmatched=tuple(unmatched),
    )
```

## Example

```python
@dataclass(frozen=True, slots=True)
class Message:
    path: str | None
    payload: str


messages = (
    Message("/api/admin/users", "admin"),
    Message("/api/items", "public"),
    Message(None, "missing"),
    Message("/health", "other"),
)
extractor_calls: list[str] = []


def message_path(message: Message) -> str | None:
    extractor_calls.append(message.payload)
    return message.path


routing = route_by_ordered_prefixes(
    messages,
    prefixes=("/api/admin/", "/api/"),
    text=message_path,
)

source_consumed = False


def source() -> Iterable[Message]:
    global source_consumed
    source_consumed = True
    yield from messages


try:
    route_by_ordered_prefixes(
        source(),
        prefixes=("/api/", "/api/"),
        text=lambda message: message.path,
    )
except ValueError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

assert (
    tuple(
        (bucket.prefix, tuple(item.payload for item in bucket.items))
        for bucket in routing.buckets
    ),
    tuple(item.payload for item in routing.unmatched),
    extractor_calls,
    duplicate_rejected,
    source_consumed,
) == (
    (("/api/admin/", ("admin",)), ("/api/", ("public",))),
    ("missing", "other"),
    ["admin", "public", "missing", "other"],
    True,
    False,
)
```

## Trade-offs and Limitations

For `n` items and `r` routes, the straightforward scan costs `O(n*r)` prefix
tests and materializes all output references. Matching is case-sensitive and
uses Python's exact Unicode code points; normalize before routing only when
the domain defines an equivalence rule. Caller order is part of the contract,
and adding a broader prefix earlier can redirect existing items. An extractor
failure or invalid return aborts after earlier extractor calls. This pure
partitioner does not provide streaming sinks, backpressure, rollback, or
delivery guarantees.

## Related Snippets

<!-- catalog:related:start -->
- [Collect Expected Parse Failures Without Stopping a Batch](collect-expected-parse-failures-without-stopping-a-batch.md)
- [Project Nested Records with Explicit Field Paths](project-nested-records-with-explicit-field-paths.md)
<!-- catalog:related:end -->
