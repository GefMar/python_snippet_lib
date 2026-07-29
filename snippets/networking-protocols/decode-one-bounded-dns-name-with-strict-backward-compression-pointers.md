---
title: "Decode One Bounded DNS Name with Strict Backward Compression Pointers"
snippet_type: recipe
use_cases:
  - networking
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-canonical-http-origin-key.md
  - parse-a-bounded-host-and-port-with-bracketed-ipv6.md
  - read-and-write-size-capped-varint-frames.md
---

# Decode One Bounded DNS Name with Strict Backward Compression Pointers

## Idea and Problem

Decode one size-bounded DNS wire-format name into raw labels while guarding compressed-pointer traversal.

An ordinary label begins with a length from 1 through 63, and a zero length
terminates the expanded name. A leading `11` bit pair instead introduces a
two-byte pointer. This deliberately strict profile accepts only pointer targets
before the pointer itself, records every parse cursor before reading it, and
rejects a repeated cursor.

The decoder reports both the expanded raw labels and where the original encoded
occurrence ends. The first pointer fixes that offset because the rest of the
name is read from another location in the message.

## When to Use

Use this recipe when a bounded binary parser needs one raw DNS name and its
input profile permits only backward compression pointers. Keeping labels as
bytes separates structural wire validation from later ASCII, escape, case, or
internationalized-name policy.

Use a maintained DNS library for complete messages, unfamiliar extensions,
presentation formatting, IDNA conversion, canonical comparison, record-type
semantics, or interoperability across broader compression behavior. Parse the
surrounding field lengths before choosing the starting offset.

## Implementation

```python
from dataclasses import dataclass

_MAX_DNS_MESSAGE_SIZE = 16_384
_MAX_DNS_NAME_WIRE_SIZE = 255


@dataclass(frozen=True, slots=True)
class DecodedDnsName:
    labels: tuple[bytes, ...]
    next_offset: int


def decode_dns_name_strict(
    message: bytes,
    offset: int,
) -> DecodedDnsName:
    """Decode one raw-label DNS name under a backward-pointer profile."""
    if type(message) is not bytes:
        raise TypeError("message must be exact bytes")
    if not 1 <= len(message) <= _MAX_DNS_MESSAGE_SIZE:
        raise ValueError("message length is outside the supported range")
    if type(offset) is not int:
        raise TypeError("offset must be an exact integer")
    if not 0 <= offset < len(message):
        raise ValueError("offset is outside message")

    labels: list[bytes] = []
    visited: set[int] = set()
    cursor = offset
    next_offset: int | None = None
    expanded_wire_size = 1

    while True:
        if not 0 <= cursor < len(message):
            raise ValueError("name is truncated")
        if cursor in visited:
            raise ValueError("name traversal revisits an offset")
        visited.add(cursor)

        leading = message[cursor]
        label_tag = leading & 0xC0

        if label_tag == 0:
            label_length = leading
            if label_length == 0:
                if next_offset is None:
                    next_offset = cursor + 1
                break

            label_end = cursor + 1 + label_length
            if label_end > len(message):
                raise ValueError("label is truncated")
            expanded_wire_size += 1 + label_length
            if expanded_wire_size > _MAX_DNS_NAME_WIRE_SIZE:
                raise ValueError("expanded name exceeds the DNS wire-size limit")
            labels.append(message[cursor + 1 : label_end])
            cursor = label_end
            continue

        if label_tag == 0xC0:
            if cursor + 1 >= len(message):
                raise ValueError("compression pointer is truncated")
            target = ((leading & 0x3F) << 8) | message[cursor + 1]
            if target >= len(message):
                raise ValueError("compression pointer target is outside message")
            if target >= cursor:
                raise ValueError("compression pointer target is not backward")
            if next_offset is None:
                next_offset = cursor + 2
            cursor = target
            continue

        raise ValueError("reserved DNS label tag is not supported")

    if next_offset is None:
        raise AssertionError("a valid name must establish its following offset")
    return DecodedDnsName(labels=tuple(labels), next_offset=next_offset)
```

## Example

```python
suffix = b"\x07example\x03com\x00"
message = suffix + b"\x03www\xc0\x00"

direct = decode_dns_name_strict(message, 0)
compressed = decode_dns_name_strict(message, len(suffix))
root = decode_dns_name_strict(message, len(suffix) - 1)

assert direct == DecodedDnsName((b"example", b"com"), 13)
assert compressed == DecodedDnsName((b"www", b"example", b"com"), 19)
assert root == DecodedDnsName((), 13)


def structurally_rejected(raw: bytes, offset: int = 0) -> bool:
    try:
        decode_dns_name_strict(raw, offset)
    except ValueError:
        return True
    return False


oversized = b"".join(bytes((63,)) + b"x" * 63 for _ in range(4)) + b"\x00"

assert structurally_rejected(b"\x03ab")
assert structurally_rejected(b"\x40")
assert structurally_rejected(b"\xc0\x02\x00")
assert structurally_rejected(b"\x01a\xc0\x00", 2)
assert structurally_rejected(oversized)
```

## Trade-offs and Limitations

Traversal is bounded by the message and expanded-name limits. It performs
`O(message_size + expanded_size)` byte work in the worst case and keeps up to
`O(message_size)` visited cursor integers, while returned label bytes total at
most 250 bytes. Slicing labels creates independent `bytes` objects.

The 255-byte check counts every expanded label-length octet, every label byte,
and one terminating root octet. Compression-pointer bytes do not count toward
that expanded size. A direct name ends after its root octet; once a pointer is
encountered, `next_offset` remains the byte after that first pointer even if
the referenced suffix contains more labels or pointers.

Backward-only pointers and rejection of the reserved `01` and `10` label tags
are intentional strict-profile decisions. The function does not parse a DNS
header or resource record, prove that a pointer targets a previously recognized
label boundary, preserve the encoded spelling, render escaped text, apply IDNA,
or normalize label case.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Canonical HTTP Origin Key](build-a-canonical-http-origin-key.md)
- [Parse a Bounded Host and Port with Bracketed IPv6](parse-a-bounded-host-and-port-with-bracketed-ipv6.md)
- [Read and Write Size-Capped Varint Frames](read-and-write-size-capped-varint-frames.md)
<!-- catalog:related:end -->
