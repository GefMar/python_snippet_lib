---
title: "Branch SHA-256 Digests After One Shared Prefix"
snippet_type: recipe
use_cases:
  - data-transformation
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - build-bounded-digest-summaries-across-explicit-lookback-horizons.md
  - ../storage-databases/store-bytes-by-their-content-digest.md
---

# Branch SHA-256 Digests After One Shared Prefix

## Idea and Problem

Calculate digests for several byte strings without hashing their shared prefix for each variant.

Feed the common chunks into one hash object. For each suffix, copy the
in-progress state, update the copy, and take its digest. The original
state stays at the end of the prefix, so the suffixes are independent.

## When to Use

Use this when several already encoded variants have an identical leading
byte sequence, such as a shared binary body with alternative trailers.
Pass the prefix as finite chunks in byte order and provide each suffix as a
byte string. Digests are yielded in suffix order as raw bytes.

## Implementation

```python
from collections.abc import Iterable, Iterator
from hashlib import sha256


def sha256_variants(prefix_chunks: Iterable[bytes], suffixes: Iterable[bytes]) -> Iterator[bytes]:
    prefix_state = sha256()
    for chunk in prefix_chunks:
        prefix_state.update(chunk)

    for suffix in suffixes:
        branch = prefix_state.copy()
        branch.update(suffix)
        yield branch.digest()
```

## Example

```python
body = bytes(range(128))
prefix_chunks = (b"sample block\n", body[:61], body[61:])
trailers = (b"", b"\nmode=preview", b"\nmode=final")
digests = tuple(sha256_variants(iter(prefix_chunks), iter(trailers)))

prefix = b"".join(prefix_chunks)
assert digests == tuple(sha256(prefix + trailer).digest() for trailer in trailers)
```

## Trade-offs and Limitations

The iterator consumes the entire prefix before yielding anything. It does
not retain or replay the chunks, but each branch still hashes its own suffix.
Hash each variant directly if the prefix is short or there is only one
variant. Separate hashing calls do not encode field boundaries, so serialize
structured fields unambiguously before passing their bytes. These unkeyed
digests do not authenticate their inputs.

## Related Snippets

<!-- catalog:related:start -->
- [Build Bounded Digest Summaries Across Explicit Lookback Horizons](build-bounded-digest-summaries-across-explicit-lookback-horizons.md)
- [Store Bytes by Their Content Digest](../storage-databases/store-bytes-by-their-content-digest.md)
<!-- catalog:related:end -->
