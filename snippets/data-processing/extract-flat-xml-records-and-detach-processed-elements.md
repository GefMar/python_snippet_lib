---
title: "Extract Flat XML Records and Detach Processed Elements"
snippet_type: recipe
use_cases:
  - parsing
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/canonicalize-trusted-size-capped-xml-for-stable-utf-8-comparison.md
  - ../configuration-serialization/parse-a-bounded-xml-envelope-with-closed-variant-dispatch.md
---

# Extract Flat XML Records and Detach Processed Elements

## Idea and Problem

Read complete XML records incrementally without keeping a tree of processed siblings.

At each record's end event, copy the needed values into a tuple and remove
that element from its parent. Calling `clear()` on the element alone would
still leave an empty child attached to the parent.

## When to Use

Use this for trusted, size-bounded input with a known flat shape: an
`inventory` root contains leaf `item` elements with a `code` attribute and
text label. Use input without namespaces or DTDs. The caller owns the open
binary stream and must keep it open during iteration.

## Implementation

```python
from collections.abc import Iterator
from typing import BinaryIO
from xml.etree import ElementTree


def iter_inventory(source: BinaryIO) -> Iterator[tuple[str, str]]:
    events = ElementTree.iterparse(source, events=("start", "end"))
    try:
        _, root = next(events)
        if root.tag != "inventory":
            raise ValueError("expected an inventory root")

        depth = 1
        for event, element in events:
            if event == "start":
                depth += 1
                if depth != 2 or element.tag != "item":
                    raise ValueError("expected one level of item elements")
                continue

            depth -= 1
            if element is root:
                continue
            code, label = element.get("code"), element.text
            if not code or label is None:
                raise ValueError("item needs a code and text label")

            root.remove(element)
            yield code, label
    finally:
        events.close()
```

## Example

```python
from contextlib import closing
from io import BytesIO

document = b"""<inventory>
  <item code="A1">Notebook</item>
  <item code="B2">Binder clip</item>
</inventory>"""

with BytesIO(document) as source:
    with closing(iter_inventory(source)) as records:
        complete = tuple(records)

    source.seek(0)
    with closing(iter_inventory(source)) as records:
        first = next(records)
    still_open = not source.closed

assert (complete, first, still_open) == (
    (("A1", "Notebook"), ("B2", "Binder clip")),
    ("A1", "Notebook"),
    True,
)
```

## Trade-offs and Limitations

The blocking parser can read ahead, and it holds the active record until its
end. Detaching processed nodes limits tree retention but does not impose a
memory cap. Keep the input and individual records bounded.

If you stop early, close the iterator explicitly. Closing it does not close
the borrowed stream, validate the remaining XML, or leave the stream at a
record boundary. Do not use this recipe as a parser for untrusted XML.

## Related Snippets

<!-- catalog:related:start -->
- [Canonicalize Trusted Size-Capped XML for Stable UTF-8 Comparison](../configuration-serialization/canonicalize-trusted-size-capped-xml-for-stable-utf-8-comparison.md)
- [Parse a Bounded XML Envelope with Closed Variant Dispatch](../configuration-serialization/parse-a-bounded-xml-envelope-with-closed-variant-dispatch.md)
<!-- catalog:related:end -->
