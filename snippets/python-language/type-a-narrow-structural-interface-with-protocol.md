---
title: "Type a Narrow Structural Interface with Protocol"
snippet_type: pattern
use_cases:
  - interoperability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - dispatch-named-strategies-with-an-explicit-function-mapping.md
  - ../networking-protocols/read-and-write-size-capped-varint-frames.md
---

# Type a Narrow Structural Interface with Protocol

## Idea and Problem

Describe one consumer-owned capability structurally so compatible adapters do not need to inherit from a shared base class.

A `Protocol` lets a static type checker accept any object with the declared
method signature. The consuming function depends on only that small interface,
while concrete adapters remain free to use their own class hierarchy or no
hierarchy at all.

## When to Use

Use this pattern at a module boundary when a consumer needs one narrow
capability and several existing or third-party objects can provide it. It is
most useful when mypy, pyright, or another static checker runs in the project.
Prefer an abstract base class when shared runtime behavior, registration, or
enforced inheritance is the actual requirement.

## Implementation

```python
from collections.abc import Iterable, Mapping
from typing import Protocol


class RecordEncoder(Protocol):
    def encode(self, record: Mapping[str, object], /) -> bytes:
        ...


def encode_records(
    records: Iterable[Mapping[str, object]],
    encoder: RecordEncoder,
) -> tuple[bytes, ...]:
    encoded: list[bytes] = []
    for record in records:
        encoded.append(encoder.encode(record))
    return tuple(encoded)
```

## Example

```python
import json


class JsonLineEncoder:
    def encode(self, record: Mapping[str, object], /) -> bytes:
        text = json.dumps(dict(record), sort_keys=True, separators=(",", ":"))
        return text.encode("utf-8") + b"\n"


encoder = JsonLineEncoder()
result = encode_records(
    [{"label": "alpha", "score": 7}, {"label": "beta"}],
    encoder,
)

assert result == (
    b'{"label":"alpha","score":7}\n',
    b'{"label":"beta"}\n',
)
```

## Trade-offs and Limitations

Python does not enforce this annotation during an ordinary call; the example
works through normal duck typing, while signature compatibility is checked by
external static analysis. This protocol deliberately omits
`runtime_checkable`: runtime protocol checks only establish the presence of
members, not their annotated signatures or behavior. Expanding a widely used
protocol can break otherwise independent adapters. The helper also
materializes every encoded value, propagates encoder exceptions, and cannot
roll back any adapter side effects completed before a failure. The example
adapter additionally requires every nested value to be JSON-serializable.

## Related Snippets

<!-- catalog:related:start -->
- [Dispatch Named Strategies with an Explicit Function Mapping](dispatch-named-strategies-with-an-explicit-function-mapping.md)
- [Read and Write Size-Capped Varint Frames](../networking-protocols/read-and-write-size-capped-varint-frames.md)
<!-- catalog:related:end -->
