---
title: "Limit Text Lines Across Arbitrary Chunks"
snippet_type: recipe
use_cases:
  - parsing
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - split-quoted-and-bracketed-log-fields.md
  - ../python-language/read-fixed-size-blocks-with-iter-sentinel.md
---

# Limit Text Lines Across Arbitrary Chunks

## Idea and Problem

Reassemble separator-delimited text arriving in arbitrary chunks while retaining only a bounded prefix of each logical line.

Chunk boundaries do not imply line boundaries. The limiter emits one record per
line, marks an overlong prefix as truncated, and discards the rest of that same
line before resuming, so a single unbounded line cannot grow its buffer.

## When to Use

Use this recipe after a trusted incremental decoder when logs or other text
streams arrive in pieces unrelated to their separators. Choose a positive
character cap and a one-character separator before processing. Use a byte-level
framer when the actual constraint is encoded size, or a full parser when
quoting and escaping affect where lines end.

## Implementation

```python
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class LimitedLine:
    text: str
    truncated: bool


class TextLineLimiter:
    def __init__(self, max_characters: int, *, separator: str = "\n") -> None:
        if isinstance(max_characters, bool) or not isinstance(max_characters, int):
            raise TypeError("max_characters must be an integer")
        if max_characters <= 0:
            raise ValueError("max_characters must be positive")
        if not isinstance(separator, str):
            raise TypeError("separator must be text")
        if len(separator) != 1:
            raise ValueError("separator must contain exactly one character")

        self._max_characters = max_characters
        self._separator = separator
        self._parts: list[str] = []
        self._length = 0
        self._discarding = False
        self._finished = False

    def _take_line(self, *, truncated: bool) -> LimitedLine:
        line = LimitedLine("".join(self._parts), truncated)
        self._parts.clear()
        self._length = 0
        return line

    def feed(self, chunk: str) -> list[LimitedLine]:
        if self._finished:
            raise RuntimeError("cannot feed a finished limiter")
        if not isinstance(chunk, str):
            raise TypeError("chunk must be text")

        emitted: list[LimitedLine] = []
        segments = chunk.split(self._separator)
        for index, segment in enumerate(segments):
            ends_line = index < len(segments) - 1

            if self._discarding:
                if ends_line:
                    self._discarding = False
                continue

            remaining = self._max_characters - self._length
            if len(segment) <= remaining:
                if segment:
                    self._parts.append(segment)
                    self._length += len(segment)
                if ends_line:
                    emitted.append(self._take_line(truncated=False))
                continue

            if remaining:
                self._parts.append(segment[:remaining])
                self._length += remaining
            emitted.append(self._take_line(truncated=True))
            self._discarding = not ends_line

        return emitted

    def finish(self) -> list[LimitedLine]:
        if self._finished:
            raise RuntimeError("limiter is already finished")
        self._finished = True
        if self._discarding or self._length == 0:
            return []
        return [self._take_line(truncated=False)]
```

## Example

```python
limiter = TextLineLimiter(5)
lines = []
lines.extend(limiter.feed("ab"))
lines.extend(limiter.feed("cde"))
lines.extend(limiter.feed("fgh\nok\n\nlast"))
lines.extend(limiter.finish())

exact = TextLineLimiter(3)
exact_lines = exact.feed("abc\n") + exact.finish()

assert (lines, exact_lines) == (
    [
        LimitedLine("abcde", True),
        LimitedLine("ok", False),
        LimitedLine("", False),
        LimitedLine("last", False),
    ],
    [LimitedLine("abc", False)],
)
```

## Trade-offs and Limitations

The cap counts Python characters, not encoded bytes or user-perceived grapheme
clusters. Input must already be decoded, and the recipe neither normalizes
newlines nor supports multi-character separators. Truncation preserves only
the prefix and does not report the discarded length. `finish()` closes the
limiter, intentionally emits no extra line after a trailing separator, and
cannot distinguish an entirely empty stream from an empty unterminated line.
Instances are stateful and not safe for concurrent calls.
Each `feed()` call also materializes its emitted records, so callers should keep
individual chunks bounded when a chunk can contain many short lines.

## Related Snippets

<!-- catalog:related:start -->
- [Split Quoted and Bracketed Log Fields](split-quoted-and-bracketed-log-fields.md)
- [Read Fixed-Size Blocks with iter() and a Sentinel](../python-language/read-fixed-size-blocks-with-iter-sentinel.md)
<!-- catalog:related:end -->
