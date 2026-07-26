---
title: "Split Quoted and Bracketed Log Fields"
snippet_type: algorithm
use_cases:
  - data-transformation
  - parsing
tested_python:
  - "3.14"
dependencies: []
related: []
---

# Split Quoted and Bracketed Log Fields

## Idea and Problem

Split a log line whose fields may be plain tokens, double-quoted text, or bracketed text without losing embedded spaces.

The parser implements a deliberately small grammar instead of relying on
`str.split()`. Quoted fields recognize only backslash, quote, tab, newline, and
carriage-return escapes. Bracketed fields end at the next closing bracket, and
plain fields may not contain structural delimiters. Malformed input fails
instead of being silently shifted into the wrong columns.

## When to Use

Use this algorithm when one controlled log format defines whitespace-separated
fields plus quoted request or agent strings and a bracketed timestamp. Freeze
the grammar and sample lines in tests before ingestion. Use a format-specific
parser or structured logging when fields, escaping, or delimiters differ.

## Implementation

```python
_QUOTED_ESCAPES = {
    "\\": "\\",
    '"': '"',
    "t": "\t",
    "n": "\n",
    "r": "\r",
}


def split_log_fields(line: str) -> list[str]:
    if not isinstance(line, str):
        raise TypeError("line must be text")
    if "\n" in line or "\r" in line:
        raise ValueError("line must not contain newline characters")

    fields: list[str] = []
    position = 0
    length = len(line)

    while position < length:
        while position < length and line[position].isspace():
            position += 1
        if position == length:
            break

        if line[position] == '"':
            position += 1
            characters: list[str] = []
            while position < length:
                character = line[position]
                if character == '"':
                    position += 1
                    break
                if character == "\\":
                    position += 1
                    if position == length or line[position] not in _QUOTED_ESCAPES:
                        raise ValueError("invalid escape in quoted field")
                    characters.append(_QUOTED_ESCAPES[line[position]])
                    position += 1
                    continue
                characters.append(character)
                position += 1
            else:
                raise ValueError("unterminated quoted field")
            field = "".join(characters)

        elif line[position] == "[":
            closing = line.find("]", position + 1)
            if closing < 0:
                raise ValueError("unterminated bracketed field")
            field = line[position + 1 : closing]
            position = closing + 1

        else:
            start = position
            while position < length and not line[position].isspace():
                position += 1
            field = line[start:position]
            if any(delimiter in field for delimiter in '"[]'):
                raise ValueError("structural delimiter inside plain field")

        if position < length and not line[position].isspace():
            raise ValueError("fields must be separated by whitespace")
        fields.append(field)

    return fields
```

## Example

```python
line = (
    r'127.0.0.1   [26/Jul/2026:10:00:00 +0000] '
    r'"GET /search?q=a\"b HTTP/1.1" 200 "agent\\name" ""'
)
fields = split_log_fields(line)

try:
    split_log_fields(r'"bad\q"')
except ValueError:
    invalid_escape_rejected = True
else:
    invalid_escape_rejected = False

try:
    split_log_fields('prefix [missing')
except ValueError:
    missing_bracket_rejected = True
else:
    missing_bracket_rejected = False

assert (
    fields,
    split_log_fields("   "),
    invalid_escape_rejected,
    missing_bracket_rejected,
) == (
    [
        "127.0.0.1",
        "26/Jul/2026:10:00:00 +0000",
        'GET /search?q=a"b HTTP/1.1',
        "200",
        "agent\\name",
        "",
    ],
    [],
    True,
    True,
)
```

## Trade-offs and Limitations

This is not a universal web-server log parser. It does not interpret a named
field layout, validate timestamps or status codes, accept hexadecimal escapes,
or support brackets nested inside bracketed values. Actual newline characters
are rejected even though escaped `\\n` is decoded inside quotes. Parsing is
linear in line length, but every decoded field is materialized as a new string.

## Related Snippets

<!-- catalog:related:start -->
No related snippets.
<!-- catalog:related:end -->
