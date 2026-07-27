---
title: "Route Estimated Work by Ordered Source and Size Rules"
snippet_type: recipe
use_cases:
  - data-transformation
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - route-items-by-ordered-text-prefixes.md
  - batch-items-by-estimated-byte-size.md
  - select-one-record-per-key-with-an-explicit-ranking-rule.md
---

# Route Estimated Work by Ordered Source and Size Rules

## Idea and Problem

Choose a destination from a bounded ordered rule table using a canonical source identifier and a non-negative work estimate.

The first rule whose source set and inclusive maximum both match wins. An
empty source set is a wildcard, `None` means that a rule has no size ceiling,
and an explicit fallback handles every item that matches no rule. The result
records only the decision; it does not submit or mutate any work.

## When to Use

Use this recipe when routing policy is deliberately small, local, and ordered,
and a caller can provide one conservative integer estimate before dispatch.
Examples include selecting a processing class or storage lane by origin and
estimated byte count.

Use a policy engine when rules require boolean expressions, dynamic state, or
independent administration. Use a scheduler when destination capacity,
fairness, admission control, retries, or execution guarantees matter.

## Implementation

```python
import re
from collections.abc import Iterable
from dataclasses import dataclass
from itertools import islice


_MAX_RULES = 64
_MAX_SOURCES_PER_RULE = 32
_MAX_IDENTIFIER_LENGTH = 64
_IDENTIFIER = re.compile(r"[a-z][a-z0-9]*(?:-[a-z0-9]+)*\Z")


def _validate_identifier(value: object, *, name: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{name} must be text")
    if len(value) > _MAX_IDENTIFIER_LENGTH or _IDENTIFIER.fullmatch(value) is None:
        raise ValueError(f"{name} must be a canonical lowercase identifier")
    return value


@dataclass(frozen=True, slots=True)
class WorkRouteRule:
    rule_id: str
    sources: tuple[str, ...]
    max_estimate: int | None
    destination: str

    def __post_init__(self) -> None:
        _validate_identifier(self.rule_id, name="rule_id")
        _validate_identifier(self.destination, name="destination")
        if type(self.sources) is not tuple:
            raise TypeError("sources must be a tuple")
        if len(self.sources) > _MAX_SOURCES_PER_RULE:
            raise ValueError("a rule has too many source identifiers")
        for source in self.sources:
            _validate_identifier(source, name="source")
        if len(set(self.sources)) != len(self.sources):
            raise ValueError("source identifiers within a rule must be unique")
        if self.max_estimate is not None:
            if type(self.max_estimate) is not int:
                raise TypeError("max_estimate must be an integer or None")
            if self.max_estimate < 0:
                raise ValueError("max_estimate must be non-negative")


@dataclass(frozen=True, slots=True)
class WorkRouteDecision:
    source: str
    estimate: int
    destination: str
    matched_rule_id: str | None


def route_estimated_work(
    *,
    source: str,
    estimate: int,
    rules: Iterable[WorkRouteRule],
    fallback_destination: str,
) -> WorkRouteDecision:
    canonical_source = _validate_identifier(source, name="source")
    fallback = _validate_identifier(
        fallback_destination,
        name="fallback_destination",
    )
    if type(estimate) is not int:
        raise TypeError("estimate must be an integer")
    if estimate < 0:
        raise ValueError("estimate must be non-negative")

    try:
        ordered_rules = tuple(islice(rules, _MAX_RULES + 1))
    except TypeError as error:
        raise TypeError("rules must be an iterable") from error
    if len(ordered_rules) > _MAX_RULES:
        raise ValueError("rule count exceeds the supported limit")
    if any(not isinstance(rule, WorkRouteRule) for rule in ordered_rules):
        raise TypeError("every rule must be a WorkRouteRule")

    rule_ids = tuple(rule.rule_id for rule in ordered_rules)
    if len(set(rule_ids)) != len(rule_ids):
        raise ValueError("rule identifiers must be unique")

    for rule in ordered_rules:
        source_matches = not rule.sources or canonical_source in rule.sources
        size_matches = rule.max_estimate is None or estimate <= rule.max_estimate
        if source_matches and size_matches:
            return WorkRouteDecision(
                source=canonical_source,
                estimate=estimate,
                destination=rule.destination,
                matched_rule_id=rule.rule_id,
            )

    return WorkRouteDecision(
        source=canonical_source,
        estimate=estimate,
        destination=fallback,
        matched_rule_id=None,
    )
```

## Example

```python
rules = (
    WorkRouteRule("small-api", ("api",), 100, "fast-lane"),
    WorkRouteRule("tiny-anywhere", (), 20, "tiny-lane"),
    WorkRouteRule("remaining-api", ("api",), None, "bulk-lane"),
)

decisions = tuple(
    route_estimated_work(
        source=source,
        estimate=estimate,
        rules=rules,
        fallback_destination="standard-lane",
    )
    for source, estimate in (("api", 100), ("file", 20), ("api", 101), ("file", 21))
)

try:
    route_estimated_work(
        source="api",
        estimate=1,
        rules=(rules[0], rules[0]),
        fallback_destination="standard-lane",
    )
except ValueError:
    duplicate_rule_rejected = True
else:
    duplicate_rule_rejected = False

assert (
    tuple((decision.destination, decision.matched_rule_id) for decision in decisions),
    duplicate_rule_rejected,
) == (
    (
        ("fast-lane", "small-api"),
        ("tiny-lane", "tiny-anywhere"),
        ("bulk-lane", "remaining-api"),
        ("standard-lane", None),
    ),
    True,
)
```

## Trade-offs and Limitations

Routing is `O(r * s)` for `r` bounded rules and at most `s` bounded source
identifiers per rule. The materialized rule tuple and returned decision are
immutable, but the function does not infer whether an estimate is accurate.
The maximum is inclusive, so a value equal to `max_estimate` matches.

Rule order is executable policy. An early wildcard or unbounded rule can
shadow later rules, and this small recipe intentionally does not attempt a
general shadowing analysis. It makes no claims about payload validity, queue
capacity, enqueue success, fairness, persistence, or eventual execution.

## Related Snippets

<!-- catalog:related:start -->
- [Route Items by Ordered Text Prefixes](route-items-by-ordered-text-prefixes.md)
- [Batch Items by Estimated Byte Size](batch-items-by-estimated-byte-size.md)
- [Select One Record per Key with an Explicit Ranking Rule](select-one-record-per-key-with-an-explicit-ranking-rule.md)
<!-- catalog:related:end -->
