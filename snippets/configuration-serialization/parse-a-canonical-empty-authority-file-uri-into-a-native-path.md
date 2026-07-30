---
title: "Parse a Canonical Empty-Authority File URI into a Native Path"
snippet_type: recipe
use_cases:
  - interoperability
  - parsing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - ../storage-databases/map-a-namespaced-posix-path-beneath-a-logical-local-root.md
  - ../security-privacy/plan-a-bounded-missing-posix-path-beneath-a-trusted-root-with-allow-missing.md
  - resolve-declared-input-paths-from-an-explicit-execution-mode.md
---

# Parse a Canonical Empty-Authority File URI into a Native Path

## Idea and Problem

Convert one canonical local file URI to a native absolute path without accepting alternate authorities or spellings.

`Path.from_uri` performs platform-aware conversion, including percent decoding,
but its authority behavior changed between Python 3.13 and 3.14. Requiring an
empty authority removes that version-dependent branch. Re-encoding the result
with `Path.as_uri()` then admits only the platform's canonical spelling and
rejects ambiguous percent escapes and other aliases.

## When to Use

Use this recipe at a small interoperability boundary that receives a local file
URI rather than a filesystem path. The accepted form starts with exactly
`file:///`, carries no authority, query, fragment, control character, parent
segment, or double-root spelling, and occupies at most 2,048 UTF-8 bytes.

Use a URL library for non-file resources. Apply a separate trusted-root and
symlink policy before opening the returned path when the caller must be confined
to a directory. Keep the URI itself when it must remain portable across hosts
with different path flavours or filesystem encodings.

## Implementation

```python
from pathlib import Path
from urllib.parse import urlsplit

_MAX_FILE_URI_BYTES = 2_048


class LocalFileURIError(ValueError):
    pass


def parse_canonical_local_file_uri(uri: str) -> Path:
    if type(uri) is not str:
        raise TypeError("uri must be an exact string")
    try:
        encoded = uri.encode("utf-8", errors="strict")
    except UnicodeEncodeError as error:
        raise LocalFileURIError("uri must be valid UTF-8 text") from error
    if not 1 <= len(encoded) <= _MAX_FILE_URI_BYTES:
        raise LocalFileURIError("uri byte size is outside the supported range")
    if any(not character.isprintable() for character in uri):
        raise LocalFileURIError("uri must not contain control characters")
    if not uri.startswith("file:///") or uri.startswith("file:////"):
        raise LocalFileURIError("uri must use canonical empty-authority file syntax")

    try:
        parts = urlsplit(uri)
    except ValueError as error:
        raise LocalFileURIError("uri syntax is invalid") from error
    if parts.scheme != "file" or parts.netloc or parts.query or parts.fragment:
        raise LocalFileURIError("uri must contain only an empty-authority file path")

    try:
        path = Path.from_uri(uri)
    except ValueError as error:
        raise LocalFileURIError("uri does not encode an absolute native path") from error
    path_text = str(path)
    try:
        path_text.encode("utf-8", errors="strict")
    except UnicodeEncodeError as error:
        raise LocalFileURIError("decoded path must be valid UTF-8 text") from error
    if any(not character.isprintable() for character in path_text):
        raise LocalFileURIError("decoded path must not contain control characters")
    if ".." in path.parts:
        raise LocalFileURIError("decoded path must not contain parent segments")
    if not path.is_absolute() or path.as_uri() != uri:
        raise LocalFileURIError("uri is not the canonical native path spelling")
    return path


```

## Example

```python
expected = Path.cwd() / "folder name" / "item.txt"
canonical = expected.as_uri()

parsed = parse_canonical_local_file_uri(canonical)

invalid_uris = (
    canonical.replace("file://", "file://localhost", 1),
    canonical.replace("%20", " "),
    canonical + "?mode=read",
    canonical + "#part",
    "file:///base/../item.txt",
    "file:////double-root/item.txt",
    "file:///item%00name",
    canonical.replace("folder%20name", "folder%2Fname"),
    canonical.replace("folder%20name", "%66older%20name"),
    canonical.replace("%20", "%2G"),
    "file:///item%FFname",
    "file:///item%C2%85name",
)
rejected = []
for invalid_uri in invalid_uris:
    try:
        parse_canonical_local_file_uri(invalid_uri)
    except LocalFileURIError:
        rejected.append(True)
    else:
        rejected.append(False)

assert (parsed, canonical, tuple(rejected)) == (
    expected,
    expected.as_uri(),
    (True, True, True, True, True, True, True, True, True, True, True, True),
)
```

## Trade-offs and Limitations

Parsing and validation are linear in at most 2,048 encoded bytes. The function
uses space within a constant multiple of that bound and performs no filesystem
access. Canonicality is native-platform lexical canonicality: a URI accepted
on one operating system is not promised to be accepted or to identify the same
path on another.

The empty-authority rule intentionally rejects `localhost`, the machine's host
name, UNC authorities, and other aliases even where `Path.from_uri` would accept
them. It also rejects raw non-ASCII path text, lowercase or unnecessary percent
escapes, encoded separators, non-UTF-8 filesystem bytes, parent traversal,
query and fragment data whenever they do not round-trip to the one spelling
produced by `Path.as_uri()`.

An absolute canonical path is not an authorization decision. The result may be
missing, may identify a symlink, and may escape a desired root through existing
filesystem links. Before opening it, apply descriptor-relative access or a
separate containment policy appropriate to the platform and threat model.

## Related Snippets

<!-- catalog:related:start -->
- [Map a Namespaced POSIX Path Beneath a Logical Local Root](../storage-databases/map-a-namespaced-posix-path-beneath-a-logical-local-root.md)
- [Plan a Bounded Missing POSIX Path Beneath a Trusted Root with Allow Missing](../security-privacy/plan-a-bounded-missing-posix-path-beneath-a-trusted-root-with-allow-missing.md)
- [Resolve Declared Input Paths from an Explicit Execution Mode](resolve-declared-input-paths-from-an-explicit-execution-mode.md)
<!-- catalog:related:end -->
