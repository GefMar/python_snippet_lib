---
title: "Audit Bounded Python Source for Tab-Width-Independent Indentation"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
  - parsing
tested_python:
  - "3.14"
dependencies: []
related:
  - compare-bounded-python-expressions-structurally-without-execution.md
  - index-bounded-python-scope-bindings-with-symtable-without-execution.md
  - parse-a-bounded-space-indented-test-outline-into-leaf-paths.md
---

# Audit Bounded Python Source for Tab-Width-Independent Indentation

## Idea and Problem

Detect Python indentation whose block structure changes with the assumed tab width, while keeping ambiguity separate from tokenization failures and never executing the bounded source.

The result has three practical channels: a clean report, an immutable ambiguity
report derived from `NannyNag`, or a lexical exception such as
`IndentationError` or `tokenize.TokenError`. Keeping those channels separate is
important because a tabnanny-clean result is not proof that the source is valid
Python.

## When to Use

Use this check in an editor, review tool, source importer, or test fixture gate
that accepts bounded in-memory Python text and wants indentation to mean the
same thing at every conventional tab width. It is especially useful before a
later parser or compiler step when source must not be imported or executed.

Do not use it as a syntax checker, formatter, style rule, or security sandbox.
A formatter or a spaces-only policy is simpler when tabs are forbidden by
project convention. Run a real parser or compiler separately when syntactic
validity is also required.

## Implementation

```python
from dataclasses import dataclass
from io import StringIO
from tabnanny import NannyNag, process_tokens
from tokenize import TokenError, generate_tokens

_MAX_SOURCE_CHARACTERS = 65_536
_MAX_SOURCE_BYTES = 65_536
_MAX_PHYSICAL_LINES = 4_096
_MAX_PHYSICAL_LINE_CHARACTERS = 4_096


class SourceIndentationLimitError(ValueError):
    """Raised when source violates the bounded text contract."""


@dataclass(frozen=True, slots=True)
class IndentationAudit:
    ambiguous: bool
    line_number: int | None = None
    message: str | None = None
    line_text: str | None = None


def _validate_indentation_source(source: str) -> None:
    if type(source) is not str:
        raise TypeError("source must be an exact str")
    if len(source) > _MAX_SOURCE_CHARACTERS:
        raise SourceIndentationLimitError("source exceeds the character limit")

    try:
        encoded = source.encode("utf-8")
    except UnicodeEncodeError as exc:
        raise SourceIndentationLimitError("source must be valid UTF-8 text") from exc
    if len(encoded) > _MAX_SOURCE_BYTES:
        raise SourceIndentationLimitError("source exceeds the UTF-8 byte limit")

    physical_lines = tuple(StringIO(source))
    if len(physical_lines) > _MAX_PHYSICAL_LINES:
        raise SourceIndentationLimitError("source exceeds the physical-line limit")
    if any(
        len(line) > _MAX_PHYSICAL_LINE_CHARACTERS for line in physical_lines
    ):
        raise SourceIndentationLimitError(
            "a physical line exceeds the character limit"
        )


def audit_indentation(source: str) -> IndentationAudit:
    """Report tab-width-dependent indentation without executing source."""
    _validate_indentation_source(source)

    try:
        process_tokens(generate_tokens(StringIO(source).readline))
    except NannyNag as error:
        return IndentationAudit(
            ambiguous=True,
            line_number=error.get_lineno(),
            message=error.get_msg(),
            line_text=error.get_line(),
        )
    return IndentationAudit(ambiguous=False)
```

## Example

```python
ambiguous = audit_indentation(
    "if True:\n"
    "\tprint('first')\n"
    "        print('second')\n"
)
spaces_only = audit_indentation("if True:\n    print('ok')\n")
tabs_only = audit_indentation("if True:\n\tif False:\n\t\tprint('ok')\n")
syntax_invalid_but_clean = audit_indentation("if True print('invalid')\n")
not_executed = audit_indentation("raise RuntimeError('must not run')\n")

try:
    audit_indentation("if True:\n    pass\n  pass\n")
except IndentationError:
    malformed_indentation = True
else:
    malformed_indentation = False

try:
    audit_indentation("'''unterminated\n")
except TokenError:
    malformed_token_stream = True
else:
    malformed_token_stream = False

line_limit_ok = not audit_indentation("x" * 4_096).ambiguous
count_limit_ok = not audit_indentation("#\n" * 4_096).ambiguous
utf8_boundary_line = "#" + "é" * 2_047 + "\n"
byte_limit_ok = not audit_indentation(utf8_boundary_line * 16).ambiguous

limit_failures = 0
for candidate in (
    "x" * 4_097,
    "#\n" * 4_097,
    utf8_boundary_line * 17,
    "#" + "a" * 4_094 + "\f" + "b" * 4_095,
):
    try:
        audit_indentation(candidate)
    except SourceIndentationLimitError:
        limit_failures += 1

assert (
    ambiguous
    == IndentationAudit(
        ambiguous=True,
        line_number=3,
        message="inconsistent use of tabs and spaces in indentation",
        line_text="        print('second')",
    )
    and not spaces_only.ambiguous
    and not tabs_only.ambiguous
    and not syntax_invalid_but_clean.ambiguous
    and not not_executed.ambiguous
    and malformed_indentation
    and malformed_token_stream
    and line_limit_ok
    and count_limit_ok
    and byte_limit_ok
    and limit_failures == 4
)
```

## Trade-offs and Limitations

`tabnanny` checks whether indentation is unambiguous across tab widths; it does
not validate Python grammar, names, types, imports, or runtime behavior. Clean
but invalid source is therefore an expected result. Conversely, malformed
indentation and unfinished token constructs remain exceptions instead of being
misreported as tab ambiguity.

The input, physical-line, and UTF-8 limits bound retained text and diagnostic
size. Token generation is a bounded pass over the source, but tabnanny compares
indentation witnesses and does not expose a portable linear-time guarantee.
The check uses the current Python tokenizer, so results should be tested again
when the supported grammar version changes. No formatter changes or normalizes
the input.

## Related Snippets

<!-- catalog:related:start -->
- [Compare Bounded Python Expressions Structurally without Execution](compare-bounded-python-expressions-structurally-without-execution.md)
- [Index Bounded Python Scope Bindings with symtable without Execution](index-bounded-python-scope-bindings-with-symtable-without-execution.md)
- [Parse a Bounded Space-Indented Test Outline into Leaf Paths](parse-a-bounded-space-indented-test-outline-into-leaf-paths.md)
<!-- catalog:related:end -->
