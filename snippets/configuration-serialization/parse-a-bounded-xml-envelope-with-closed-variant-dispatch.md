---
title: "Parse a Bounded XML Envelope with Closed Variant Dispatch"
snippet_type: integration
use_cases:
  - parsing
  - security
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: defusedxml
    version: "0.7.1"
related:
  - ../networking-protocols/parse-a-bounded-three-state-json-response-envelope.md
  - load-a-bounded-protobuf-descriptor-set-in-dependency-order.md
  - parse-a-bounded-nested-bracket-tree.md
---

# Parse a Bounded XML Envelope with Closed Variant Dispatch

## Idea and Problem

Parse one size-capped XML document into an accepted, action-needed, or rejected immutable value under a closed namespaced grammar.

The root is exactly `{urn:example:bounded-envelope:1}envelope` with unqualified
`version="1"` and `kind` attributes. Each known kind has one ordered set of
attribute-free leaf elements. Parser defenses reject DTDs, entity declarations,
and external references; a separate post-parse walk bounds elements, depth,
attributes, individual text nodes, and total text before variant dispatch.

## When to Use

Use this integration when a complete small XML envelope is already available as
immutable bytes and every producer follows the same fixed version-one contract.
Catch `XmlEnvelopeError` at the input boundary, then handle the returned closed
union explicitly.

Keep document acquisition, trust decisions, and application behavior outside
this parser. Define a new namespace or versioned parser when the grammar changes;
do not silently accept additional kinds, attributes, or children. A schema-driven
validator is a better fit when a large externally maintained XML vocabulary is
the actual contract.

## Implementation

```python
import re
from dataclasses import dataclass
from typing import Never, TypeAlias
from xml.etree.ElementTree import Element, ParseError

from defusedxml import ElementTree
from defusedxml.common import DefusedXmlException


_MAX_SOURCE_BYTES = 64 * 1024
_MAX_ELEMENTS = 4
_MAX_DEPTH = 2
_MAX_ATTRIBUTES = 2
_MAX_TEXT_NODE_BYTES = 256
_MAX_TOTAL_TEXT_BYTES = 1_024
_MAX_REVISION = 1_000_000
_MAX_PRIORITY = 100

_NAMESPACE = "urn:example:bounded-envelope:1"
_ROOT_TAG = f"{{{_NAMESPACE}}}envelope"
_IDENTIFIER_TAG = f"{{{_NAMESPACE}}}identifier"
_REVISION_TAG = f"{{{_NAMESPACE}}}revision"
_ACTION_TAG = f"{{{_NAMESPACE}}}action"
_PRIORITY_TAG = f"{{{_NAMESPACE}}}priority"
_REASON_TAG = f"{{{_NAMESPACE}}}reason"

_IDENTIFIER = re.compile(r"[a-z][a-z0-9_-]{0,31}", re.ASCII)
_CANONICAL_INTEGER = re.compile(r"(?:0|[1-9][0-9]{0,6})", re.ASCII)


class XmlEnvelopeError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class Accepted:
    identifier: str
    revision: int


@dataclass(frozen=True, slots=True)
class ActionNeeded:
    identifier: str
    action: str
    priority: int


@dataclass(frozen=True, slots=True)
class Rejected:
    identifier: str
    reason: str


Envelope: TypeAlias = Accepted | ActionNeeded | Rejected


def _invalid() -> Never:
    raise XmlEnvelopeError("invalid XML envelope")


def _add_text_to_budget(text: str | None, total: int) -> int:
    if text is None:
        return total
    try:
        size = len(text.encode("utf-8"))
    except UnicodeEncodeError:
        _invalid()
    if size > _MAX_TEXT_NODE_BYTES or size > _MAX_TOTAL_TEXT_BYTES - total:
        _invalid()
    return total + size


def _has_non_whitespace(text: str | None) -> bool:
    return text is not None and bool(text.strip(" \t\r\n"))


def _validate_tree_budget(root: Element) -> None:
    elements = 0
    attributes = 0
    text_bytes = 0
    pending = [(root, 1)]

    while pending:
        element, depth = pending.pop()
        elements += 1
        attributes += len(element.attrib)
        if elements > _MAX_ELEMENTS:
            _invalid()
        if depth > _MAX_DEPTH:
            _invalid()
        if attributes > _MAX_ATTRIBUTES:
            _invalid()

        text_bytes = _add_text_to_budget(element.text, text_bytes)
        text_bytes = _add_text_to_budget(element.tail, text_bytes)
        children = tuple(element)
        if children and (
            _has_non_whitespace(element.text)
            or any(_has_non_whitespace(child.tail) for child in children)
        ):
            _invalid()
        pending.extend((child, depth + 1) for child in reversed(children))


def _exact_leaves(root: Element, expected_tags: tuple[str, ...]) -> tuple[Element, ...]:
    children = tuple(root)
    child_tags = tuple(child.tag for child in children)
    if len(child_tags) != len(set(child_tags)):
        _invalid()
    if child_tags != expected_tags:
        _invalid()
    if any(child.attrib or len(child) for child in children):
        _invalid()
    return children


def _validated_identifier(element: Element) -> str:
    text = element.text
    if text is None or _IDENTIFIER.fullmatch(text) is None:
        _invalid()
    return text


def _validated_integer(
    element: Element,
    *,
    minimum: int,
    maximum: int,
) -> int:
    text = element.text
    if text is None or _CANONICAL_INTEGER.fullmatch(text) is None:
        _invalid()
    value = int(text)
    if not minimum <= value <= maximum:
        _invalid()
    return value


def parse_bounded_xml_envelope(payload: bytes) -> Envelope:
    if type(payload) is not bytes:
        raise TypeError("payload must be exact bytes")
    if not 1 <= len(payload) <= _MAX_SOURCE_BYTES:
        raise XmlEnvelopeError("XML envelope byte length is invalid")

    try:
        root = ElementTree.fromstring(
            payload,
            forbid_dtd=True,
            forbid_entities=True,
            forbid_external=True,
        )
    except (ParseError, DefusedXmlException, LookupError):
        raise XmlEnvelopeError("invalid XML envelope") from None

    _validate_tree_budget(root)
    if root.tag != _ROOT_TAG:
        _invalid()
    if set(root.attrib) != {"version", "kind"}:
        _invalid()
    if root.attrib["version"] != "1":
        _invalid()

    kind = root.attrib["kind"]
    if kind == "accepted":
        identifier, revision = _exact_leaves(
            root,
            (_IDENTIFIER_TAG, _REVISION_TAG),
        )
        return Accepted(
            identifier=_validated_identifier(identifier),
            revision=_validated_integer(
                revision,
                minimum=1,
                maximum=_MAX_REVISION,
            ),
        )

    if kind == "action-needed":
        identifier, action, priority = _exact_leaves(
            root,
            (_IDENTIFIER_TAG, _ACTION_TAG, _PRIORITY_TAG),
        )
        return ActionNeeded(
            identifier=_validated_identifier(identifier),
            action=_validated_identifier(action),
            priority=_validated_integer(
                priority,
                minimum=0,
                maximum=_MAX_PRIORITY,
            ),
        )

    if kind == "rejected":
        identifier, reason = _exact_leaves(
            root,
            (_IDENTIFIER_TAG, _REASON_TAG),
        )
        return Rejected(
            identifier=_validated_identifier(identifier),
            reason=_validated_identifier(reason),
        )

    _invalid()
```

## Example

```python
document = b"""\
<envelope xmlns="urn:example:bounded-envelope:1" version="1" kind="accepted">
  <identifier>record_7</identifier>
  <revision>12</revision>
</envelope>"""
accepted = parse_bounded_xml_envelope(document)

duplicate_child = b"""\
<envelope xmlns="urn:example:bounded-envelope:1" version="1" kind="accepted">
  <identifier>record_7</identifier>
  <identifier>record_8</identifier>
</envelope>"""
try:
    parse_bounded_xml_envelope(duplicate_child)
except XmlEnvelopeError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

assert (accepted, duplicate_rejected) == (Accepted("record_7", 12), True)
```

## Trade-offs and Limitations

The exact-byte input check and 65,536-byte ceiling run before parsing. The tree
walk is iterative and rejects any parsed tree beyond four elements, depth two,
two total attributes, 256 UTF-8 bytes per text node, or 1,024 UTF-8 text bytes
overall. Those maxima deliberately leave no room for recursive extension of the
accepted grammar.

Parser hardening is not a resource budget. `defusedxml` blocks DTD, entity, and
external-reference features, while the byte cap bounds source size; the element,
depth, attribute, and text checks happen only after `ElementTree` has allocated
the tree. Use a hardened streaming design that enforces limits while reading, or
an isolated parser with operating-system limits, when even a capped in-memory
parse is outside the threat model.

The parser treats whitespace-only character data around children as formatting
and rejects other mixed content. Child order, namespace, version, discriminator,
attributes, and leaf sets are exact. Identifiers are lowercase ASCII tokens of
one to 32 characters; integers use canonical unsigned decimal notation and
field-specific ranges. ElementTree discards comments and processing instructions,
so this validates the element, attribute, and character-data contract rather
than preserving lexical XML syntax.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Three-State JSON Response Envelope](../networking-protocols/parse-a-bounded-three-state-json-response-envelope.md)
- [Load a Bounded Protobuf Descriptor Set in Dependency Order](load-a-bounded-protobuf-descriptor-set-in-dependency-order.md)
- [Parse a Bounded Nested Bracket Tree](parse-a-bounded-nested-bracket-tree.md)
<!-- catalog:related:end -->
