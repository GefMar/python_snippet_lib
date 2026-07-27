---
title: "Build a Bounded File Manifest with Internal Symlink Aliases"
snippet_type: recipe
use_cases:
  - persistence
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../security-privacy/audit-symlinks-in-a-bounded-directory-tree.md
  - compare-bounded-apparent-sizes-of-two-file-tree-snapshots.md
  - fingerprint-a-bounded-flat-file-set-with-framed-sha-256.md
---

# Build a Bounded File Manifest with Internal Symlink Aliases

## Idea and Problem

Build a deterministic manifest by classifying a bounded directory tree before resolving symbolic-link aliases to tokenized regular files.

The two-phase design makes traversal order irrelevant: every regular file is
offered to a caller-supplied extractor first, and aliases are resolved only
after all accepted tokens are known. The result separates tokenized files,
flattened aliases, and entries that the extractor deliberately did not
recognize.

## When to Use

Use this recipe for a locally controlled, quiescent resource directory whose
regular files can be described by a short printable token, such as a format or
compatibility identifier. Choose entry, depth, path, and token limits from the
expected directory shape before scanning unfamiliar content.

Use a plain symlink audit when no file-specific token is needed. Do not use a
path-based manifest as an authorization boundary for a tree that another
process can modify during the scan.

## Implementation

```python
import os
import stat
from collections.abc import Callable
from dataclasses import dataclass
from itertools import islice
from pathlib import Path
from typing import Literal


EntryKind = Literal["directory", "file", "symlink"]
UnsupportedReason = Literal[
    "extractor-returned-none",
    "target-produced-no-token",
]

_MAX_ENTRIES = 100_000
_MAX_DEPTH = 128
_MAX_PATH_BYTES = 65_536
_MAX_TOKEN_BYTES = 65_536


class ManifestError(ValueError):
    pass


class ManifestLimitError(ManifestError):
    pass


@dataclass(frozen=True, slots=True)
class ManifestFile:
    path: str
    token: str


@dataclass(frozen=True, slots=True)
class ManifestAlias:
    path: str
    target: str
    token: str


@dataclass(frozen=True, slots=True)
class ManifestUnsupported:
    path: str
    reason: UnsupportedReason


@dataclass(frozen=True, slots=True)
class FileManifest:
    files: tuple[ManifestFile, ...]
    aliases: tuple[ManifestAlias, ...]
    unsupported: tuple[ManifestUnsupported, ...]


@dataclass(frozen=True, slots=True)
class _DiscoveredEntry:
    path: Path
    kind: EntryKind
    link_target: str | None = None


def _bounded_manifest_integer(
    value: int,
    *,
    name: str,
    minimum: int,
    maximum: int,
) -> int:
    if isinstance(value, bool) or not isinstance(value, int):
        raise TypeError(f"{name} must be an integer")
    if not minimum <= value <= maximum:
        raise ValueError(f"{name} is outside the supported range")
    return value


def _checked_manifest_text(
    value: str,
    *,
    description: str,
    max_bytes: int,
) -> bytes:
    try:
        encoded = value.encode("utf-8", errors="strict")
    except UnicodeEncodeError as error:
        raise ManifestError(f"{description} is not valid Unicode text") from error
    if not value or not value.isprintable():
        raise ManifestError(f"{description} must be nonempty printable text")
    if len(encoded) > max_bytes:
        raise ManifestLimitError(f"{description} exceeds its byte limit")
    return encoded


def _resolve_alias(
    alias_parts: tuple[str, ...],
    entries: dict[tuple[str, ...], _DiscoveredEntry],
    *,
    root: Path,
    max_path_bytes: int,
) -> tuple[str, ...]:
    alias = entries[alias_parts]
    try:
        resolved = alias.path.resolve(strict=True)
    except RuntimeError as error:
        raise ManifestError("a symbolic-link cycle was found") from error
    except OSError as error:
        raise ManifestError("a symbolic link is broken or unreadable") from error

    try:
        relative = resolved.relative_to(root)
    except ValueError as error:
        raise ManifestError("a symbolic link points outside the root") from error
    target_parts = relative.parts
    if not target_parts:
        raise ManifestError("a symbolic link resolves to the root directory")
    _checked_manifest_text(
        relative.as_posix(),
        description="a resolved symbolic-link path",
        max_bytes=max_path_bytes,
    )

    target = entries.get(target_parts)
    if target is None:
        raise ManifestError("a symbolic link target was not discovered")
    if target.kind == "directory":
        raise ManifestError("a symbolic link resolves to a directory")
    if target.kind != "file":
        raise ManifestError("a symbolic link did not resolve to a regular file")
    return target_parts


def build_file_manifest(
    root: Path,
    extractor: Callable[[Path], str | None],
    *,
    max_entries: int = 10_000,
    max_depth: int = 32,
    max_path_bytes: int = 4_096,
    max_token_bytes: int = 256,
) -> FileManifest:
    max_entries = _bounded_manifest_integer(
        max_entries,
        name="max_entries",
        minimum=0,
        maximum=_MAX_ENTRIES,
    )
    max_depth = _bounded_manifest_integer(
        max_depth,
        name="max_depth",
        minimum=0,
        maximum=_MAX_DEPTH,
    )
    max_path_bytes = _bounded_manifest_integer(
        max_path_bytes,
        name="max_path_bytes",
        minimum=1,
        maximum=_MAX_PATH_BYTES,
    )
    max_token_bytes = _bounded_manifest_integer(
        max_token_bytes,
        name="max_token_bytes",
        minimum=1,
        maximum=_MAX_TOKEN_BYTES,
    )
    if not callable(extractor):
        raise TypeError("extractor must be callable")

    root = Path(root)
    metadata = os.stat(root, follow_symlinks=False)
    if not stat.S_ISDIR(metadata.st_mode):
        raise ManifestError("root must be a real directory")
    root = root.resolve(strict=True)

    entries: dict[tuple[str, ...], _DiscoveredEntry] = {}
    pending: list[tuple[Path, tuple[str, ...]]] = [(root, ())]
    while pending:
        directory, parent_parts = pending.pop()
        remaining_entries = max_entries - len(entries)
        with os.scandir(directory) as directory_entries:
            children = list(islice(directory_entries, remaining_entries + 1))
        if len(children) > remaining_entries:
            raise ManifestLimitError("max_entries was exceeded")

        prepared: list[tuple[bytes, os.DirEntry[str], tuple[str, ...]]] = []
        for child in children:
            child_parts = (*parent_parts, child.name)
            relative = "/".join(child_parts)
            sort_key = _checked_manifest_text(
                relative,
                description="a manifest path",
                max_bytes=max_path_bytes,
            )
            prepared.append((sort_key, child, child_parts))

        child_directories: list[tuple[Path, tuple[str, ...]]] = []
        for _, child, child_parts in sorted(prepared, key=lambda item: item[0]):
            if len(child_parts) > max_depth:
                raise ManifestLimitError("max_depth was exceeded")

            child_metadata = child.stat(follow_symlinks=False)
            child_path = Path(child.path)
            if stat.S_ISDIR(child_metadata.st_mode):
                entry = _DiscoveredEntry(child_path, "directory")
                child_directories.append((child_path, child_parts))
            elif stat.S_ISREG(child_metadata.st_mode):
                entry = _DiscoveredEntry(child_path, "file")
            elif stat.S_ISLNK(child_metadata.st_mode):
                target = os.readlink(child_path)
                _checked_manifest_text(
                    target,
                    description="a symbolic-link target",
                    max_bytes=max_path_bytes,
                )
                entry = _DiscoveredEntry(child_path, "symlink", target)
            else:
                raise ManifestError("the tree contains a special file")
            entries[child_parts] = entry

        pending.extend(reversed(child_directories))

    files: list[ManifestFile] = []
    unsupported: list[ManifestUnsupported] = []
    tokens: dict[tuple[str, ...], str] = {}
    for parts, entry in sorted(entries.items()):
        if entry.kind != "file":
            continue
        relative = "/".join(parts)
        token = extractor(entry.path)
        if token is None:
            unsupported.append(
                ManifestUnsupported(relative, "extractor-returned-none")
            )
            continue
        if not isinstance(token, str):
            raise TypeError("extractor must return a string or None")
        _checked_manifest_text(
            token,
            description="an extracted token",
            max_bytes=max_token_bytes,
        )
        if token != token.strip():
            raise ManifestError("an extracted token has surrounding whitespace")
        tokens[parts] = token
        files.append(ManifestFile(relative, token))

    aliases: list[ManifestAlias] = []
    for parts, entry in sorted(entries.items()):
        if entry.kind != "symlink":
            continue
        relative = "/".join(parts)
        target_parts = _resolve_alias(
            parts,
            entries,
            root=root,
            max_path_bytes=max_path_bytes,
        )
        token = tokens.get(target_parts)
        if token is None:
            unsupported.append(
                ManifestUnsupported(relative, "target-produced-no-token")
            )
            continue
        aliases.append(
            ManifestAlias(relative, "/".join(target_parts), token)
        )

    return FileManifest(
        files=tuple(files),
        aliases=tuple(aliases),
        unsupported=tuple(sorted(unsupported, key=lambda item: item.path)),
    )
```

## Example

```python
from tempfile import TemporaryDirectory


def read_abi_token(path: Path) -> str | None:
    text = path.read_text(encoding="utf-8")
    return text.removeprefix("ABI:") if text.startswith("ABI:") else None


with TemporaryDirectory() as temporary_directory:
    root = Path(temporary_directory) / "bundle"
    (root / "lib").mkdir(parents=True)

    # Both aliases sort before their eventual target.
    (root / "00-current").symlink_to("05-forward")
    (root / "05-forward").symlink_to("lib/module.bin")
    (root / "lib" / "module.bin").write_text("ABI:cp314", encoding="utf-8")
    (root / "notes.txt").write_text("documentation", encoding="utf-8")

    manifest = build_file_manifest(root, read_abi_token, max_entries=8)

assert manifest == FileManifest(
    files=(ManifestFile("lib/module.bin", "cp314"),),
    aliases=(
        ManifestAlias("00-current", "lib/module.bin", "cp314"),
        ManifestAlias("05-forward", "lib/module.bin", "cp314"),
    ),
    unsupported=(
        ManifestUnsupported("notes.txt", "extractor-returned-none"),
    ),
)
```

## Trade-offs and Limitations

Discovery, records, and sorting require `O(n)` memory and `O(n log n)` time;
resolving many long alias chains can add further work. The extractor controls
file reads and may impose its own size, format, and error policy. Its
exceptions propagate, and `None` means only that the file is unsupported by
that extractor, not that the file is safe or invalid.

The walk is not atomic. It assumes a quiescent real root; replacements,
retargeted links, and content changes can create a mixed-time result or a
check/use race. Traversal never descends through directory links; link
resolution follows host filesystem semantics only to classify the final
target. Broken, cyclic, external, directory-targeting, and special-file
entries abort the whole scan. Hard links remain separate paths, mount
boundaries are crossed, and the manifest preserves host filesystem naming
semantics rather than defining a cross-platform normalization format.

## Related Snippets

<!-- catalog:related:start -->
- [Audit Symlinks in a Bounded Directory Tree](../security-privacy/audit-symlinks-in-a-bounded-directory-tree.md)
- [Compare Bounded Apparent Sizes of Two File-Tree Snapshots](compare-bounded-apparent-sizes-of-two-file-tree-snapshots.md)
- [Fingerprint a Bounded Flat File Set with Framed SHA-256](fingerprint-a-bounded-flat-file-set-with-framed-sha-256.md)
<!-- catalog:related:end -->
