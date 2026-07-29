---
title: "Parse a Canonical UUIDv7 with an Explicit Millisecond Window"
snippet_type: recipe
use_cases:
  - interoperability
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - embed-a-small-routing-hint-in-a-random-uuidv8.md
  - parse-a-closed-rfc-3339-timestamp-subset-into-an-aware-datetime.md
  - append-a-fixed-width-crc-check-to-a-human-readable-identifier.md
---

# Parse a Canonical UUIDv7 with an Explicit Millisecond Window

## Idea and Problem

Parse one canonical UUIDv7 and require its embedded Unix-millisecond field to lie inside a caller-supplied inclusive window.

UUID parsers often accept several equivalent textual spellings, while UUIDv7
also exposes a time field that can be mistaken for trusted clock evidence.
This boundary selects one lowercase hyphenated representation, verifies the
version and RFC variant, and compares the embedded 48-bit value only with
explicit integer bounds.

## When to Use

Use this helper at a protocol or storage boundary that declares canonical
UUIDv7 text and a concrete admissible timestamp range. Returning the parsed
`UUID` together with the original text and millisecond value avoids repeated
parsing and makes the checked claims visible to the caller.

Choose a generic UUID parser when alternate spellings or versions are valid.
Authenticate the surrounding message when the timestamp is security-sensitive;
anyone who can choose an identifier can also choose its embedded UUIDv7 time.

## Implementation

```python
import re
from dataclasses import dataclass
from uuid import RFC_4122, UUID

_MAX_UUID7_TIMESTAMP = (1 << 48) - 1
_CANONICAL_UUID = re.compile(
    r"[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-"
    r"[0-9a-f]{4}-[0-9a-f]{12}",
    re.ASCII,
)


@dataclass(frozen=True, slots=True)
class ParsedUUID7:
    text: str
    value: UUID
    timestamp_ms: int


def parse_uuid7_in_window(
    text: str,
    *,
    minimum_ms: int,
    maximum_ms: int,
) -> ParsedUUID7:
    if type(text) is not str:
        raise TypeError("text must be an exact string")
    if type(minimum_ms) is not int or type(maximum_ms) is not int:
        raise TypeError("window bounds must be exact integers")
    if not 0 <= minimum_ms <= maximum_ms <= _MAX_UUID7_TIMESTAMP:
        raise ValueError("millisecond window is outside the 48-bit range")
    if len(text) != 36 or _CANONICAL_UUID.fullmatch(text) is None:
        raise ValueError("UUID text is not canonical lowercase hyphenated form")

    try:
        value = UUID(text)
    except ValueError as error:
        raise ValueError("UUID text cannot be parsed") from error
    if str(value) != text:
        raise ValueError("UUID text does not round-trip canonically")
    if value.variant != RFC_4122 or value.version != 7:
        raise ValueError("UUID must use the RFC variant and version 7")

    timestamp_ms = value.time
    if not minimum_ms <= timestamp_ms <= maximum_ms:
        raise ValueError("UUIDv7 timestamp is outside the admitted window")
    return ParsedUUID7(text, value, timestamp_ms)
```

## Example

```python
rfc_text = "017f22e2-79b0-7cc3-98c4-dc0c0c07398f"
rfc_timestamp_ms = 1_645_557_742_000
parsed = parse_uuid7_in_window(
    rfc_text,
    minimum_ms=rfc_timestamp_ms,
    maximum_ms=rfc_timestamp_ms,
)

invalid_values = (
    rfc_text.upper(),
    "{" + rfc_text + "}",
    rfc_text.replace("-", ""),
    "f81d4fae-7dec-11d0-a765-00a0c91e6bf6",
)
rejected = []
for invalid_value in invalid_values:
    try:
        parse_uuid7_in_window(
            invalid_value,
            minimum_ms=0,
            maximum_ms=_MAX_UUID7_TIMESTAMP,
        )
    except ValueError:
        rejected.append(invalid_value)

assert parsed.text == rfc_text
assert parsed.value.version == 7
assert parsed.value.variant == RFC_4122
assert parsed.timestamp_ms == rfc_timestamp_ms
assert tuple(rejected) == invalid_values
```

## Trade-offs and Limitations

Parsing has fixed time and state because the accepted text and timestamp field
have fixed widths. The explicit window avoids an ambient clock read, but the
caller must choose bounds that match its own protocol. Converting milliseconds
to `datetime`, handling clock skew, and deciding retention or database-index
policy are separate concerns.

This helper does not generate UUIDs and makes no claim about monotonicity,
total ordering, entropy, uniqueness, authenticity, or freshness. UUIDv7 sorts
approximately by its embedded time, but identifiers created within a
millisecond contain additional bits whose generation policy is outside this
parser.

## Related Snippets

<!-- catalog:related:start -->
- [Embed a Small Routing Hint in a Random UUIDv8](embed-a-small-routing-hint-in-a-random-uuidv8.md)
- [Parse a Closed RFC 3339 Timestamp Subset into an Aware Datetime](parse-a-closed-rfc-3339-timestamp-subset-into-an-aware-datetime.md)
- [Append a Fixed-Width CRC Check to a Human-Readable Identifier](append-a-fixed-width-crc-check-to-a-human-readable-identifier.md)
<!-- catalog:related:end -->
