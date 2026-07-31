---
title: "Encode One Bounded Raw-Label DNS Name Without Compression"
snippet_type: recipe
use_cases:
  - networking
  - serialization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - decode-one-bounded-dns-name-with-strict-backward-compression-pointers.md
  - build-a-canonical-http-origin-key.md
  - read-and-write-size-capped-varint-frames.md
---

# Encode One Bounded Raw-Label DNS Name Without Compression

## Idea and Problem

Encode one already separated tuple of raw DNS labels as a complete uncompressed wire-format name.

Each non-root label is preceded by one length octet, and one zero octet
terminates the name. The empty tuple therefore has the single-byte root
encoding `b"\x00"`. Validation derives the maximum possible label count from
the DNS size limit, checks every label before allocating the result, and
preserves every admitted payload octet unchanged.

## When to Use

Use this recipe for bounded protocol fixtures or as one small component of a
DNS message encoder after another boundary has already produced exact raw
labels. It is useful when deterministic uncompressed output is preferable to
coordinating compression pointers with the offsets of a surrounding message.

Raw labels are bytes, not dot-separated text. Perform hostname policy, text
decoding, IDNA conversion, and label splitting before this boundary when those
semantics belong to the protocol. Use a maintained DNS implementation for
complete messages, compression, extended label types, resolver behavior, or
interoperability with unfamiliar extensions.

## Implementation

```python
_MAX_DNS_LABEL_OCTETS = 63
_MAX_DNS_NAME_WIRE_OCTETS = 255
_MAX_DNS_LABELS = 127


def encode_uncompressed_dns_name(labels: tuple[bytes, ...]) -> bytes:
    """Return one complete uncompressed DNS wire-format name."""
    if type(labels) is not tuple:
        raise TypeError("labels must be an exact tuple")
    if len(labels) > _MAX_DNS_LABELS:
        raise ValueError("label count cannot fit in one DNS wire-format name")

    wire_size = 1
    for index, label in enumerate(labels):
        if type(label) is not bytes:
            raise TypeError(f"labels[{index}] must be exact bytes")
        if not 1 <= len(label) <= _MAX_DNS_LABEL_OCTETS:
            raise ValueError(f"labels[{index}] length is outside 1..63")
        wire_size += 1 + len(label)
        if wire_size > _MAX_DNS_NAME_WIRE_OCTETS:
            raise ValueError("encoded DNS name exceeds 255 octets")

    encoded = bytearray(wire_size)
    offset = 0
    for label in labels:
        encoded[offset] = len(label)
        offset += 1
        encoded[offset : offset + len(label)] = label
        offset += len(label)
    encoded[offset] = 0
    return bytes(encoded)
```

## Example

```python
def scan_test_wire_name(encoded: bytes) -> tuple[bytes, ...]:
    """Decode this closed uncompressed form for independent example checks."""
    labels: list[bytes] = []
    offset = 0
    while True:
        assert offset < len(encoded)
        label_size = encoded[offset]
        offset += 1
        if label_size == 0:
            break
        assert 1 <= label_size <= 63
        label_end = offset + label_size
        assert label_end <= len(encoded)
        labels.append(encoded[offset:label_end])
        offset = label_end
    assert offset == len(encoded)
    return tuple(labels)


root = encode_uncompressed_dns_name(())
labels = (b"www", b"example", b"com")
ordinary = encode_uncompressed_dns_name(labels)
arbitrary_octets = (b"\x00\xff.\xc0",)
arbitrary = encode_uncompressed_dns_name(arbitrary_octets)
maximum_labels = (
    b"a" * 63,
    b"b" * 63,
    b"c" * 63,
    b"d" * 61,
)
maximum = encode_uncompressed_dns_name(maximum_labels)

assert root == b"\x00"
assert ordinary == b"\x03www\x07example\x03com\x00"
assert arbitrary == b"\x04\x00\xff.\xc0\x00"
assert len(maximum) == 255
assert scan_test_wire_name(root) == ()
assert scan_test_wire_name(ordinary) == labels
assert scan_test_wire_name(arbitrary) == arbitrary_octets
assert scan_test_wire_name(maximum) == maximum_labels


def assert_rejected(value: object, expected: type[Exception]) -> None:
    try:
        encode_uncompressed_dns_name(value)
    except expected:
        return
    raise AssertionError("invalid labels were unexpectedly accepted")


assert_rejected([b"example"], TypeError)
assert_rejected((bytearray(b"example"),), TypeError)
assert_rejected((b"",), ValueError)
assert_rejected((b"x" * 64,), ValueError)
assert_rejected((*maximum_labels[:-1], b"d" * 62), ValueError)
assert_rejected((b"x",) * 128, ValueError)

assert scan_test_wire_name(encode_uncompressed_dns_name((b"final",))) == (b"final",)
```

## Trade-offs and Limitations

For `k` labels and `w` encoded octets, validation takes `O(k)` time and output
construction takes `O(w)` time and space. The result is allocated only after
the complete label tuple has passed validation. The 255-octet wire limit also
implies at most 127 non-empty one-octet labels.

Uncompressed output is simple and independent of a containing message, but it
can be larger than a name that reuses an earlier suffix through compression.
Compression cannot be added locally because a pointer value depends on an
offset in the complete DNS message.

Admitted label payloads may contain any octet, including zero, dots, bytes
above ASCII, and bytes that resemble compression-pointer tags. This encoder
does not claim that those labels are hostnames, compare names case-insensitively,
interpret a trailing textual dot, or establish an IDNA policy. It does not
build a DNS header, question, resource record, packet, lookup, or network I/O.

## Related Snippets

<!-- catalog:related:start -->
- [Decode One Bounded DNS Name with Strict Backward Compression Pointers](decode-one-bounded-dns-name-with-strict-backward-compression-pointers.md)
- [Build a Canonical HTTP Origin Key](build-a-canonical-http-origin-key.md)
- [Read and Write Size-Capped Varint Frames](read-and-write-size-capped-varint-frames.md)
<!-- catalog:related:end -->
