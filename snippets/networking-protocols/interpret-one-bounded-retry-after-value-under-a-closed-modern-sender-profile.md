---
title: "Interpret One Bounded Retry-After Value Under a Closed Modern-Sender Profile"
snippet_type: recipe
use_cases:
  - interoperability
  - networking
  - parsing
  - retry-recovery
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - extract-bounded-singleton-http-headers-with-explicit-rules.md
  - ../configuration-serialization/parse-a-closed-rfc-3339-timestamp-subset-into-an-aware-datetime.md
  - ../reliability-resilience/poll-with-deterministic-capped-backoff-under-one-monotonic-deadline.md
---

# Interpret One Bounded Retry-After Value Under a Closed Modern-Sender Profile

## Idea and Problem

Interpret one already-isolated bounded ASCII Retry-After value as a nonnegative integer delay under a deliberately closed modern-sender profile.

The value is either decimal delay-seconds or one exact canonical IMF-fixdate.
An explicit UTC receipt time anchors the date form. Parsing and reformatting the
date catches obsolete and noncanonical spellings, while exact integer
microsecond arithmetic rounds any positive fractional difference upward. A
caller-provided maximum rejects excessive delays instead of silently clamping
them.

## When to Use

Use this recipe after a trusted HTTP layer has isolated exactly one field value
and the sender contract permits only decimal delay-seconds or canonical
IMF-fixdate. It fits retry planning that needs a plain integer delay before a
separate policy decides whether and how to schedule another attempt.

Capture `received_at` when the field is received and pass an exact aware
`datetime` whose `tzinfo` is `datetime.UTC`. Choose the maximum delay from the
application's own retry budget. Decimal delay-seconds may contain leading
zeroes; date values must match the modern form byte for byte.

## Implementation

```python
from datetime import UTC, datetime, timedelta
from email.utils import format_datetime, parsedate_to_datetime

_MAX_RETRY_AFTER_VALUE_LENGTH = 64
_MICROSECONDS_PER_SECOND = 1_000_000


class RetryAfterValueError(ValueError):
    pass


def _require_utc_receipt(value: object) -> datetime:
    if type(value) is not datetime:
        raise TypeError("received_at must be an exact datetime")
    if value.tzinfo is not UTC:
        raise ValueError("received_at must use datetime.UTC")
    return value


def _require_maximum_delay(value: object) -> int:
    if type(value) is not int:
        raise TypeError("maximum_delay_seconds must be an exact integer")
    if value < 0:
        raise ValueError("maximum_delay_seconds must be nonnegative")
    return value


def _parse_canonical_imf_fixdate(value: str) -> datetime:
    try:
        parsed = parsedate_to_datetime(value)
    except (OverflowError, TypeError, ValueError) as error:
        raise RetryAfterValueError("invalid Retry-After date") from error
    if parsed is None:
        raise RetryAfterValueError("invalid Retry-After date")

    try:
        canonical = format_datetime(parsed, usegmt=True)
    except ValueError as error:
        raise RetryAfterValueError("Retry-After date must use GMT") from error
    if canonical != value:
        raise RetryAfterValueError("Retry-After date is not canonical IMF-fixdate")
    return parsed


def _ceil_positive_seconds(delta: timedelta) -> int:
    if delta <= timedelta(0):
        return 0
    microseconds = (
        delta.days * 86_400 * _MICROSECONDS_PER_SECOND
        + delta.seconds * _MICROSECONDS_PER_SECOND
        + delta.microseconds
    )
    return (microseconds + _MICROSECONDS_PER_SECOND - 1) // _MICROSECONDS_PER_SECOND


def interpret_retry_after_value(
    value: str,
    *,
    received_at: datetime,
    maximum_delay_seconds: int,
) -> int:
    if type(value) is not str:
        raise TypeError("Retry-After value must be an exact string")
    if not 1 <= len(value) <= _MAX_RETRY_AFTER_VALUE_LENGTH:
        raise RetryAfterValueError("Retry-After value length is outside the supported range")
    if not value.isascii():
        raise RetryAfterValueError("Retry-After value must contain only ASCII")

    receipt = _require_utc_receipt(received_at)
    maximum = _require_maximum_delay(maximum_delay_seconds)

    if all("0" <= character <= "9" for character in value):
        delay = int(value)
    else:
        moment = _parse_canonical_imf_fixdate(value)
        delay = _ceil_positive_seconds(moment - receipt)

    if delay > maximum:
        raise RetryAfterValueError("Retry-After delay exceeds maximum_delay_seconds")
    return delay
```

## Example

```python
received_at = datetime(
    1994,
    11,
    6,
    8,
    49,
    35,
    750_000,
    tzinfo=UTC,
)

interpreted = (
    interpret_retry_after_value(
        "120",
        received_at=received_at,
        maximum_delay_seconds=120,
    ),
    interpret_retry_after_value(
        "0005",
        received_at=received_at,
        maximum_delay_seconds=5,
    ),
    interpret_retry_after_value(
        "Sun, 06 Nov 1994 08:49:37 GMT",
        received_at=received_at,
        maximum_delay_seconds=2,
    ),
    interpret_retry_after_value(
        "Sun, 06 Nov 1994 08:49:35 GMT",
        received_at=received_at,
        maximum_delay_seconds=0,
    ),
)

invalid_values = (
    ("121", 120),
    ("+5", 5),
    (" 5", 5),
    ("Sunday, 06-Nov-94 08:49:37 GMT", 10),
    ("Sun Nov  6 08:49:37 1994", 10),
    ("sun, 06 nov 1994 08:49:37 gmt", 10),
    ("Mon, 06 Nov 1994 08:49:37 GMT", 10),
    ("Sun, 06 Nov 1994 08:49:37 +0000", 10),
    ("Sun, 06 Nov 1994 08:49:60 GMT", 10),
    ("Sun, 06 Nov 1994 08:49:38 GMT", 2),
)
rejected = []
for candidate, maximum in invalid_values:
    try:
        interpret_retry_after_value(
            candidate,
            received_at=received_at,
            maximum_delay_seconds=maximum,
        )
    except RetryAfterValueError:
        rejected.append(candidate)

try:
    interpret_retry_after_value(
        "1",
        received_at=received_at,
        maximum_delay_seconds=True,
    )
except TypeError:
    boolean_maximum_rejected = True
else:
    boolean_maximum_rejected = False

try:
    interpret_retry_after_value(
        "1",
        received_at=received_at.replace(tzinfo=None),
        maximum_delay_seconds=1,
    )
except ValueError:
    naive_receipt_rejected = True
else:
    naive_receipt_rejected = False

assert (
    interpreted,
    tuple(rejected),
    boolean_maximum_rejected,
    naive_receipt_rejected,
) == (
    (120, 5, 2, 0),
    tuple(candidate for candidate, _maximum in invalid_values),
    True,
    True,
)
```

## Trade-offs and Limitations

This is a closed modern-sender profile, not a complete HTTP date recipient. The
standard-library parser initially recognizes more date syntax, but the exact
`format_datetime(..., usegmt=True)` round trip admits only canonical
IMF-fixdate. Obsolete HTTP date forms, alternate whitespace or casing, numeric
offsets, and leap-second spellings are rejected. Decimal delay-seconds are
ASCII digits only and may have leading zeroes.

The function interprets one value already isolated by another layer. It does
not parse a header block, resolve duplicate fields, decide whether an operation
is authorized, idempotent, or eligible for retry, or account for clock skew
between sender and recipient. Past dates produce zero, positive fractional
differences round upward, and values above the caller's maximum are rejected.
The function performs no scheduling, sleeping, I/O, or mutation.

## Related Snippets

<!-- catalog:related:start -->
- [Extract Bounded Singleton HTTP Headers with Explicit Rules](extract-bounded-singleton-http-headers-with-explicit-rules.md)
- [Parse a Closed RFC 3339 Timestamp Subset into an Aware Datetime](../configuration-serialization/parse-a-closed-rfc-3339-timestamp-subset-into-an-aware-datetime.md)
- [Poll with Deterministic Capped Backoff Under One Monotonic Deadline](../reliability-resilience/poll-with-deterministic-capped-backoff-under-one-monotonic-deadline.md)
<!-- catalog:related:end -->
