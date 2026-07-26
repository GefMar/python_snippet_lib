---
title: "Keep Exception Handlers Narrow with try/else"
snippet_type: idiom
use_cases:
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related: []
---

# Keep Exception Handlers Narrow with try/else

## Idea and Problem

Put success-only work in a try statement's else clause so the preceding handler covers only the operation expected to fail.

A broad `try` block can accidentally classify a later validation or processing
error as a parsing failure. The `else` clause runs only when the `try` suite
finishes normally, and an exception raised inside `else` is not handled by the
preceding `except` clauses.

## When to Use

Use `try/else` when one small operation has a known failure that should be
translated or recovered from, while subsequent work has a different error
contract. Keep both the `try` suite and the caught exception type narrow. Plain
straight-line code after `except` may be clearer when the success path is very
short or the team is unfamiliar with this form.

## Implementation

```python
class InvalidProbability(ValueError):
    pass


def parse_probability(text: str) -> float:
    try:
        probability = float(text)
    except ValueError as error:
        raise InvalidProbability("probability must be numeric") from error
    else:
        if not 0.0 <= probability <= 1.0:
            raise InvalidProbability("probability must be between zero and one")
        return probability
```

## Example

```python
valid_probability = parse_probability("0.25")

try:
    parse_probability("likely")
except InvalidProbability as error:
    parse_error_has_cause = type(error.__cause__) is ValueError
else:
    parse_error_has_cause = False

try:
    parse_probability("nan")
except InvalidProbability as error:
    validation_error_has_no_cause = error.__cause__ is None
else:
    validation_error_has_no_cause = False

assert (
    valid_probability,
    parse_error_has_cause,
    validation_error_has_no_cause,
) == (
    0.25,
    True,
    True,
)
```

## Trade-offs and Limitations

The construct can be unfamiliar, and it does not replace careful exception
design. Catching `Exception` or wrapping a large parsing operation still hides
unrelated defects. An `else` clause also does not run after `return`, `break`,
or `continue` leaves the `try` suite. If the success-only work does not benefit
from a visually explicit branch, placing it after the complete `try/except`
statement provides the same narrow-handler boundary.

## Related Snippets

<!-- catalog:related:start -->
No related snippets.
<!-- catalog:related:end -->
