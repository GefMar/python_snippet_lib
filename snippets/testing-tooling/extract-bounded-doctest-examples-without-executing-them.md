---
title: "Extract Bounded Doctest Examples Without Executing Them"
snippet_type: testing-technique
use_cases:
  - parsing
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - classify-bounded-interactive-python-source-as-complete-incomplete-or-invalid.md
  - compare-a-bounded-text-capture-against-a-golden-fixture.md
  - extract-a-bounded-static-python-annotation-index-without-evaluation.md
---

# Extract Bounded Doctest Examples Without Executing Them

## Idea and Problem

Extract source, expected output, exception text, location, indentation, and option overrides from a bounded doctest transcript without compiling or executing any example.

`doctest.DocTestParser` is separate from doctest's runners. A fixed local
option map and immutable output records make the retained representation
explicit while rejecting custom ambient option flags.

## When to Use

Use this in a documentation audit, test inventory, migration tool, or static
report that needs the structure of small doctest transcripts. It is useful when
the source may be syntactically invalid or deliberately side-effecting and the
caller must inspect text rather than determine whether examples pass.

Do not use extracted source as trusted code. Use a doctest runner only when
execution is intentional and isolated appropriately. Keep sensitive
transcripts out of logs: source, expected output, and exception messages are
returned as text and can contain secrets.

## Implementation

```python
import doctest
from dataclasses import dataclass
from io import StringIO

_MAX_TEXT_CHARACTERS = 65_536
_MAX_TEXT_BYTES = 65_536
_MAX_PHYSICAL_LINES = 4_096
_MAX_LINE_CHARACTERS = 4_096
_MAX_EXPANDED_CHARACTERS = 131_072
_MAX_EXAMPLES = 128
_MAX_FIELD_BYTES = 8_192
_MAX_RETAINED_BYTES = 65_536
_BUILTIN_OPTIONS = {
    doctest.DONT_ACCEPT_TRUE_FOR_1: "DONT_ACCEPT_TRUE_FOR_1",
    doctest.DONT_ACCEPT_BLANKLINE: "DONT_ACCEPT_BLANKLINE",
    doctest.NORMALIZE_WHITESPACE: "NORMALIZE_WHITESPACE",
    doctest.ELLIPSIS: "ELLIPSIS",
    doctest.SKIP: "SKIP",
    doctest.IGNORE_EXCEPTION_DETAIL: "IGNORE_EXCEPTION_DETAIL",
    doctest.REPORT_UDIFF: "REPORT_UDIFF",
    doctest.REPORT_CDIFF: "REPORT_CDIFF",
    doctest.REPORT_NDIFF: "REPORT_NDIFF",
    doctest.REPORT_ONLY_FIRST_FAILURE: "REPORT_ONLY_FIRST_FAILURE",
    doctest.FAIL_FAST: "FAIL_FAST",
}


class DoctestExtractionError(ValueError):
    """Base error for the closed extraction profile."""


class DoctestLimitError(DoctestExtractionError):
    """Raised when input or retained output exceeds a fixed limit."""


class DoctestParseError(DoctestExtractionError):
    """Raised when the parser cannot produce supported records."""


@dataclass(frozen=True, slots=True)
class ExtractedDoctestExample:
    line: int
    indent: int
    source: str
    expected_output: str
    expected_exception: str | None
    options: tuple[tuple[str, bool], ...]


def _utf8_size(value: str, *, field: str) -> int:
    try:
        return len(value.encode("utf-8"))
    except UnicodeEncodeError:
        raise DoctestLimitError(
            f"{field} must be UTF-8 encodable"
        ) from None


def _prepare_doctest_text(text: str) -> str:
    if type(text) is not str:
        raise TypeError("text must be an exact str")
    if len(text) > _MAX_TEXT_CHARACTERS:
        raise DoctestLimitError("text exceeds the character limit")
    if _utf8_size(text, field="text") > _MAX_TEXT_BYTES:
        raise DoctestLimitError("text exceeds the UTF-8 byte limit")

    normalized = text.replace("\r\n", "\n").replace("\r", "\n")
    physical_lines = tuple(StringIO(normalized))
    if len(physical_lines) > _MAX_PHYSICAL_LINES:
        raise DoctestLimitError("text exceeds the physical-line limit")
    if any(
        len(line[:-1] if line.endswith("\n") else line)
        > _MAX_LINE_CHARACTERS
        for line in physical_lines
    ):
        raise DoctestLimitError(
            "a physical line exceeds the character limit"
        )

    expanded = normalized.expandtabs(8)
    if len(expanded) > _MAX_EXPANDED_CHARACTERS:
        raise DoctestLimitError(
            "tab-expanded text exceeds the character limit"
        )
    return expanded


def extract_doctest_examples(
    text: str,
) -> tuple[ExtractedDoctestExample, ...]:
    """Parse bounded doctest records without compiling or running source."""
    prepared = _prepare_doctest_text(text)
    try:
        examples = doctest.DocTestParser().get_examples(
            prepared,
            "<snippet>",
        )
    except ValueError:
        raise DoctestParseError(
            "text is not a valid supported doctest transcript"
        ) from None

    validated_options: list[tuple[tuple[str, bool], ...]] = []
    for example in examples:
        try:
            options = tuple(
                sorted(
                    (_BUILTIN_OPTIONS[flag], enabled)
                    for flag, enabled in example.options.items()
                )
            )
        except KeyError:
            raise DoctestParseError(
                "text is not a valid supported doctest transcript"
            ) from None
        if any(type(enabled) is not bool for _, enabled in options):
            raise DoctestParseError(
                "text is not a valid supported doctest transcript"
            )
        validated_options.append(options)

    if len(examples) > _MAX_EXAMPLES:
        raise DoctestLimitError("example count exceeds the limit")

    retained_bytes = 0
    result: list[ExtractedDoctestExample] = []
    for example, options in zip(
        examples,
        validated_options,
        strict=True,
    ):
        retained_fields = (
            example.source,
            example.want,
            example.exc_msg or "",
        )
        field_sizes = tuple(
            _utf8_size(field, field="an extracted field")
            for field in retained_fields
        )
        if any(size > _MAX_FIELD_BYTES for size in field_sizes):
            raise DoctestLimitError(
                "an extracted field exceeds the UTF-8 byte limit"
            )
        retained_bytes += sum(field_sizes)
        if retained_bytes > _MAX_RETAINED_BYTES:
            raise DoctestLimitError(
                "extracted text exceeds the aggregate byte limit"
            )

        result.append(
            ExtractedDoctestExample(
                line=example.lineno + 1,
                indent=example.indent,
                source=example.source,
                expected_output=example.want,
                expected_exception=example.exc_msg,
                options=options,
            )
        )
    return tuple(result)
```

## Example

```python
transcript = """A small transcript.\r
>>> total = (\r
...     1 + 2\r
... )\r
>>> total  # doctest: +ELLIPSIS, -NORMALIZE_WHITESPACE\r
3\r
>>> print("first\\n\\nthird")\r
first\r
<BLANKLINE>\r
third\r
>>> raise LookupError("missing")\r
Traceback (most recent call last):\r
...\r
LookupError: missing\r
>>> effects.append("must not run")\r
"""
effects: list[str] = []
records = extract_doctest_examples(transcript)

invalid_python = extract_doctest_examples(
    ">>> this is not valid Python !!!\n"
)
prose_only = extract_doctest_examples("Only prose.\n")

try:
    extract_doctest_examples(
        ">>> 1  # doctest: +UNKNOWN_OPTION\n"
        + ("x" * 3_000 + "\n") * 3
    )
except DoctestParseError:
    unknown_option_rejected = True
else:
    unknown_option_rejected = False

limit_failures = 0
for candidate in (
    "x" * 65_537,
    "#\n" * 4_097,
    ("\t" * 4_095 + "\n") * 5,
    "".join(f">>> {index}\n" for index in range(129)),
    ">>> 1\n" + "x" * 8_193 + "\n",
):
    try:
        extract_doctest_examples(candidate)
    except DoctestLimitError:
        limit_failures += 1

assert (
    len(records) == 5
    and effects == []
    and records[0].line == 2
    and records[0].source == "total = (\n    1 + 2\n)\n"
    and records[1].options
    == (("ELLIPSIS", True), ("NORMALIZE_WHITESPACE", False))
    and records[2].expected_output
    == "first\n<BLANKLINE>\nthird\n"
    and records[3].expected_exception == "LookupError: missing\n"
    and records[4].expected_output == ""
    and len(invalid_python) == 1
    and prose_only == ()
    and unknown_option_rejected
    and limit_failures == 5
)
```

## Trade-offs and Limitations

CRLF and CR are normalized to LF, and tabs expand to eight-column stops before
parsing, matching the returned source and line positions. A terminal LF does
not count toward a physical line's character limit; Unicode line separators
remain ordinary characters. Per-field and aggregate byte limits count source,
expected output, and the separately retained final exception text, so
traceback information is deliberately counted twice where it appears in both
records.

The wrapper never directly reads or mutates
`doctest.OPTIONFLAGS_BY_NAME` and never registers a flag. The standard
parser necessarily consults that process-global registry while recognizing
directives; every parsed option is checked against the fixed 11-entry local
map before count or retained-field limits are evaluated.
An unregistered custom directive fails in the parser, while a registered one
fails at the local allowlist, with the same public error.

Input, expanded text, examples, and retained output are bounded, but the
standard parser's regular-expression work is implementation-dependent. Parsing
does not validate Python syntax or determine whether expected output would
match. No finder, runner, `testmod`, compiler, or execution API is used.

## Related Snippets

<!-- catalog:related:start -->
- [Classify Bounded Interactive Python Source as Complete, Incomplete, or Invalid](classify-bounded-interactive-python-source-as-complete-incomplete-or-invalid.md)
- [Compare a Bounded Text Capture Against a Golden Fixture](compare-a-bounded-text-capture-against-a-golden-fixture.md)
- [Extract a Bounded Static Python Annotation Index without Evaluation](extract-a-bounded-static-python-annotation-index-without-evaluation.md)
<!-- catalog:related:end -->
