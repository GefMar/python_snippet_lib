---
title: "Classify Progress from Complete Bounded Counter Snapshots"
snippet_type: recipe
use_cases:
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - measure-cache-hit-ratios-from-monotonic-counter-snapshots.md
  - classify-required-health-stamps-by-freshness.md
  - report-partition-offsets-behind-a-fixed-checkpoint.md
---

# Classify Progress from Complete Bounded Counter Snapshots

## Idea and Problem

Parse two complete bounded counter snapshots and classify every required absolute or interval-progress rule without coupling the decision to metric transport.

Each snapshot uses one anchored line grammar and must contain every declared
counter exactly once. Absolute rules can decide from the first observation,
while delta rules report an explicit warming state until a baseline exists.
Counter decreases remain visible as resets instead of becoming misleading
negative progress.

## When to Use

Use this recipe when a caller can fetch a small coherent text snapshot from the
same cumulative counters before and after a known interval. It fits diagnostic
checks that need structured evidence rather than one opaque Boolean result.
Capture both samples with the same monotonic clock and choose thresholds that
match the expected workload.

Parse a real exposition format before this boundary when it includes labels,
comments, floats, or escaping. Do not use counter progress as universal
liveness: an idle but healthy producer may have no delta, and a partial scrape
must never be treated as a complete snapshot.

## Implementation

```python
import math
import re
from dataclasses import dataclass
from enum import StrEnum
from typing import TypeAlias

_COUNTER_NAME = re.compile(r"[a-z][a-z0-9_]{0,31}", re.ASCII).fullmatch
_COUNTER_LINE = re.compile(
    r"(?P<name>[a-z][a-z0-9_]{0,31}) (?P<value>0|[1-9][0-9]{0,19})\Z",
    re.ASCII,
).fullmatch
_MAX_COUNTER = (1 << 64) - 1
_MAX_RULES = 16
_MAX_TEXT_BYTES = 4 * 1_024
_MAX_LINE_BYTES = 96
_MAX_CLOCK_MAGNITUDE = 1_000_000_000_000_000.0


class RuleKind(StrEnum):
    ABSOLUTE = "absolute"
    DELTA = "delta"


class CounterState(StrEnum):
    SATISFIED = "satisfied"
    BELOW_MINIMUM = "below-minimum"
    INSUFFICIENT_DELTA = "insufficient-delta"
    WARMING = "warming"
    RESET = "reset"


class ProgressState(StrEnum):
    READY = "ready"
    NOT_READY = "not-ready"
    WARMING = "warming"
    RESET = "reset"


@dataclass(frozen=True, slots=True)
class AtLeastCounter:
    name: str
    minimum: int


@dataclass(frozen=True, slots=True)
class IncreaseCounterBy:
    name: str
    minimum_delta: int


CounterRule: TypeAlias = AtLeastCounter | IncreaseCounterBy


@dataclass(frozen=True, slots=True)
class CounterValue:
    name: str
    value: int


@dataclass(frozen=True, slots=True)
class CounterSnapshot:
    captured_at: float
    counters: tuple[CounterValue, ...]


@dataclass(frozen=True, slots=True)
class CounterAssessment:
    name: str
    rule_kind: RuleKind
    state: CounterState
    previous: int | None
    current: int
    delta: int | None
    required: int


@dataclass(frozen=True, slots=True)
class ProgressOutcome:
    state: ProgressState
    elapsed: float | None
    assessments: tuple[CounterAssessment, ...]


def _counter(value: object, *, name: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not 0 <= value <= _MAX_COUNTER:
        raise ValueError(f"{name} is outside the unsigned 64-bit range")
    return value


def _clock(value: object, *, name: str) -> float:
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError(f"{name} must be numeric")
    try:
        normalized = float(value)
    except (OverflowError, TypeError, ValueError) as error:
        raise ValueError(f"{name} must fit in a finite float") from error
    if not math.isfinite(normalized) or abs(normalized) > _MAX_CLOCK_MAGNITUDE:
        raise ValueError(f"{name} is outside the supported finite range")
    return normalized


def _validated_rules(value: object) -> tuple[CounterRule, ...]:
    if type(value) is not tuple:
        raise TypeError("rules must be an exact tuple")
    if not 1 <= len(value) <= _MAX_RULES:
        raise ValueError("rule count is outside the supported range")

    validated: list[CounterRule] = []
    names: set[str] = set()
    for rule in value:
        if type(rule) not in (AtLeastCounter, IncreaseCounterBy):
            raise TypeError("rules must contain closed counter rule values")
        if type(rule.name) is not str or _COUNTER_NAME(rule.name) is None:
            raise ValueError("counter names must be conservative ASCII identifiers")
        if rule.name in names:
            raise ValueError("counter rule names must be unique")
        names.add(rule.name)

        if type(rule) is AtLeastCounter:
            validated.append(AtLeastCounter(rule.name, _counter(rule.minimum, name="minimum")))
        else:
            minimum_delta = _counter(rule.minimum_delta, name="minimum_delta")
            if minimum_delta == 0:
                raise ValueError("minimum_delta must be positive")
            validated.append(IncreaseCounterBy(rule.name, minimum_delta))
    return tuple(validated)


def capture_complete_counter_snapshot(
    rules: tuple[CounterRule, ...],
    text: str,
    *,
    captured_at: float,
) -> CounterSnapshot:
    validated_rules = _validated_rules(rules)
    if type(text) is not str:
        raise TypeError("counter snapshot must be an exact string")
    try:
        encoded_size = len(text.encode("utf-8"))
    except UnicodeEncodeError as error:
        raise ValueError("counter snapshot must be valid Unicode text") from error
    if not 1 <= encoded_size <= _MAX_TEXT_BYTES:
        raise ValueError("counter snapshot byte size is outside the supported range")

    body = text[:-1] if text.endswith("\n") else text
    if not body or body.endswith("\n"):
        raise ValueError("counter snapshot must not contain blank lines")
    lines = body.split("\n")
    if len(lines) != len(validated_rules):
        raise ValueError("counter snapshot must contain exactly one line per rule")

    observed: dict[str, int] = {}
    for line in lines:
        if len(line.encode("utf-8")) > _MAX_LINE_BYTES:
            raise ValueError("counter snapshot contains an oversized line")
        match = _COUNTER_LINE(line)
        if match is None:
            raise ValueError("counter lines must match '<name> <unsigned-integer>' exactly")
        name = match["name"]
        if name in observed:
            raise ValueError("counter names must appear exactly once")
        observed[name] = _counter(int(match["value"]), name="counter value")

    required_names = {rule.name for rule in validated_rules}
    if set(observed) != required_names:
        raise ValueError("counter snapshot must contain the exact required name set")
    return CounterSnapshot(
        captured_at=_clock(captured_at, name="captured_at"),
        counters=tuple(CounterValue(rule.name, observed[rule.name]) for rule in validated_rules),
    )


def _validated_snapshot(
    value: object,
    rules: tuple[CounterRule, ...],
    *,
    name: str,
) -> CounterSnapshot:
    if type(value) is not CounterSnapshot:
        raise TypeError(f"{name} must be an exact CounterSnapshot")
    if type(value.counters) is not tuple or len(value.counters) != len(rules):
        raise ValueError(f"{name} must contain exactly one counter per rule")

    counters: list[CounterValue] = []
    for rule, counter in zip(rules, value.counters, strict=True):
        if type(counter) is not CounterValue or counter.name != rule.name:
            raise ValueError(f"{name} counter order must match the rule order")
        counters.append(CounterValue(counter.name, _counter(counter.value, name="counter value")))
    return CounterSnapshot(_clock(value.captured_at, name=f"{name}.captured_at"), tuple(counters))


def classify_counter_progress(
    rules: tuple[CounterRule, ...],
    earlier: CounterSnapshot | None,
    later: CounterSnapshot,
) -> ProgressOutcome:
    validated_rules = _validated_rules(rules)
    current = _validated_snapshot(later, validated_rules, name="later")
    previous = (
        None if earlier is None else _validated_snapshot(earlier, validated_rules, name="earlier")
    )
    if previous is None:
        elapsed = None
    else:
        elapsed = current.captured_at - previous.captured_at
        if not math.isfinite(elapsed) or elapsed <= 0.0:
            raise ValueError("later must follow earlier on the same monotonic clock")

    previous_values = (
        None if previous is None else {counter.name: counter.value for counter in previous.counters}
    )
    current_values = {counter.name: counter.value for counter in current.counters}
    assessments: list[CounterAssessment] = []
    for rule in validated_rules:
        current_value = current_values[rule.name]
        previous_value = None if previous_values is None else previous_values[rule.name]
        delta = None if previous_value is None else current_value - previous_value

        if previous_value is not None and current_value < previous_value:
            state = CounterState.RESET
        elif type(rule) is AtLeastCounter:
            state = (
                CounterState.SATISFIED
                if current_value >= rule.minimum
                else CounterState.BELOW_MINIMUM
            )
        elif previous_value is None:
            state = CounterState.WARMING
        else:
            state = (
                CounterState.SATISFIED
                if delta is not None and delta >= rule.minimum_delta
                else CounterState.INSUFFICIENT_DELTA
            )

        assessments.append(
            CounterAssessment(
                name=rule.name,
                rule_kind=(RuleKind.ABSOLUTE if type(rule) is AtLeastCounter else RuleKind.DELTA),
                state=state,
                previous=previous_value,
                current=current_value,
                delta=delta,
                required=(rule.minimum if type(rule) is AtLeastCounter else rule.minimum_delta),
            )
        )

    states = {assessment.state for assessment in assessments}
    if CounterState.RESET in states:
        overall = ProgressState.RESET
    elif states & {CounterState.BELOW_MINIMUM, CounterState.INSUFFICIENT_DELTA}:
        overall = ProgressState.NOT_READY
    elif CounterState.WARMING in states:
        overall = ProgressState.WARMING
    else:
        overall = ProgressState.READY
    return ProgressOutcome(overall, elapsed, tuple(assessments))
```

## Example

```python
rules = (
    AtLeastCounter("workers_ready", 2),
    IncreaseCounterBy("records_processed", 3),
)
baseline = capture_complete_counter_snapshot(
    rules,
    "records_processed 10\nworkers_ready 2\n",
    captured_at=100,
)
current = capture_complete_counter_snapshot(
    rules,
    "workers_ready 2\nrecords_processed 14\n",
    captured_at=105,
)
after_reset = capture_complete_counter_snapshot(
    rules,
    "workers_ready 2\nrecords_processed 1\n",
    captured_at=110,
)

warming = classify_counter_progress(rules, None, baseline)
ready = classify_counter_progress(rules, baseline, current)
reset = classify_counter_progress(rules, current, after_reset)

try:
    capture_complete_counter_snapshot(
        rules,
        "workers_ready 2\nworkers_ready 3\n",
        captured_at=120,
    )
except ValueError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

assert (
    warming.state,
    ready.state,
    ready.elapsed,
    tuple(assessment.state for assessment in ready.assessments),
    reset.state,
    duplicate_rejected,
) == (
    ProgressState.WARMING,
    ProgressState.READY,
    5.0,
    (CounterState.SATISFIED, CounterState.SATISFIED),
    ProgressState.RESET,
    True,
)
```

## Trade-offs and Limitations

Parsing and classification use linear time and memory for at most 16 rules and
a 4 KiB snapshot. The exact grammar accepts only one ASCII counter name, one
space, and one canonical unsigned integer per line, with an optional single
terminal newline. Input order is irrelevant, but missing, extra, duplicate,
blank, malformed, negative, or oversized values reject the complete snapshot.

The caller owns fetching, authentication, timeouts, coherent capture, and the
monotonic timestamps; this code performs no HTTP or partial-state update. A
reset outcome requires a new baseline, while a warming outcome means a delta
rule has only its first complete sample. Counter movement is workload evidence,
not proof of dependency health, and fixed thresholds do not replace alerting,
rate calculation, labels, aggregation, or an exporter-aware parser.

## Related Snippets

<!-- catalog:related:start -->
- [Measure Cache Hit Ratios from Monotonic Counter Snapshots](measure-cache-hit-ratios-from-monotonic-counter-snapshots.md)
- [Classify Required Health Stamps by Freshness](classify-required-health-stamps-by-freshness.md)
- [Report Partition Offsets Behind a Fixed Checkpoint](report-partition-offsets-behind-a-fixed-checkpoint.md)
<!-- catalog:related:end -->
