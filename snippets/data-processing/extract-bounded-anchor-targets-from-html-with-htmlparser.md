---
title: "Extract Bounded Anchor Targets from HTML with HTMLParser"
snippet_type: recipe
use_cases:
  - data-transformation
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/parse-a-bounded-xml-envelope-with-closed-variant-dispatch.md
  - ../security-privacy/resolve-a-bounded-relative-http-reference-under-a-same-origin-policy.md
  - ../algorithms-data-structures/find-all-exact-multi-pattern-text-matches-with-aho-corasick.md
---

# Extract Bounded Anchor Targets from HTML with HTMLParser

## Idea and Problem

Extract decoded href values and source positions from a bounded HTML fragment without building a DOM or fetching anything.

String searches cannot distinguish real start tags from text inside comments or
script elements, and they do not apply HTML attribute entity decoding.
`HTMLParser` provides event callbacks for those lexical distinctions. Counting
every start-tag callback and attribute keeps the tolerant parser's retained
work explicit.

## When to Use

Use this helper for a small trusted or pre-screened HTML fragment when a test,
documentation tool, or offline transformation needs anchor targets in source
order. Empty targets and duplicates are preserved because they can be
meaningful to the caller.

This is tolerant extraction, not HTML conformance validation or browser DOM
construction. The returned strings have not been resolved, normalized, or
classified as safe URLs. Apply a separate URL policy before using a target for
navigation, filesystem access, or a network request.

## Implementation

```python
from dataclasses import dataclass
from html.parser import HTMLParser

_MAX_HTML_CHARACTERS = 65_536
_MAX_HTML_BYTES = 65_536
_MAX_START_TAGS = 4_096
_MAX_ATTRIBUTES = 8_192
_MAX_ANCHORS = 512
_MAX_HREF_BYTES = 2_048


@dataclass(frozen=True, slots=True)
class AnchorTarget:
    line: int
    column: int
    href: str


class _AnchorParser(HTMLParser):
    def __init__(self) -> None:
        super().__init__(convert_charrefs=True)
        self.start_tag_count = 0
        self.attribute_count = 0
        self.targets: list[AnchorTarget] = []

    def _handle_start(
        self,
        tag: str,
        attributes: list[tuple[str, str | None]],
    ) -> None:
        self.start_tag_count += 1
        if self.start_tag_count > _MAX_START_TAGS:
            raise ValueError("HTML exceeds the supported start-tag count")
        self.attribute_count += len(attributes)
        if self.attribute_count > _MAX_ATTRIBUTES:
            raise ValueError("HTML exceeds the supported attribute count")
        if tag != "a":
            return

        href_values = [
            value for name, value in attributes if name == "href"
        ]
        if not href_values:
            return
        if len(href_values) != 1:
            raise ValueError("an anchor must not contain duplicate href attributes")
        href = href_values[0]
        if href is None:
            raise ValueError("an anchor href must have a value")
        if len(href) > _MAX_HREF_BYTES:
            raise ValueError("anchor href exceeds the supported byte count")
        try:
            encoded_href = href.encode("utf-8")
        except UnicodeEncodeError as error:
            raise ValueError("anchor href must not contain Unicode surrogates") from error
        if len(encoded_href) > _MAX_HREF_BYTES:
            raise ValueError("anchor href exceeds the supported byte count")
        if len(self.targets) >= _MAX_ANCHORS:
            raise ValueError("HTML exceeds the supported retained-anchor count")

        line, column = self.getpos()
        self.targets.append(AnchorTarget(line=line, column=column, href=href))

    def handle_starttag(
        self,
        tag: str,
        attrs: list[tuple[str, str | None]],
    ) -> None:
        self._handle_start(tag, attrs)

    def handle_startendtag(
        self,
        tag: str,
        attrs: list[tuple[str, str | None]],
    ) -> None:
        self._handle_start(tag, attrs)


def extract_anchor_targets(html: str) -> tuple[AnchorTarget, ...]:
    if type(html) is not str:
        raise TypeError("html must be an exact string")
    if len(html) > _MAX_HTML_CHARACTERS:
        raise ValueError("HTML exceeds the supported character count")
    if len(html) > _MAX_HTML_BYTES:
        raise ValueError("HTML exceeds the supported byte count")
    try:
        encoded_html = html.encode("utf-8")
    except UnicodeEncodeError as error:
        raise ValueError("HTML must not contain Unicode surrogates") from error
    if len(encoded_html) > _MAX_HTML_BYTES:
        raise ValueError("HTML exceeds the supported UTF-8 byte count")

    parser = _AnchorParser()
    try:
        parser.feed(html)
        parser.close()
    except (MemoryError, RecursionError) as error:
        raise ValueError("HTML exceeds the parser's supported depth") from error
    return tuple(parser.targets)
```

## Example

```python
fragment = """<A HREF="docs?a=1&amp;b=2">first</A>
<a href=next>second</a>
<!-- <a href="comment">ignored</a> -->
<script>const example = '<a href="script">';</script>
<a href=""/><a href="same">third</a><a HREF="same">fourth</a>
<a>without a target</a>"""

targets = extract_anchor_targets(fragment)

assert tuple((target.line, target.href) for target in targets) == (
    (1, "docs?a=1&b=2"),
    (2, "next"),
    (5, ""),
    (5, "same"),
    (5, "same"),
)
assert targets[0].column == 0

for malformed in ('<a href="one" HREF="two">', "<a href>"):
    try:
        extract_anchor_targets(malformed)
    except ValueError:
        pass
    else:
        raise AssertionError("duplicate or valueless href must be rejected")

href_boundary = "x" * 2_048
assert extract_anchor_targets(
    f'<a href="{href_boundary}">'
)[0].href == href_boundary
try:
    extract_anchor_targets(f'<a href="{"x" * 2_049}">')
except ValueError:
    pass
else:
    raise AssertionError("a 2,049-byte href must be rejected")

assert len(extract_anchor_targets('<a href="x">' * 512)) == 512
try:
    extract_anchor_targets('<a href="x">' * 513)
except ValueError:
    pass
else:
    raise AssertionError("513 retained anchors must be rejected")

assert extract_anchor_targets("<div>" * 4_096) == ()
try:
    extract_anchor_targets("<div>" * 4_097)
except ValueError:
    pass
else:
    raise AssertionError("4,097 start tags must be rejected")

attribute_boundary = "<div " + "a=x " * 8_192 + ">"
assert extract_anchor_targets(attribute_boundary) == ()
try:
    extract_anchor_targets("<div " + "a=x " * 8_193 + ">")
except ValueError:
    pass
else:
    raise AssertionError("8,193 attributes must be rejected")

assert targets[-1].href == "same"
```

## Trade-offs and Limitations

The parser consumes one bounded fragment. For `G` start and start-end tags,
`A` attributes, and `H` processed href bytes, the wrapper performs
`O(G + A + H)` callback work. The immutable result retains `O(R + H)` state
for `R` accepted anchors and their target text. Parser internals are
implementation-dependent beyond the fixed input cap.

`HTMLParser` deliberately recovers from many malformed inputs. Successful
extraction therefore does not mean the fragment is valid, safe, or parsed
exactly as a browser would build its DOM. The source position is the start of
the anchor tag, with one-based lines and zero-based columns; it is not the
position of the href value itself.

Character references in attribute values are decoded by the parser. The
helper does not preserve original quoting, attribute spelling, or raw entity
text, and it ignores anchors without `href`. It performs no URL resolution,
same-origin enforcement, scheme filtering, network access, or document-wide
link validation.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded XML Envelope with Closed Variant Dispatch](../configuration-serialization/parse-a-bounded-xml-envelope-with-closed-variant-dispatch.md)
- [Resolve a Bounded Relative HTTP Reference under a Same-Origin Policy](../security-privacy/resolve-a-bounded-relative-http-reference-under-a-same-origin-policy.md)
- [Find All Exact Multi-Pattern Text Matches with Aho-Corasick](../algorithms-data-structures/find-all-exact-multi-pattern-text-matches-with-aho-corasick.md)
<!-- catalog:related:end -->
