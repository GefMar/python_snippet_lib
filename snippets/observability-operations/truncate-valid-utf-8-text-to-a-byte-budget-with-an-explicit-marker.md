---
title: "Truncate Valid UTF-8 Text to a Byte Budget with an Explicit Marker"
snippet_type: recipe
use_cases:
  - data-transformation
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - format-log-records-as-json-with-explicit-extra-fields.md
  - ../data-processing/limit-text-lines-across-arbitrary-chunks.md
  - ../configuration-serialization/decode-one-bounded-strict-utf-8-stream-across-arbitrary-byte-chunks.md
---

# Truncate Valid UTF-8 Text to a Byte Budget with an Explicit Marker

## Idea and Problem

Keep a bounded text prefix and append an explicit marker without splitting a UTF-8 code-point encoding or exceeding a byte budget.

Python string slicing counts code points rather than encoded bytes. A byte cap
therefore needs to account for the UTF-8 size of both the retained prefix and
the marker. Stopping before the first code point that does not fit preserves a
valid prefix because every later code point would require that same skipped
position first.

## When to Use

Use this recipe when already decoded, bounded text must fit a strict UTF-8 byte
field, such as a log attribute or small diagnostic record. Choose the marker
and total budget explicitly, and validate them even when the original text is
short enough to return unchanged.

Use a grapheme-aware or display-width-aware library when the boundary is
visible to people. Truncating at a Python code-point boundary keeps UTF-8 valid
but can still separate a combining mark, emoji modifier, or joined display
sequence from the code points around it.

## Implementation

```python
_MAX_TEXT_CODE_POINTS = 65_536
_MAX_TEXT_UTF8_BYTES = 262_144
_MAX_MARKER_CODE_POINTS = 32
_MAX_MARKER_UTF8_BYTES = 128
_MAX_BYTE_BUDGET = 262_144


def _validated_utf8(
    value: object,
    *,
    name: str,
    max_code_points: int,
    max_bytes: int,
) -> bytes:
    if type(value) is not str:
        raise TypeError(f"{name} must be an exact string")
    if len(value) > max_code_points:
        raise ValueError(f"{name} exceeds the code-point limit")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError:
        raise ValueError(f"{name} must be valid UTF-8 text") from None
    if len(encoded) > max_bytes:
        raise ValueError(f"{name} exceeds the UTF-8 byte limit")
    return encoded


def truncate_utf8_with_marker(
    text: str,
    *,
    marker: str,
    byte_budget: int,
) -> str:
    """Return text or its greatest fitting code-point prefix plus marker."""
    text_bytes = _validated_utf8(
        text,
        name="text",
        max_code_points=_MAX_TEXT_CODE_POINTS,
        max_bytes=_MAX_TEXT_UTF8_BYTES,
    )
    marker_bytes = _validated_utf8(
        marker,
        name="marker",
        max_code_points=_MAX_MARKER_CODE_POINTS,
        max_bytes=_MAX_MARKER_UTF8_BYTES,
    )
    if type(byte_budget) is not int:
        raise TypeError("byte_budget must be an exact non-boolean integer")
    if not 0 <= byte_budget <= _MAX_BYTE_BUDGET:
        raise ValueError("byte_budget is outside the supported range")
    if len(marker_bytes) > byte_budget:
        raise ValueError("marker does not fit the byte budget")

    if len(text_bytes) <= byte_budget:
        return text

    prefix_budget = byte_budget - len(marker_bytes)
    prefix_bytes = 0
    prefix_length = 0
    for character in text:
        character_bytes = len(character.encode("utf-8"))
        if prefix_bytes + character_bytes > prefix_budget:
            break
        prefix_bytes += character_bytes
        prefix_length += 1

    return text[:prefix_length] + marker
```

## Example

```python
def brute_utf8_truncation(text: str, marker: str, budget: int) -> str:
    if len(text.encode("utf-8")) <= budget:
        return text
    fitting_prefixes = [
        text[:stop]
        for stop in range(len(text) + 1)
        if len(text[:stop].encode("utf-8")) + len(marker.encode("utf-8"))
        <= budget
    ]
    return fitting_prefixes[-1] + marker


def verify_short_utf8_cases() -> None:
    from itertools import product

    alphabet = ("a", "é", "€", "😀")
    texts = tuple(
        "".join(characters)
        for length in range(4)
        for characters in product(alphabet, repeat=length)
    )
    markers = ("", "~", "€", "😀")

    for sample in texts:
        sample_bytes = len(sample.encode("utf-8"))
        for sample_marker in markers:
            marker_bytes = len(sample_marker.encode("utf-8"))
            for budget in range(
                marker_bytes,
                max(marker_bytes, sample_bytes) + 1,
            ):
                result = truncate_utf8_with_marker(
                    sample,
                    marker=sample_marker,
                    byte_budget=budget,
                )
                assert result == brute_utf8_truncation(
                    sample,
                    sample_marker,
                    budget,
                )
                assert len(result.encode("utf-8")) <= budget


verify_short_utf8_cases()

assert truncate_utf8_with_marker("ab😀cd", marker="…", byte_budget=7) == "ab…"
```

## Trade-offs and Limitations

For `n` code points and `B` admitted UTF-8 bytes, validation and the prefix
scan take `O(n + B)` time. The temporary full encoding and returned string use
`O(B)` memory. Each code point is encoded again during the scan, but a UTF-8
code point occupies at most four bytes, so this does not change the bound.

The marker is part of the same byte budget. It is validated unconditionally,
including when the original text fits and the marker will not be returned. An
empty marker is valid, and a zero budget therefore admits only an empty marker.
The longest fitting prefix may leave unused bytes when the next whole code
point is larger than the remaining capacity.

The function preserves a code-point prefix only. It does not preserve grapheme
clusters, display width, a suffix, normalization, ANSI escape integrity, or
log-format escaping, and it does not report how much text was removed.

## Related Snippets

<!-- catalog:related:start -->
- [Format Log Records as JSON with Explicit Extra Fields](format-log-records-as-json-with-explicit-extra-fields.md)
- [Limit Text Lines Across Arbitrary Chunks](../data-processing/limit-text-lines-across-arbitrary-chunks.md)
- [Decode One Bounded Strict UTF-8 Stream Across Arbitrary Byte Chunks](../configuration-serialization/decode-one-bounded-strict-utf-8-stream-across-arbitrary-byte-chunks.md)
<!-- catalog:related:end -->
