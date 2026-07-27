---
title: "Expand a Bounded EUI-48 Address Interval"
snippet_type: algorithm
use_cases:
  - data-transformation
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - expand-bounded-nested-brace-alternatives.md
  - parse-explicit-decimal-and-binary-byte-sizes.md
  - ../algorithms-data-structures/cover-a-half-open-integer-range-with-dyadic-intervals.md
---

# Expand a Bounded EUI-48 Address Interval

## Idea and Problem

Parse two explicit EUI-48 endpoints, validate the inclusive interval size before allocation, and return every address in one canonical form.

Accepting only twelve hexadecimal digits or six colon-separated octets keeps
the input language inspectable. Converting endpoints to 48-bit integers makes
ordering and interval length exact, while formatting every result as lowercase
colon notation gives callers one stable representation.

## When to Use

Use this algorithm when a small, complete EUI-48 interval is genuinely needed
in memory and the caller can set an upper bound no larger than the fixed safety
ceiling. Start and end are separate values, the interval is inclusive, and a
singleton is valid. Reject a reversed interval rather than guessing intent.

Use an iterator or numeric interval representation when the complete result is
not required. Keep address assignment, device discovery, and network policy in
separate components; this function only parses, expands, and formats values.

## Implementation

```python
import re


_COMPACT_EUI_48 = re.compile(r"[0-9A-Fa-f]{12}", re.ASCII)
_COLON_EUI_48 = re.compile(r"(?:[0-9A-Fa-f]{2}:){5}[0-9A-Fa-f]{2}", re.ASCII)
_EUI_48_LIMIT = 1 << 48
_MAX_RETURNED_ADDRESSES = 65_536


def _parse_eui_48(address: str) -> int:
    if type(address) is not str:
        raise TypeError("an EUI-48 address must be text")
    if _COMPACT_EUI_48.fullmatch(address) is not None:
        hexadecimal = address
    elif _COLON_EUI_48.fullmatch(address) is not None:
        hexadecimal = address.replace(":", "")
    else:
        raise ValueError("an address must use compact or six-octet colon notation")

    value = int(hexadecimal, 16)
    if not 0 <= value < _EUI_48_LIMIT:
        raise ValueError("an address is outside the 48-bit range")
    return value


def _format_eui_48(value: int) -> str:
    if not 0 <= value < _EUI_48_LIMIT:
        raise ValueError("an address is outside the 48-bit range")
    hexadecimal = f"{value:012x}"
    return ":".join(
        hexadecimal[index : index + 2] for index in range(0, 12, 2)
    )


def expand_eui_48_interval(
    start: str,
    end: str,
    *,
    max_addresses: int,
) -> tuple[str, ...]:
    if isinstance(max_addresses, bool) or not isinstance(max_addresses, int):
        raise TypeError("max_addresses must be an integer")
    if not 1 <= max_addresses <= _MAX_RETURNED_ADDRESSES:
        raise ValueError("max_addresses is outside the supported range")

    first = _parse_eui_48(start)
    last = _parse_eui_48(end)
    if first > last:
        raise ValueError("the EUI-48 interval must not be reversed")

    address_count = last - first + 1
    if address_count > max_addresses:
        raise ValueError("the EUI-48 interval exceeds max_addresses")
    return tuple(_format_eui_48(value) for value in range(first, last + 1))
```

## Example

```python
expanded = expand_eui_48_interval(
    "0A0B0C0D0EFE",
    "0a:0b:0c:0d:0f:00",
    max_addresses=3,
)
maximum_singleton = expand_eui_48_interval(
    "FF:FF:FF:FF:FF:FF",
    "ffffffffffff",
    max_addresses=1,
)

try:
    expand_eui_48_interval("000000000000", "000000000002", max_addresses=2)
except ValueError:
    cap_checked = True
else:
    cap_checked = False

try:
    expand_eui_48_interval("00:00:00:00:00:02", "000000000001", max_addresses=2)
except ValueError:
    reversed_rejected = True
else:
    reversed_rejected = False

malformed = ("00-00-00-00-00-00", "00000000000", "gg0000000000")
malformed_rejected = []
for address in malformed:
    try:
        expand_eui_48_interval(address, "000000000000", max_addresses=1)
    except ValueError:
        malformed_rejected.append(address)

try:
    expand_eui_48_interval(
        "000000000000",
        "000000000000",
        max_addresses=65_537,
    )
except ValueError:
    maximum_cap_enforced = True
else:
    maximum_cap_enforced = False

assert (
    expanded,
    maximum_singleton,
    cap_checked,
    reversed_rejected,
    tuple(malformed_rejected),
    maximum_cap_enforced,
) == (
    (
        "0a:0b:0c:0d:0e:fe",
        "0a:0b:0c:0d:0e:ff",
        "0a:0b:0c:0d:0f:00",
    ),
    ("ff:ff:ff:ff:ff:ff",),
    True,
    True,
    malformed,
    True,
)
```

## Trade-offs and Limitations

The function returns a complete tuple, so time and memory grow linearly with
the inclusive interval length. Both the caller-provided cap and the fixed
safety ceiling are checked before the tuple is allocated. Canonical output
discards input capitalization and separator style deliberately.

The accepted grammar has no whitespace, hyphens, dot notation, prefixes, or
wildcards. This narrow utility does not support EUI-64 conversion, interpret
OUI ownership, infer multicast or locally administered meaning, discover a
device, or make authentication or authorization claims about an address.

## Related Snippets

<!-- catalog:related:start -->
- [Expand Bounded Nested Brace Alternatives](expand-bounded-nested-brace-alternatives.md)
- [Parse Explicit Decimal and Binary Byte Sizes](parse-explicit-decimal-and-binary-byte-sizes.md)
- [Cover a Half-Open Integer Range with Dyadic Intervals](../algorithms-data-structures/cover-a-half-open-integer-range-with-dyadic-intervals.md)
<!-- catalog:related:end -->
