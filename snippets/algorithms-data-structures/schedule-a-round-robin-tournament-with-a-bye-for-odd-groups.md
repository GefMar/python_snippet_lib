---
title: "Schedule a Round-Robin Tournament with a Bye for Odd Groups"
snippet_type: algorithm
use_cases:
  - automation
  - data-transformation
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - generate-smooth-weighted-round-robin-slots-for-fixed-consumers.md
  - ../data-processing/interleave-bounded-finite-tuple-sources-round-robin-until-exhausted.md
---

# Schedule a Round-Robin Tournament with a Bye for Odd Groups

## Idea and Problem

Schedule each unordered pair of participants exactly once and give each participant at most one match per round.

Keep the first position fixed and rotate the remaining positions. Pair the
first position with the last, the second with the penultimate, and so on.
For an odd group, add one placeholder whose opponent receives a bye. The
number of rotating positions is then odd. The sum of two rotating positions
advances by two modulo that odd count, so each pair reaches its matching
alignment exactly once. The fixed position sees every other position once.

## When to Use

Use this when a finite group of distinct participant names must complete a
single round-robin tournament. Input order fixes the schedule. An empty group
has no rounds. A one-person group has one round with a bye and no matches.

## Implementation

```python
from collections.abc import Iterable, Iterator
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class TournamentRound:
    matches: tuple[tuple[str, str], ...]
    bye: str | None


def round_robin_rounds(participants: Iterable[str]) -> Iterator[TournamentRound]:
    names = tuple(participants)
    if any(type(name) is not str for name in names):
        raise TypeError("participant names must be strings")
    if len(set(names)) != len(names):
        raise ValueError("participant names must be distinct")
    if not names:
        return

    positions: list[str | None] = list(names)
    if len(positions) % 2:
        positions.append(None)

    for _ in range(len(positions) - 1):
        matches: list[tuple[str, str]] = []
        bye = None
        for index in range(len(positions) // 2):
            first, second = positions[index], positions[-1 - index]
            if first is None:
                bye = second
            elif second is None:
                bye = first
            else:
                matches.append((first, second))
        yield TournamentRound(tuple(matches), bye)
        positions.insert(1, positions.pop())
```

## Example

```python
odd = tuple(round_robin_rounds(iter(("oak", "pine", "elm"))))
even = tuple(round_robin_rounds(("oak", "pine", "elm", "ash")))

assert (odd, tuple(round_.matches for round_ in even), tuple(round_.bye for round_ in even)) == (
    (
        TournamentRound((("pine", "elm"),), "oak"),
        TournamentRound((("oak", "elm"),), "pine"),
        TournamentRound((("oak", "pine"),), "elm"),
    ),
    (
        (("oak", "ash"), ("pine", "elm")),
        (("oak", "elm"), ("ash", "pine")),
        (("oak", "pine"), ("elm", "ash")),
    ),
    (None, None, None),
)
```

## Trade-offs and Limitations

The iterator stores the participant list and one round at a time. A full
schedule has quadratically many matches in the group size. Input validation
happens on first iteration.

Match order is deterministic. This pairing rule leaves venue availability,
rest periods and balancing home or away assignments to a separate scheduler.

## Related Snippets

<!-- catalog:related:start -->
- [Generate Smooth Weighted Round-Robin Slots for Fixed Consumers](generate-smooth-weighted-round-robin-slots-for-fixed-consumers.md)
- [Interleave Bounded Finite Tuple Sources Round-Robin Until Exhausted](../data-processing/interleave-bounded-finite-tuple-sources-round-robin-until-exhausted.md)
<!-- catalog:related:end -->
