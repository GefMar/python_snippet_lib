---
title: "Classify a Naive Local Datetime Against an Explicit ZoneInfo Transition"
snippet_type: algorithm
use_cases:
  - configuration
  - interoperability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-closed-rfc-3339-timestamp-subset-into-an-aware-datetime.md
  - convert-a-weekday-bitmask-to-a-canonical-cron-schedule.md
---

# Classify a Naive Local Datetime Against an Explicit ZoneInfo Transition

## Idea and Problem

Classify one naive local datetime as unique, ambiguous, or nonexistent under the transition rules of one explicit ZoneInfo.

Both possible `fold` values are attached in turn, converted through UTC, and
converted back to the same zone. A candidate is retained only when the
round-trip recovers the original wall fields. Deduplicating by UTC instant
collapses the two equivalent folds of an ordinary local time while preserving
the two distinct instants of an ambiguous time.

## When to Use

Use this algorithm after a boundary has parsed calendar and clock fields but
before it stores or schedules the resulting instant. The caller must already
have selected one exact IANA zone and must treat ambiguity as an explicit
decision rather than silently accepting the default fold.

Use an explicit offset parser when the input already contains an offset. Use a
separate application policy to reject ambiguity, choose one candidate, ask a
user, or move a nonexistent time. This function classifies the wall time only;
it never makes that policy decision.

## Implementation

```python
from dataclasses import dataclass
from datetime import UTC, datetime
from enum import StrEnum
from zoneinfo import ZoneInfo


class LocalDatetimeKind(StrEnum):
    UNIQUE = "unique"
    AMBIGUOUS = "ambiguous"
    NONEXISTENT = "nonexistent"


@dataclass(frozen=True, slots=True)
class LocalDatetimeClassification:
    kind: LocalDatetimeKind
    candidates: tuple[datetime, ...]


def classify_local_datetime(
    local: datetime,
    zone: ZoneInfo,
) -> LocalDatetimeClassification:
    if type(local) is not datetime:
        raise TypeError("local must be an exact datetime")
    if local.tzinfo is not None:
        raise ValueError("local must be naive")
    if local.fold != 0:
        raise ValueError("a naive input must not preselect a fold")
    if not 2 <= local.year <= 9998:
        raise ValueError("local year is outside the safe round-trip range")
    if type(zone) is not ZoneInfo:
        raise TypeError("zone must be an exact ZoneInfo")

    candidates_by_instant: dict[datetime, datetime] = {}
    for fold in (0, 1):
        tentative = local.replace(tzinfo=zone, fold=fold)
        instant = tentative.astimezone(UTC)
        round_tripped = instant.astimezone(zone)
        recovered_wall = round_tripped.replace(tzinfo=None, fold=0)
        if recovered_wall == local:
            candidates_by_instant.setdefault(instant, round_tripped)

    candidates = tuple(
        candidates_by_instant[instant]
        for instant in sorted(candidates_by_instant)
    )
    if not candidates:
        kind = LocalDatetimeKind.NONEXISTENT
    elif len(candidates) == 1:
        kind = LocalDatetimeKind.UNIQUE
    else:
        kind = LocalDatetimeKind.AMBIGUOUS
    return LocalDatetimeClassification(kind, candidates)
```

## Example

```python
new_york = ZoneInfo("America/New_York")

unique = classify_local_datetime(
    datetime(2024, 1, 15, 12, 0),
    new_york,
)
ambiguous = classify_local_datetime(
    datetime(2024, 11, 3, 1, 30),
    new_york,
)
nonexistent = classify_local_datetime(
    datetime(2024, 3, 10, 2, 30),
    new_york,
)

assert (
    unique.kind,
    tuple(value.astimezone(UTC) for value in unique.candidates),
    ambiguous.kind,
    tuple(value.fold for value in ambiguous.candidates),
    tuple(value.astimezone(UTC) for value in ambiguous.candidates),
    nonexistent,
) == (
    LocalDatetimeKind.UNIQUE,
    (datetime(2024, 1, 15, 17, 0, tzinfo=UTC),),
    LocalDatetimeKind.AMBIGUOUS,
    (0, 1),
    (
        datetime(2024, 11, 3, 5, 30, tzinfo=UTC),
        datetime(2024, 11, 3, 6, 30, tzinfo=UTC),
    ),
    LocalDatetimeClassification(LocalDatetimeKind.NONEXISTENT, ()),
)
```

## Trade-offs and Limitations

The year range excludes the two extreme `datetime` years so conversion across
an offset cannot overflow the representable calendar. At most two tentative
values and two retained values are created, and candidates are returned as
aware local datetimes ordered by their UTC instants.

`ZoneInfo` uses the IANA data available in the running environment; the Python
standard library does not bundle that database on every platform. Deployments
that need portable availability must provide `tzdata`, and persisted decisions
can differ when timezone rules are updated. The function does not validate a
zone key, load timezone data, parse text, model leap seconds, select a fold, or
guarantee that future political timezone rules will remain unchanged.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Closed RFC 3339 Timestamp Subset into an Aware Datetime](parse-a-closed-rfc-3339-timestamp-subset-into-an-aware-datetime.md)
- [Convert a Weekday Bitmask to a Canonical Cron Schedule](convert-a-weekday-bitmask-to-a-canonical-cron-schedule.md)
<!-- catalog:related:end -->
