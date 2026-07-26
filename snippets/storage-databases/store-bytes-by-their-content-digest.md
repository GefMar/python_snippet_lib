---
title: "Store Bytes by Their Content Digest"
snippet_type: recipe
use_cases:
  - caching
  - persistence
tested_python:
  - "3.14"
dependencies: []
related: []
---

# Store Bytes by Their Content Digest

## Idea and Problem

Store immutable byte payloads under paths derived from their SHA-256 digest so identical content shares one deterministic location.

The first digest bytes form shallow shard directories. A new payload is
written and flushed to a temporary file in the target directory before
`os.replace()` publishes it atomically. Lookups accept only canonical lowercase
hexadecimal digests and verify that the stored bytes still match their name.

## When to Use

Use this recipe for a trusted local cache or small immutable artifact store
when callers already hold each payload in memory. Deterministic names make
deduplication and cache references simple. The root directory must be owned by
the application and protected from untrusted filesystem modification.

## Implementation

```python
import hashlib
import os
import re
import tempfile
from pathlib import Path


_SHA256_HEX = re.compile(r"[0-9a-f]{64}\Z")


class ContentIntegrityError(OSError):
    pass


class ContentDigestStore:
    def __init__(self, root: Path) -> None:
        self.root = Path(root)

    def path_for_digest(self, digest: str) -> Path:
        if not isinstance(digest, str):
            raise TypeError("digest must be text")
        if _SHA256_HEX.fullmatch(digest) is None:
            raise ValueError("digest must be 64 lowercase hexadecimal characters")
        return self.root / digest[:2] / digest[2:4] / digest

    def put(self, data: bytes) -> str:
        if not isinstance(data, bytes):
            raise TypeError("data must be bytes")

        digest = hashlib.sha256(data).hexdigest()
        target = self.path_for_digest(digest)
        if target.is_file():
            self.get(digest)
            return digest
        if target.exists():
            raise OSError(f"content target is not a regular file: {target}")

        target.parent.mkdir(parents=True, exist_ok=True)
        descriptor, temporary_name = tempfile.mkstemp(
            prefix=f".{digest}.",
            dir=target.parent,
        )
        temporary = Path(temporary_name)
        try:
            with os.fdopen(descriptor, "wb") as output:
                output.write(data)
                output.flush()
                os.fsync(output.fileno())
            os.replace(temporary, target)
        finally:
            temporary.unlink(missing_ok=True)
        return digest

    def get(self, digest: str) -> bytes:
        target = self.path_for_digest(digest)
        data = target.read_bytes()
        if hashlib.sha256(data).hexdigest() != digest:
            raise ContentIntegrityError(f"content does not match digest: {digest}")
        return data
```

## Example

```python
from pathlib import Path
from tempfile import TemporaryDirectory


with TemporaryDirectory() as temporary_directory:
    store = ContentDigestStore(Path(temporary_directory))
    digest = store.put(b"hello")
    repeated_digest = store.put(b"hello")
    payload = store.get(digest)
    stored_files = sum(path.is_file() for path in store.root.rglob("*"))

    try:
        store.get("../outside")
    except ValueError:
        traversal_rejected = True
    else:
        traversal_rejected = False

    store.path_for_digest(digest).write_bytes(b"corrupted")
    try:
        store.get(digest)
    except ContentIntegrityError:
        corruption_detected = True
    else:
        corruption_detected = False

assert (
    digest,
    repeated_digest,
    payload,
    stored_files,
    traversal_rejected,
    corruption_detected,
) == (
    "2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824",
    "2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824",
    b"hello",
    1,
    True,
    True,
)
```

## Trade-offs and Limitations

Both `put()` and `get()` materialize the complete payload, and reads rehash it.
The implementation has no streaming, size limit, eviction, metadata, locking,
quota, or garbage-collection policy. Atomic replacement prevents readers from
seeing a partial file on the same filesystem, but directory entries are not
fsynced, so this is not a complete crash-durability protocol. A trusted root is
essential because symlinks or external writers can invalidate path assumptions.

## Related Snippets

<!-- catalog:related:start -->
No related snippets.
<!-- catalog:related:end -->
