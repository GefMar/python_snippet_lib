---
title: "Read One Descriptor-Relative Regular File Without Following Its Final Symlink"
snippet_type: recipe
use_cases:
  - resource-management
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - audit-symlinks-in-a-bounded-directory-tree.md
  - ../storage-databases/publish-a-new-posix-file-without-replacement-and-sync-directory-entries.md
  - validate-a-conservative-unicode-filename-component.md
---

# Read One Descriptor-Relative Regular File Without Following Its Final Symlink

## Idea and Problem

Read one bounded regular file relative to a trusted directory descriptor without following a final-component symbolic link.

The helper accepts one bounded ASCII basename, verifies that the
caller-owned root descriptor still names a directory, and opens the leaf with
descriptor-relative no-follow, nonblocking, and close-on-exec flags. It checks
the opened descriptor itself for a regular file, then reads at most one byte
beyond the caller's cap so a file that grows after inspection still fails
closed.

## When to Use

Use this recipe on a POSIX platform when trusted code already owns an open
descriptor for the intended directory and needs to read one small leaf by
name. The name must be an exact ASCII basename from 1 through 255 bytes, and
the accepted payload limit must be an exact integer from 1 through 1,048,576
bytes.

Treat acquisition and lifetime of the root descriptor as a separate trusted
operation. This helper is useful when path replacement must not redirect a
leaf read through a symbolic link, but it is not a complete hostile-filesystem
sandbox.

## Implementation

```python
import os
import stat
from pathlib import Path
from tempfile import TemporaryDirectory

MAX_DESCRIPTOR_RELATIVE_FILE_BYTES = 1_048_576
_MAX_BASENAME_BYTES = 255
_REQUIRED_OPEN_FLAGS = ("O_NOFOLLOW", "O_NONBLOCK", "O_CLOEXEC")


class DescriptorRelativeReadError(ValueError):
    pass


class InvalidDescriptorRelativeBasenameError(DescriptorRelativeReadError):
    pass


class DescriptorRelativeNonRegularFileError(DescriptorRelativeReadError):
    pass


class DescriptorRelativeFileTooLargeError(DescriptorRelativeReadError):
    pass


def _require_descriptor_relative_open() -> None:
    supported_dir_fd_functions = getattr(os, "supports_dir_fd", ())
    if (
        os.name != "posix"
        or os.open not in supported_dir_fd_functions
        or any(not hasattr(os, flag_name) for flag_name in _REQUIRED_OPEN_FLAGS)
    ):
        raise NotImplementedError(
            "descriptor-relative no-follow regular-file reads are unsupported"
        )


def _validate_ascii_basename(basename: object) -> str:
    if type(basename) is not str:
        raise TypeError("basename must be exact text")
    if not 1 <= len(basename) <= _MAX_BASENAME_BYTES:
        raise InvalidDescriptorRelativeBasenameError(
            "basename length is outside the supported range"
        )
    try:
        encoded = basename.encode("ascii")
    except UnicodeEncodeError as error:
        raise InvalidDescriptorRelativeBasenameError(
            "basename must contain only ASCII"
        ) from error
    if not 1 <= len(encoded) <= _MAX_BASENAME_BYTES:
        raise InvalidDescriptorRelativeBasenameError(
            "basename length is outside the supported range"
        )
    if basename in {".", ".."} or "/" in basename or "\0" in basename:
        raise InvalidDescriptorRelativeBasenameError(
            "basename must identify one non-dot POSIX path component"
        )
    return basename


def read_regular_file_at(
    directory_fd: int,
    basename: str,
    *,
    max_bytes: int,
) -> bytes:
    """Read one bounded regular leaf without taking ownership of directory_fd."""
    _require_descriptor_relative_open()
    if type(directory_fd) is not int:
        raise TypeError("directory_fd must be an exact integer")
    if directory_fd < 0:
        raise ValueError("directory_fd must be non-negative")
    basename = _validate_ascii_basename(basename)
    if type(max_bytes) is not int:
        raise TypeError("max_bytes must be an exact integer")
    if not 1 <= max_bytes <= MAX_DESCRIPTOR_RELATIVE_FILE_BYTES:
        raise ValueError("max_bytes is outside the supported range")

    root_metadata = os.fstat(directory_fd)
    if not stat.S_ISDIR(root_metadata.st_mode):
        raise NotADirectoryError("directory_fd must refer to a directory")

    flags = os.O_RDONLY | os.O_NOFOLLOW | os.O_NONBLOCK | os.O_CLOEXEC
    leaf_fd = os.open(basename, flags, dir_fd=directory_fd)
    try:
        leaf_metadata = os.fstat(leaf_fd)
        if not stat.S_ISREG(leaf_metadata.st_mode):
            raise DescriptorRelativeNonRegularFileError("opened leaf is not a regular file")
        if leaf_metadata.st_size > max_bytes:
            raise DescriptorRelativeFileTooLargeError("regular file exceeds max_bytes")

        chunks: list[bytes] = []
        total_bytes = 0
        while total_bytes <= max_bytes:
            chunk = os.read(leaf_fd, max_bytes + 1 - total_bytes)
            if not chunk:
                break
            chunks.append(chunk)
            total_bytes += len(chunk)
            if total_bytes > max_bytes:
                raise DescriptorRelativeFileTooLargeError("regular file exceeds max_bytes")
        return b"".join(chunks)
    finally:
        os.close(leaf_fd)
```

## Example

```python
with TemporaryDirectory() as temporary_directory:
    root = Path(temporary_directory)
    (root / "payload.bin").write_bytes(b"north\x00south")
    (root / "large.bin").write_bytes(b"12345")
    (root / "folder").mkdir()
    (root / "alias.bin").symlink_to("payload.bin")

    root_fd = os.open(root, os.O_RDONLY | os.O_CLOEXEC)
    try:
        payload = read_regular_file_at(root_fd, "payload.bin", max_bytes=11)

        try:
            read_regular_file_at(root_fd, "large.bin", max_bytes=4)
        except DescriptorRelativeFileTooLargeError:
            oversized_rejected = True
        else:
            oversized_rejected = False

        try:
            read_regular_file_at(root_fd, "folder", max_bytes=16)
        except DescriptorRelativeNonRegularFileError:
            nonregular_rejected = True
        else:
            nonregular_rejected = False

        try:
            read_regular_file_at(root_fd, "alias.bin", max_bytes=16)
        except OSError:
            final_symlink_rejected = True
        else:
            final_symlink_rejected = False

        root_descriptor_survived = stat.S_ISDIR(os.fstat(root_fd).st_mode)
    finally:
        os.close(root_fd)

assert (
    payload,
    oversized_rejected,
    nonregular_rejected,
    final_symlink_rejected,
    root_descriptor_survived,
) == (b"north\x00south", True, True, True, True)
```

## Trade-offs and Limitations

The caller retains ownership of the root descriptor; the helper closes only
the leaf descriptor it opens. `O_NONBLOCK` prevents a substituted FIFO from
blocking during open on common POSIX systems, while the descriptor type check
rejects every non-regular leaf. Opening a special file can itself have
platform-specific effects, however, so the supplied directory must remain a
trusted boundary. Unsupported platforms or missing flags fail with
`NotImplementedError`; filesystem `OSError` exceptions propagate unchanged.

Only one final component is accepted and protected. The helper does not obtain
the root descriptor, walk path components, authorize hard links, cross-check
mount boundaries, or defend against device-node behavior. `O_NOFOLLOW` does
not make a directory tree a general security boundary.

The open descriptor keeps later path replacement from redirecting this read,
but it does not create a stable content snapshot. Another writer can change or
truncate the same regular file while it is read. The initial size check is only
an early rejection; reading at most `max_bytes + 1` is the authoritative size
bound. The function materializes the accepted content and performs no text
decoding, checksum verification, locking, or retry.

## Related Snippets

<!-- catalog:related:start -->
- [Audit Symlinks in a Bounded Directory Tree](audit-symlinks-in-a-bounded-directory-tree.md)
- [Publish a New POSIX File Without Replacement and Sync Directory Entries](../storage-databases/publish-a-new-posix-file-without-replacement-and-sync-directory-entries.md)
- [Validate a Conservative Unicode Filename Component](validate-a-conservative-unicode-filename-component.md)
<!-- catalog:related:end -->
