---
title: "Tokenize a Bounded POSIX-Style Argument String Without Expansion"
snippet_type: recipe
use_cases:
  - configuration
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - expand-bounded-nested-brace-alternatives.md
  - parse-a-bounded-component-options-expression.md
  - ../security-privacy/separate-executable-and-redacted-views-of-a-command-argument-vector.md
---

# Tokenize a Bounded POSIX-Style Argument String Without Expansion

## Idea and Problem

Turn one small single-line configuration string into an immutable tuple of tokens under one fixed POSIX-style lexical mode.

Quotes and backslashes can group or escape text without involving a command
interpreter. Comments, source inclusion, punctuation splitting, environment
variables, substitutions, globs, and every other expansion step stay disabled.
Input, token count, individual token size, and aggregate token size are all
bounded before the tuple is returned.

## When to Use

Use this recipe when a configuration format deliberately stores a short
human-authored argument-like string and its consumer needs only quote removal
and backslash handling. Treat the returned strings as untrusted configuration:
the caller still owns the allowed vocabulary and the meaning of every token.

Prefer a JSON array, TOML array, or another structured list when the format can
represent tokens directly. Those formats avoid a second lexical language and
preserve each value without quoting rules.

## Implementation

```python
import shlex

_MAX_SOURCE_BYTES = 4_096
_MAX_TOKENS = 64
_MAX_TOKEN_BYTES = 1_024
_MAX_TOTAL_TOKEN_BYTES = 3_072


def _validated_argument_source(value: object) -> str:
    if type(value) is not str:
        raise TypeError("source must be an exact string")
    if not 1 <= len(value) <= _MAX_SOURCE_BYTES:
        raise ValueError("source byte size is outside the supported range")
    if any(not character.isprintable() for character in value):
        raise ValueError("source must contain printable single-line text only")
    if len(value.encode("utf-8")) > _MAX_SOURCE_BYTES:
        raise ValueError("source byte size is outside the supported range")
    return value


def tokenize_posix_style_arguments(source: str) -> tuple[str, ...]:
    text = _validated_argument_source(source)
    lexer = shlex.shlex(text, posix=True, punctuation_chars=False)
    lexer.commenters = ""
    lexer.source = None
    lexer.whitespace_split = True

    tokens: list[str] = []
    total_token_bytes = 0
    while True:
        try:
            token = lexer.get_token()
        except ValueError:
            raise ValueError("source has invalid quoting or escaping") from None
        if token is None:
            break
        if len(tokens) == _MAX_TOKENS:
            raise ValueError("token count exceeds the supported limit")

        token_bytes = len(token.encode("utf-8"))
        if token_bytes > _MAX_TOKEN_BYTES:
            raise ValueError("a token exceeds the supported byte size")
        total_token_bytes += token_bytes
        if total_token_bytes > _MAX_TOTAL_TOKEN_BYTES:
            raise ValueError("tokens exceed the aggregate byte limit")
        tokens.append(token)

    if not tokens:
        raise ValueError("source must produce at least one token")
    return tuple(tokens)
```

## Example

```python
tokens = tokenize_posix_style_arguments(
    'worker --label "daily report" \'$HOME\' "*.txt" \'x;y\' "" #literal'
)

try:
    tokenize_posix_style_arguments("worker 'unfinished")
except ValueError:
    malformed_rejected = True
else:
    malformed_rejected = False

try:
    tokenize_posix_style_arguments("worker\nnext")
except ValueError:
    control_rejected = True
else:
    control_rejected = False

assert (tokens, malformed_rejected, control_rejected) == (
    (
        "worker",
        "--label",
        "daily report",
        "$HOME",
        "*.txt",
        "x;y",
        "",
        "#literal",
    ),
    True,
    True,
)
```

## Trade-offs and Limitations

This is one POSIX-style lexical profile, not a canonical serialization of an
argument vector, a complete shell grammar, or Windows command-line syntax.
It removes supported quotes and interprets backslash escapes, but it performs
no variable, tilde, command, arithmetic, or glob expansion. With punctuation
splitting disabled, shell operators have no special execution meaning here.
The function never invokes a program or decides whether a parsed token is an
acceptable executable, option, path, or value.

The complete source is already resident in memory and is bounded before
lexing. Token limits are enforced as the lexer produces values, and the result
requires memory proportional to the admitted token bytes. All non-printable
characters, including tabs and line breaks, are rejected. Printable Unicode is
kept exactly apart from quote and backslash processing; no normalization or
platform encoding policy is supplied.

## Related Snippets

<!-- catalog:related:start -->
- [Expand Bounded Nested Brace Alternatives](expand-bounded-nested-brace-alternatives.md)
- [Parse a Bounded Component Options Expression](parse-a-bounded-component-options-expression.md)
- [Separate Executable and Redacted Views of a Command Argument Vector](../security-privacy/separate-executable-and-redacted-views-of-a-command-argument-vector.md)
<!-- catalog:related:end -->
