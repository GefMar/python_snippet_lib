---
title: "Replace Selected Python NAME Tokens without Touching Strings or Comments"
snippet_type: recipe
use_cases:
  - automation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - decode-bounded-python-source-bytes-with-tokenize-detect-encoding.md
  - compare-bounded-python-expressions-structurally-without-execution.md
  - audit-bounded-python-source-for-tab-width-independent-indentation.md
---

# Replace Selected Python NAME Tokens without Touching Strings or Comments

## Idea and Problem

Replace selected lexical Python NAME tokens while preserving every code point outside their spans.

Token positions provide exact one-based row and zero-based column coordinates.
Mapping those coordinates back to offsets permits one reconstruction from the
original untouched gaps and replacement identifiers. This avoids
`tokenize.untokenize`, whose output is token-equivalent but is not designed to
preserve the original spacing and newline spelling exactly.

Strings and comments are different token kinds, so matching text inside them
is left alone. In Python 3.14, names inside f-string expressions are tokenized
as `NAME` and are replaced, while f-string literal text remains untouched.

## When to Use

Use this recipe for a controlled lexical migration, fixture transformation, or
source experiment where replacing every matching name spelling is precisely
the desired rule. The input must be trusted only as text: the function
tokenizes it but never compiles, imports, evaluates, or executes it.

Use a concrete-syntax-tree refactoring tool when scope, binding, attribute, or
keyword-argument meaning matters. Lexical equality deliberately cannot tell
those roles apart.

## Implementation

```python
import io
import keyword
import tokenize

_MAX_SOURCE_CHARACTERS = 65_536
_MAX_SOURCE_BYTES = 131_072
_MAX_PHYSICAL_ROWS = 4_096
_MAX_TOKENS = 32_768
_MAX_REPLACEMENTS = 64
_MAX_IDENTIFIER_CHARACTERS = 128
_MAX_IDENTIFIER_BYTES = 256


class PythonNameRewriteError(ValueError):
    pass


def _encoded_size(value: str, *, field: str) -> int:
    try:
        return len(value.encode("utf-8"))
    except UnicodeEncodeError:
        raise PythonNameRewriteError(
            f"{field} must contain Unicode scalar values only"
        ) from None


def _validate_source(source: object) -> str:
    if type(source) is not str:
        raise TypeError("source must be an exact string")
    if not source:
        raise ValueError("source must not be empty")
    if len(source) > _MAX_SOURCE_CHARACTERS:
        raise ValueError("source exceeds the character limit")
    if _encoded_size(source, field="source") > _MAX_SOURCE_BYTES:
        raise ValueError("source exceeds the UTF-8 byte limit")
    for index, character in enumerate(source):
        if character == "\r" and (
            index + 1 == len(source) or source[index + 1] != "\n"
        ):
            raise PythonNameRewriteError("source must use LF or CRLF newlines")
    if source.count("\n") + 1 > _MAX_PHYSICAL_ROWS:
        raise ValueError("source exceeds the physical-row limit")
    return source


def _validate_replacements(replacements: object) -> dict[str, str]:
    if type(replacements) is not dict:
        raise TypeError("replacements must be an exact dict")
    if not 1 <= len(replacements) <= _MAX_REPLACEMENTS:
        raise ValueError("replacement count is outside the supported range")

    checked: dict[str, str] = {}
    for old, new in replacements.items():
        if type(old) is not str or type(new) is not str:
            raise TypeError("replacement keys and values must be exact strings")
        for value, field in ((old, "old identifier"), (new, "new identifier")):
            if not 1 <= len(value) <= _MAX_IDENTIFIER_CHARACTERS:
                raise ValueError(f"{field} character length is unsupported")
            if _encoded_size(value, field=field) > _MAX_IDENTIFIER_BYTES:
                raise ValueError(f"{field} exceeds the UTF-8 byte limit")
            if not value.isidentifier():
                raise ValueError(f"{field} must be a Python identifier")
            if keyword.iskeyword(value) or keyword.issoftkeyword(value):
                raise ValueError(f"{field} must not be a Python keyword")
        if old == new:
            raise ValueError("every replacement must change the spelling")
        checked[old] = new
    return checked


def replace_python_names(
    source: str,
    replacements: dict[str, str],
) -> str:
    text = _validate_source(source)
    checked = _validate_replacements(replacements)
    line_starts = [0]
    line_starts.extend(
        index + 1
        for index, character in enumerate(text)
        if character == "\n"
    )

    edits: list[tuple[int, int, str]] = []
    token_count = 0
    try:
        tokens = tokenize.generate_tokens(io.StringIO(text).readline)
        for token in tokens:
            token_count += 1
            if token_count > _MAX_TOKENS:
                raise ValueError("source exceeds the generated-token limit")
            if token.type == tokenize.ERRORTOKEN:
                raise PythonNameRewriteError("source contains an error token")
            if token.type != tokenize.NAME or token.string not in checked:
                continue

            start_row, start_column = token.start
            end_row, end_column = token.end
            start = line_starts[start_row - 1] + start_column
            end = line_starts[end_row - 1] + end_column
            edits.append((start, end, checked[token.string]))
    except (IndentationError, tokenize.TokenError) as error:
        raise PythonNameRewriteError("source cannot be tokenized completely") from error

    pieces: list[str] = []
    cursor = 0
    for start, end, replacement in edits:
        pieces.append(text[cursor:start])
        pieces.append(replacement)
        cursor = end
    pieces.append(text[cursor:])
    return "".join(pieces)
```

## Example

```python
source = (
    "value = obj.value\r\n"
    'text = "value"\r\n'
    "# value\r\n"
    'result = f"{value}:{obj.value}"\r\n'
)

rewritten = replace_python_names(
    source,
    {"value": "item", "result": "output"},
)

assert rewritten == (
    "item = obj.item\r\n"
    'text = "value"\r\n'
    "# value\r\n"
    'output = f"{item}:{obj.item}"\r\n'
)
assert rewritten.count("\r\n") == source.count("\r\n")
assert replace_python_names("other = 1\n", {"value": "item"}) == "other = 1\n"
```

## Trade-offs and Limitations

Tokenization plus one-pass reconstruction takes `O(s + o)` time for source
size `s` and output size `o`. Auxiliary state is proportional to physical rows,
matching tokens, and the returned text. The independent source, token, and
identifier limits imply a conservative output bound of 4,227,072 code points
and 8,486,912 UTF-8 bytes when every admitted match expands maximally.

The function rejects standalone carriage returns, error tokens, incomplete
multiline constructs, invalid indentation, surrogate code points, and
oversized input. It admits LF and CRLF source but does not require a
syntactically valid module: tokenizable fragments may still fail parsing. Exact
gap copying preserves the admitted newline spelling, tabs, indentation,
comments, string contents, and f-string literal text outside replaced spans.

This is lexical rewriting rather than refactoring. It replaces matching names
after dots, in definitions and calls, in keyword-argument labels, and across
unrelated or shadowed scopes. Many source names may map to one target name, so
collisions are allowed. The function does not follow imports, resolve bindings,
format code, prove syntax, or claim behavioral equivalence.

## Related Snippets

<!-- catalog:related:start -->
- [Decode Bounded Python Source Bytes with Python Encoding Detection](decode-bounded-python-source-bytes-with-tokenize-detect-encoding.md)
- [Compare Bounded Python Expressions Structurally without Execution](compare-bounded-python-expressions-structurally-without-execution.md)
- [Audit Bounded Python Source for Tab-Width-Independent Indentation](audit-bounded-python-source-for-tab-width-independent-indentation.md)
<!-- catalog:related:end -->
