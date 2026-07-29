---
title: "Parse and Format a Strict W3C traceparent Version 00 Value"
snippet_type: recipe
use_cases:
  - interoperability
  - networking
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - extract-bounded-singleton-http-headers-with-explicit-rules.md
  - parse-one-bounded-printable-ascii-http-1-field-section-with-exact-crlf-framing.md
  - ../observability-operations/scope-structured-log-fields-with-context-variables.md
---

# Parse and Format a Strict W3C traceparent Version 00 Value

## Idea and Problem

Parse and format the closed version-00 form of a W3C Trace Context traceparent value without accepting ambiguous spellings.

[W3C Trace Context Level 2](https://www.w3.org/TR/trace-context-2/)
defines a 55-character version-00 value containing a trace ID, parent ID, and
flags byte. This profile requires canonical lowercase hexadecimal text,
nonzero identifiers, and zeroes in every reserved flag bit.

The parsed immutable value retains exact identifier bytes. Its properties
expose the sampled and random-trace-ID assertions while leaving trust and
policy decisions to the caller.

## When to Use

Use this parser after an HTTP boundary has already selected exactly one
`traceparent` field value. It fits closed deployments that intentionally
support only the current fixed version-00 grammar and prefer rejecting
noncanonical input.

Use a maintained tracing SDK when future-version forwarding, `tracestate`,
baggage, propagation, ID generation, or sampling policy is required.

## Implementation

```python
from dataclasses import dataclass

_LOWER_HEX = frozenset("0123456789abcdef")
_DEFINED_FLAGS_MASK = 0x03


@dataclass(frozen=True, slots=True)
class TraceParentV00:
    trace_id: bytes
    parent_id: bytes
    flags: int

    @property
    def sampled(self) -> bool:
        return bool(self.flags & 0x01)

    @property
    def random_trace_id(self) -> bool:
        return bool(self.flags & 0x02)


def _decode_identifier(
    name: str,
    encoded: str,
    *,
    byte_length: int,
) -> bytes:
    if len(encoded) != byte_length * 2 or any(
        character not in _LOWER_HEX for character in encoded
    ):
        raise ValueError(f"{name} must be canonical lowercase hexadecimal")
    decoded = bytes.fromhex(encoded)
    if not any(decoded):
        raise ValueError(f"{name} must not be all zero")
    return decoded


def parse_traceparent_v00(value: str) -> TraceParentV00:
    """Parse one strict W3C Trace Context Level 2 version-00 value."""
    if type(value) is not str:
        raise TypeError("value must be an exact string")
    if len(value) != 55:
        raise ValueError("traceparent version 00 must contain 55 characters")
    if value[2] != "-" or value[35] != "-" or value[52] != "-":
        raise ValueError("traceparent separators are malformed")
    if value[:2] != "00":
        raise ValueError("only traceparent version 00 is supported")

    trace_id = _decode_identifier(
        "trace_id",
        value[3:35],
        byte_length=16,
    )
    parent_id = _decode_identifier(
        "parent_id",
        value[36:52],
        byte_length=8,
    )
    encoded_flags = value[53:55]
    if any(character not in _LOWER_HEX for character in encoded_flags):
        raise ValueError("flags must be canonical lowercase hexadecimal")
    flags = int(encoded_flags, 16)
    if flags & ~_DEFINED_FLAGS_MASK:
        raise ValueError("reserved traceparent flag bits must be zero")

    return TraceParentV00(trace_id, parent_id, flags)


def format_traceparent_v00(value: TraceParentV00) -> str:
    """Format one completely revalidated version-00 value."""
    if type(value) is not TraceParentV00:
        raise TypeError("value must be an exact TraceParentV00")
    if type(value.trace_id) is not bytes or len(value.trace_id) != 16:
        raise ValueError("trace_id must contain exactly 16 bytes")
    if not any(value.trace_id):
        raise ValueError("trace_id must not be all zero")
    if type(value.parent_id) is not bytes or len(value.parent_id) != 8:
        raise ValueError("parent_id must contain exactly 8 bytes")
    if not any(value.parent_id):
        raise ValueError("parent_id must not be all zero")
    if type(value.flags) is not int:
        raise TypeError("flags must be an exact integer")
    if not 0 <= value.flags <= _DEFINED_FLAGS_MASK:
        raise ValueError("flags contain a reserved bit")

    return (
        f"00-{value.trace_id.hex()}-{value.parent_id.hex()}-"
        f"{value.flags:02x}"
    )
```

## Example

```python
sampled_header = (
    "00-4bf92f3577b34da6a3ce929d0e0e4736-"
    "00f067aa0ba902b7-01"
)
sampled_parent = parse_traceparent_v00(sampled_header)
assert sampled_parent.sampled
assert not sampled_parent.random_trace_id
assert format_traceparent_v00(sampled_parent) == sampled_header

random_and_sampled = TraceParentV00(
    trace_id=bytes.fromhex("00000000000000000000000000000001"),
    parent_id=bytes.fromhex("0000000000000001"),
    flags=0x03,
)
encoded = format_traceparent_v00(random_and_sampled)
assert parse_traceparent_v00(encoded) == random_and_sampled
assert random_and_sampled.sampled and random_and_sampled.random_trace_id
```

## Trade-offs and Limitations

Parsing and formatting perform fixed bounded work and allocate only the two
identifiers and one short output string. Rejecting reserved bits is deliberate
for this closed Level 2 profile; it is not a general strategy for forwarding
unknown or future Trace Context versions.

The random flag is a sender assertion, not proof that an identifier was
generated with sufficient randomness. Neither flag grants trust or authority.
Inbound identifiers can correlate activity, so retention and logging policies
remain the caller's responsibility.

This snippet does not select among duplicate HTTP fields, trim whitespace,
parse comma-joined values, support `tracestate` or baggage, generate IDs,
enforce sampling, make trust decisions, define propagation policy, or parse
future versions and extensions.

## Related Snippets

<!-- catalog:related:start -->
- [Extract Bounded Singleton HTTP Headers with Explicit Rules](extract-bounded-singleton-http-headers-with-explicit-rules.md)
- [Parse One Bounded Printable-ASCII HTTP/1 Field Section with Exact CRLF Framing](parse-one-bounded-printable-ascii-http-1-field-section-with-exact-crlf-framing.md)
- [Scope Structured Log Fields with Context Variables](../observability-operations/scope-structured-log-fields-with-context-variables.md)
<!-- catalog:related:end -->
