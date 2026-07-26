---
title: "Collect Expected Parse Failures Without Stopping a Batch"
snippet_type: pattern
use_cases:
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - normalize-optional-csv-columns-in-a-single-pass.md
  - parse-pipe-delimited-tables-with-continuation-rows.md
  - ../python-language/keep-exception-handlers-narrow-with-try-else.md
---

# Collect Expected Parse Failures Without Stopping a Batch

## Idea and Problem

Represent expected input failures as typed values so one malformed item does not discard successful results from the same batch.

The parser returns either `Parsed(value)` or `Rejected(reason)`. The collector
keeps successes and failures in their respective input order, records the
zero-based position of each rejection, and does not retain the raw rejected
item. Exceptions and unsupported return values remain programming or system
failures and stop collection instead of being mislabeled as bad data.

## When to Use

Use this pattern when inputs are independent, partial success is useful, and
the caller needs all ordinary validation failures in one pass. It fits imports,
line-oriented decoding, and form processing when the parser can state safe,
actionable reasons. Prefer exceptions when any invalid item must abort the
whole operation or when partial results cannot be used safely.

## Implementation

```python
from collections.abc import Callable, Iterable
from dataclasses import dataclass
from typing import Generic, TypeVar


InputT = TypeVar("InputT")
OutputT = TypeVar("OutputT")


@dataclass(frozen=True, slots=True)
class Parsed(Generic[OutputT]):
    value: OutputT


@dataclass(frozen=True, slots=True)
class Rejected:
    reason: str

    def __post_init__(self) -> None:
        if not isinstance(self.reason, str) or not self.reason.strip():
            raise ValueError("a rejection reason must be non-empty text")


@dataclass(frozen=True, slots=True)
class ParseFailure:
    input_index: int
    reason: str


@dataclass(frozen=True, slots=True)
class ParseReport(Generic[OutputT]):
    values: tuple[OutputT, ...]
    failures: tuple[ParseFailure, ...]


def collect_parse_results(
    items: Iterable[InputT],
    parser: Callable[[InputT], Parsed[OutputT] | Rejected],
) -> ParseReport[OutputT]:
    if not callable(parser):
        raise TypeError("parser must be callable")

    values: list[OutputT] = []
    failures: list[ParseFailure] = []
    for input_index, item in enumerate(items):
        outcome = parser(item)
        if isinstance(outcome, Parsed):
            values.append(outcome.value)
        elif isinstance(outcome, Rejected):
            failures.append(ParseFailure(input_index, outcome.reason))
        else:
            raise TypeError("parser must return Parsed or Rejected")

    return ParseReport(tuple(values), tuple(failures))
```

## Example

```python
parser_calls: list[str] = []


def parse_count(text: str) -> Parsed[int] | Rejected:
    parser_calls.append(text)
    try:
        value = int(text)
    except ValueError:
        return Rejected("expected an integer")
    if value < 0:
        return Rejected("expected a non-negative value")
    return Parsed(value)


report = collect_parse_results(["4", "bad", "2", "-1"], parse_count)

try:
    collect_parse_results(["input"], lambda _item: None)  # type: ignore[arg-type]
except TypeError:
    invalid_result_rejected = True
else:
    invalid_result_rejected = False

assert (
    report,
    parser_calls,
    invalid_result_rejected,
) == (
    ParseReport(
        values=(4, 2),
        failures=(
            ParseFailure(1, "expected an integer"),
            ParseFailure(3, "expected a non-negative value"),
        ),
    ),
    ["4", "bad", "2", "-1"],
    True,
)
```

## Trade-offs and Limitations

The report retains every success and failure, so memory grows with the input;
stream separate outcomes when the batch itself is too large. The reason text
is safe only if the parser keeps secrets and raw values out of it. A parser
exception aborts after any earlier callbacks have run, and a malformed result
is detected only when reached. This pattern separates expected data errors
from exceptional failures; it does not provide rollback, retries, logging, or
automatic error recovery.

## Related Snippets

<!-- catalog:related:start -->
- [Normalize Optional CSV Columns in a Single Pass](normalize-optional-csv-columns-in-a-single-pass.md)
- [Parse Pipe-Delimited Tables with Continuation Rows](parse-pipe-delimited-tables-with-continuation-rows.md)
- [Keep Exception Handlers Narrow with try/else](../python-language/keep-exception-handlers-narrow-with-try-else.md)
<!-- catalog:related:end -->
