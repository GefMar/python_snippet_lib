---
title: "Count Distinct Active Keys in a Trailing Event Window"
snippet_type: algorithm
use_cases:
  - data-transformation
  - observability
tested_python:
  - "3.14"
dependencies: []
related:
  - fan-out-events-into-bounded-lookback-windows.md
  - ../algorithms-data-structures/count-distinct-values-in-bounded-half-open-ranges-offline-with-a-fenwick-tree.md
---

# Count Distinct Active Keys in a Trailing Event Window

## Idea and Problem

Count unique keys active in a fixed trailing window without retaining every event in the window.

A key is active if its latest event is recent enough. An `OrderedDict` keeps
one timestamp per key, with the least recently seen key first. Expire keys
whose timestamp falls on or before the cutoff, then update recency with all
events at the current tick. The count covers `(tick - width, tick]`, with
one result per distinct tick.

## When to Use

Use this to count recent senders, devices or records in timestamp-ordered
input. Supply `(tick, key)` pairs with integer ticks in nondecreasing order
and string keys. The window width is a positive number of ticks. Results are
generated at event ticks, so monitoring at arbitrary times also needs query
ticks or a time-based trigger.

## Implementation

```python
from collections import OrderedDict
from collections.abc import Iterable, Iterator
from itertools import groupby
from operator import itemgetter


def _ordered_events(events: Iterable[tuple[int, str]]) -> Iterator[tuple[int, str]]:
    previous_tick: int | None = None
    for tick, key in events:
        if type(tick) is not int or type(key) is not str:
            raise TypeError("events require integer ticks and string keys")
        if previous_tick is not None and tick < previous_tick:
            raise ValueError("event ticks must be nondecreasing")
        previous_tick = tick
        yield tick, key


def active_key_counts(events: Iterable[tuple[int, str]], width: int) -> Iterator[tuple[int, int]]:
    """Yield (tick, distinct active keys) after consuming each tick's events."""
    if type(width) is not int or width <= 0:
        raise ValueError("width must be a positive integer")

    latest: OrderedDict[str, int] = OrderedDict()
    for tick, same_tick in groupby(_ordered_events(events), key=itemgetter(0)):
        cutoff = tick - width
        while latest and next(iter(latest.values())) <= cutoff:
            latest.popitem(last=False)

        for _, key in same_tick:
            latest[key] = tick
            latest.move_to_end(key)
        yield tick, len(latest)
```

## Example

```python
readings = (
    (0, "alpha"),
    (1, "bravo"),
    (2, "alpha"),
    (4, "charlie"),
    (4, "delta"),
    (4, "delta"),
    (10, "echo"),
)

# At tick 4, bravo falls on the cutoff. Alpha remains active because its
# latest event is at tick 2. Duplicate events at tick 4 contribute one key.
assert tuple(active_key_counts(iter(readings), width=3)) == (
    (0, 1),
    (1, 2),
    (2, 2),
    (4, 3),
    (10, 1),
)
```

## Trade-offs and Limitations

State grows with the number of distinct active keys. Repeated events refresh
one entry, so a busy key cannot accumulate a queue of its old timestamps.
Each event updates the recency map, and each expiry removes one entry. A
single tick can expire many keys.

The iterator consumes all events at a tick before producing its result, with
one lookahead item to detect the next tick. Invalid order can surface after
earlier counts have been emitted. Buffer or reorder late events upstream if
the producer cannot guarantee timestamp order.

## Related Snippets

<!-- catalog:related:start -->
- [Fan Out Events into Bounded Lookback Windows](fan-out-events-into-bounded-lookback-windows.md)
- [Count Distinct Values in Bounded Half-Open Ranges Offline with a Fenwick Tree](../algorithms-data-structures/count-distinct-values-in-bounded-half-open-ranges-offline-with-a-fenwick-tree.md)
<!-- catalog:related:end -->
