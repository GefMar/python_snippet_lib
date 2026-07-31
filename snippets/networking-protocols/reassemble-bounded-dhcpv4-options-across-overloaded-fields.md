---
title: "Reassemble Bounded DHCPv4 Options Across Overloaded Fields"
snippet_type: recipe
use_cases:
  - interoperability
  - networking
  - parsing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - parse-one-bounded-coap-udp-datagram-envelope.md
  - decode-one-bounded-dns-name-with-strict-backward-compression-pointers.md
  - reassemble-one-bounded-ipv4-payload-from-decoded-fragments.md
---

# Reassemble Bounded DHCPv4 Options Across Overloaded Fields

## Idea and Problem

Parse three already isolated DHCPv4 option fields, select overloaded fields from option 52, and concatenate repeated option values in RFC 3396 aggregate order.

[RFC 2132 Section 2](https://www.rfc-editor.org/rfc/rfc2132.html#section-2)
encodes ordinary options as one-byte code, one-byte length, and zero to 255
value bytes; Pad and End are single-byte exceptions. Option 52 in the main
field declares whether the fixed `file` field, the fixed `sname` field, or
both also contain options, as defined by
[RFC 2132 Section 9.3](https://www.rfc-editor.org/rfc/rfc2132.html#section-9.3).

[RFC 3396 Section 5](https://www.rfc-editor.org/rfc/rfc3396.html#section-5)
defines logical `main`, `file`, `sname` order, which differs from the physical
field order in a DHCPv4 packet, and
[Section 7](https://www.rfc-editor.org/rfc/rfc3396.html#section-7) requires
repeated codes to be joined in that encounter order. The result retains each
ordinary code once, ordered by its first occurrence, and exposes the validated
overload value separately.

## When to Use

Use this recipe after another boundary has isolated the bytes after the DHCP
magic cookie as `main_options` and copied the 128-byte `file` and 64-byte
`sname` fields. It is useful for bounded protocol fixtures, offline inspection,
or a narrow adapter that must preserve unknown and repeated option values
without interpreting their registered meanings.

Use a maintained DHCP implementation for a client, server, or relay. Field
reassembly alone does not validate the fixed header, magic cookie, message
type, option-specific length or content rules, authentication, transaction
state, address assignment, relay behavior, or network I/O.

## Implementation

```python
from dataclasses import dataclass

_MIN_MAIN_OPTIONS_BYTES = 1
_MAX_MAIN_OPTIONS_BYTES = 4_096
_FILE_FIELD_BYTES = 128
_SNAME_FIELD_BYTES = 64
_PAD = 0
_OPTION_OVERLOAD = 52
_END = 255
_MAX_FRAGMENTS = 256
_MAX_AGGREGATE_VALUE_BYTES = 2_048


class DhcpOptionFieldError(ValueError):
    """Raised when option fields violate the bounded reassembly profile."""


@dataclass(frozen=True, slots=True)
class DhcpOption:
    code: int
    value: bytes


@dataclass(frozen=True, slots=True)
class DhcpOptionSet:
    overload: int | None
    options: tuple[DhcpOption, ...]


def _parse_option_field(
    field: bytes,
    *,
    field_name: str,
    allow_overload: bool,
    fragment_count: int,
    aggregate_value_bytes: int,
) -> tuple[list[tuple[int, bytes]], int | None, int, int]:
    fragments: list[tuple[int, bytes]] = []
    overload: int | None = None
    offset = 0

    while offset < len(field):
        code = field[offset]
        offset += 1

        if code == _PAD:
            continue
        if code == _END:
            if any(field[offset:]):
                raise DhcpOptionFieldError(
                    f"{field_name} contains a non-Pad byte after End"
                )
            return (
                fragments,
                overload,
                fragment_count,
                aggregate_value_bytes,
            )

        if offset >= len(field):
            raise DhcpOptionFieldError(
                f"{field_name} ends before option {code}'s length"
            )
        value_length = field[offset]
        offset += 1
        value_end = offset + value_length
        if value_end > len(field):
            raise DhcpOptionFieldError(
                f"option {code} crosses the end of {field_name}"
            )
        if fragment_count == _MAX_FRAGMENTS:
            raise DhcpOptionFieldError("option fragment count exceeds 256")
        if value_length > _MAX_AGGREGATE_VALUE_BYTES - aggregate_value_bytes:
            raise DhcpOptionFieldError(
                "aggregate option value bytes exceed 2,048"
            )

        value = field[offset:value_end]
        fragment_count += 1
        aggregate_value_bytes += value_length
        offset = value_end

        if code == _OPTION_OVERLOAD:
            if not allow_overload:
                raise DhcpOptionFieldError(
                    "option 52 may appear only in the main options field"
                )
            if overload is not None:
                raise DhcpOptionFieldError("option 52 must not be repeated")
            if value not in (b"\x01", b"\x02", b"\x03"):
                raise DhcpOptionFieldError(
                    "option 52 must have length one and value 1, 2, or 3"
                )
            overload = value[0]
        else:
            fragments.append((code, value))

    raise DhcpOptionFieldError(f"{field_name} has no End option")


def parse_dhcpv4_option_fields(
    main_options: bytes,
    file_field: bytes,
    sname_field: bytes,
) -> DhcpOptionSet:
    """Parse and reassemble one bounded set of isolated DHCPv4 fields."""
    if type(main_options) is not bytes:
        raise TypeError("main_options must be exact bytes")
    if not _MIN_MAIN_OPTIONS_BYTES <= len(main_options) <= _MAX_MAIN_OPTIONS_BYTES:
        raise DhcpOptionFieldError("main_options length is outside 1..4,096 bytes")
    if type(file_field) is not bytes:
        raise TypeError("file_field must be exact bytes")
    if len(file_field) != _FILE_FIELD_BYTES:
        raise DhcpOptionFieldError("file_field must contain exactly 128 bytes")
    if type(sname_field) is not bytes:
        raise TypeError("sname_field must be exact bytes")
    if len(sname_field) != _SNAME_FIELD_BYTES:
        raise DhcpOptionFieldError("sname_field must contain exactly 64 bytes")

    fragments, overload, fragment_count, aggregate_value_bytes = (
        _parse_option_field(
            main_options,
            field_name="main_options",
            allow_overload=True,
            fragment_count=0,
            aggregate_value_bytes=0,
        )
    )

    if overload in (1, 3):
        parsed, _, fragment_count, aggregate_value_bytes = _parse_option_field(
            file_field,
            field_name="file_field",
            allow_overload=False,
            fragment_count=fragment_count,
            aggregate_value_bytes=aggregate_value_bytes,
        )
        fragments.extend(parsed)
    if overload in (2, 3):
        parsed, _, fragment_count, aggregate_value_bytes = _parse_option_field(
            sname_field,
            field_name="sname_field",
            allow_overload=False,
            fragment_count=fragment_count,
            aggregate_value_bytes=aggregate_value_bytes,
        )
        fragments.extend(parsed)

    grouped: dict[int, list[bytes]] = {}
    for code, value in fragments:
        grouped.setdefault(code, []).append(value)

    return DhcpOptionSet(
        overload=overload,
        options=tuple(
            DhcpOption(code=code, value=b"".join(parts))
            for code, parts in grouped.items()
        ),
    )
```

## Example

```python
def encoded_option(code: int, value: bytes) -> bytes:
    assert 1 <= code <= 254 and len(value) <= 255
    return bytes((code, len(value))) + value


def fixed_field(size: int, *options: bytes) -> bytes:
    encoded = b"".join(options) + bytes((_END,))
    assert len(encoded) <= size
    return encoded.ljust(size, bytes((_PAD,)))


main = b"".join(
    (
        bytes((_PAD,)),
        encoded_option(_OPTION_OVERLOAD, b"\x03"),
        encoded_option(67, b"/diskle"),
        encoded_option(12, b"node"),
        bytes((_END, _PAD, _PAD)),
    )
)
file_field = fixed_field(
    _FILE_FIELD_BYTES,
    encoded_option(67, b"ss/"),
    encoded_option(15, b"example"),
)
sname_field = fixed_field(
    _SNAME_FIELD_BYTES,
    encoded_option(67, b"foo"),
    encoded_option(15, b".test"),
    encoded_option(60, b""),
)

parsed = parse_dhcpv4_option_fields(main, file_field, sname_field)

assert parsed == DhcpOptionSet(
    overload=3,
    options=(
        DhcpOption(67, b"/diskless/foo"),
        DhcpOption(12, b"node"),
        DhcpOption(15, b"example.test"),
        DhcpOption(60, b""),
    ),
)

# Overload values 1 and 2 select exactly one secondary field. Repeated codes
# inside main and across the selected field retain their aggregate order.
file_only = parse_dhcpv4_option_fields(
    b"".join(
        (
            encoded_option(_OPTION_OVERLOAD, b"\x01"),
            encoded_option(1, b"a"),
            encoded_option(2, b"x"),
            encoded_option(1, b"b"),
            bytes((_END,)),
        )
    ),
    fixed_field(
        _FILE_FIELD_BYTES,
        encoded_option(1, b"c"),
        encoded_option(2, b"y"),
    ),
    b"inactive sname".ljust(_SNAME_FIELD_BYTES, b"\x7f"),
)
assert file_only == DhcpOptionSet(
    1,
    (DhcpOption(1, b"abc"), DhcpOption(2, b"xy")),
)

sname_only = parse_dhcpv4_option_fields(
    encoded_option(_OPTION_OVERLOAD, b"\x02")
    + encoded_option(3, b"left")
    + bytes((_END,)),
    b"inactive file".ljust(_FILE_FIELD_BYTES, b"\x7f"),
    fixed_field(_SNAME_FIELD_BYTES, encoded_option(3, b"right")),
)
assert sname_only == DhcpOptionSet(2, (DhcpOption(3, b"leftright"),))

# Inactive fields are fixed-width BOOTP data and are not parsed as options.
inactive = parse_dhcpv4_option_fields(
    encoded_option(12, b"host") + bytes((_END,)),
    b"boot image name".ljust(_FILE_FIELD_BYTES, b"\x7f"),
    b"server name".ljust(_SNAME_FIELD_BYTES, b"\x7f"),
)
assert inactive == DhcpOptionSet(None, (DhcpOption(12, b"host"),))

exact_fragment_cap = parse_dhcpv4_option_fields(
    encoded_option(1, b"") * _MAX_FRAGMENTS + bytes((_END,)),
    bytes(_FILE_FIELD_BYTES),
    bytes(_SNAME_FIELD_BYTES),
)
assert exact_fragment_cap == DhcpOptionSet(None, (DhcpOption(1, b""),))

exact_value_cap = parse_dhcpv4_option_fields(
    encoded_option(1, b"x" * 255) * 8
    + encoded_option(1, b"y" * 8)
    + bytes((_END,)),
    bytes(_FILE_FIELD_BYTES),
    bytes(_SNAME_FIELD_BYTES),
)
assert len(exact_value_cap.options[0].value) == _MAX_AGGREGATE_VALUE_BYTES


def is_rejected(
    candidate_main: object,
    candidate_file: object = bytes(_FILE_FIELD_BYTES),
    candidate_sname: object = bytes(_SNAME_FIELD_BYTES),
) -> bool:
    try:
        parse_dhcpv4_option_fields(
            candidate_main,  # type: ignore[arg-type]
            candidate_file,  # type: ignore[arg-type]
            candidate_sname,  # type: ignore[arg-type]
        )
    except (TypeError, ValueError):
        return True
    return False


bad_overload_in_file = fixed_field(
    _FILE_FIELD_BYTES,
    encoded_option(_OPTION_OVERLOAD, b"\x01"),
)
missing_file_end = bytes((_PAD,)) * _FILE_FIELD_BYTES
fragment_limit_main = (
    encoded_option(_OPTION_OVERLOAD, b"\x01")
    + encoded_option(1, b"") * (_MAX_FRAGMENTS - 1)
    + bytes((_END,))
)
one_more_fragment = fixed_field(_FILE_FIELD_BYTES, encoded_option(2, b""))

rejected = (
    is_rejected(bytearray(bytes((_END,)))),
    is_rejected(b""),
    is_rejected(bytes((_END,)) * (_MAX_MAIN_OPTIONS_BYTES + 1)),
    is_rejected(bytes((_END,)), bytes(_FILE_FIELD_BYTES - 1)),
    is_rejected(bytes((_END,)), bytes(_FILE_FIELD_BYTES), bytes(_SNAME_FIELD_BYTES + 1)),
    is_rejected(bytes((1,))),  # missing length and End
    is_rejected(bytes((1, 3, 0xAA, 0xBB))),  # value crosses the field boundary
    is_rejected(encoded_option(1, b"x")),  # missing End
    is_rejected(bytes((_END, 1))),  # non-Pad after End
    is_rejected(encoded_option(_OPTION_OVERLOAD, b"") + bytes((_END,))),
    is_rejected(encoded_option(_OPTION_OVERLOAD, b"\x04") + bytes((_END,))),
    is_rejected(
        encoded_option(_OPTION_OVERLOAD, b"\x01") * 2 + bytes((_END,))
    ),
    is_rejected(
        encoded_option(_OPTION_OVERLOAD, b"\x01") + bytes((_END,)),
        bad_overload_in_file,
    ),
    is_rejected(
        encoded_option(_OPTION_OVERLOAD, b"\x01") + bytes((_END,)),
        missing_file_end,
    ),
    is_rejected(fragment_limit_main, one_more_fragment),
    is_rejected(encoded_option(1, b"x" * 255) * 9 + bytes((_END,))),
)

assert all(rejected)
```

## Trade-offs and Limitations

Parsing is linear in the active field bytes and retains at most 256 option
fragments containing at most 2,048 aggregate value bytes. Both caps count the
validated option 52 even though it is returned separately rather than included
in `options`. Concatenation copies each retained ordinary value once more.

The parser applies RFC 3396 concatenation to every repeated ordinary code,
including unknown codes and values split on arbitrary byte boundaries. Output
order by first occurrence is a deterministic API choice; fragment boundaries
carry no semantic meaning. Unselected `file` and `sname` contents are ignored.

Each active field must contain End, every option must fit wholly within its
physical field, and every byte after End must be Pad. RFC 2132 requires End and
says following bytes should be Pad; rejecting a non-Pad tail is an intentional
closed-profile rule. Zero-length ordinary options remain valid.

This recipe does not parse a DHCPv4 packet or its cookie and does not require
option 53. It does not decode registered option values, validate their
individual lengths or repeatability, authenticate a message, or decide whether
the result is suitable for a DHCP state transition. The 4,096-byte main-field
limit and aggregate caps are local resource policy, not protocol maxima.

## Related Snippets

<!-- catalog:related:start -->
- [Parse One Bounded CoAP UDP Datagram Envelope](parse-one-bounded-coap-udp-datagram-envelope.md)
- [Decode One Bounded DNS Name with Strict Backward Compression Pointers](decode-one-bounded-dns-name-with-strict-backward-compression-pointers.md)
- [Reassemble One Bounded IPv4 Payload from Decoded Fragments](reassemble-one-bounded-ipv4-payload-from-decoded-fragments.md)
<!-- catalog:related:end -->
