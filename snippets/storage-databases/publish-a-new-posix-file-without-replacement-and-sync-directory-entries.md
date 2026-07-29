---
title: "Publish a New POSIX File Without Replacement and Sync Directory Entries"
snippet_type: recipe
use_cases:
  - persistence
  - resource-management
  - concurrency-control
tested_python:
  - "3.14"
dependencies: []
related:
  - replace-a-file-atomically-with-a-sibling-temporary-file.md
  - split-a-binary-stream-into-exclusively-created-numbered-parts.md
  - store-bytes-by-their-content-digest.md
---

# Publish a New POSIX File Without Replacement and Sync Directory Entries

## Idea and Problem

Publish one complete new file with no-replacement semantics, then explicitly synchronize both the new link and removal of its sibling temporary directory entry.

The payload is written and synchronized through an exclusively created
temporary file in the target directory. A same-directory hard link makes the
prepared inode visible under the target name only if that name does not already
exist. Directory synchronization occurs once after linking and again after
removing the temporary name.

## When to Use

Use this recipe in a trusted, existing directory on a local POSIX filesystem
whose documentation supports hard links and directory `fsync`. It fits bounded
immutable artifacts where concurrent creators may race but an existing target
must never be replaced.

Use a database, object-store conditional create, or filesystem-specific
transaction when stronger portable recovery semantics are required. The
explicit synchronization calls are operations the filesystem may honor; they
are not a universal guarantee about every device, mount option, remote
filesystem, or power-loss path.

## Implementation

```python
import os
import re
import secrets
import stat
from enum import StrEnum
from pathlib import Path
from tempfile import TemporaryDirectory

_MAX_PAYLOAD_BYTES = 16 << 20
_TEMPORARY_NAME_ATTEMPTS = 32
_TARGET_NAME = re.compile(
    r"[A-Za-z0-9][A-Za-z0-9._-]{0,127}",
    re.ASCII,
)


class PublishResult(StrEnum):
    CREATED = "created"
    EXISTS = "exists"


class PublicationIndeterminateError(OSError):
    pass


def _check_posix_directory_operations() -> None:
    required = {os.open, os.link, os.unlink}
    if (
        os.name != "posix"
        or not hasattr(os, "O_DIRECTORY")
        or not required <= os.supports_dir_fd
    ):
        raise NotImplementedError(
            "POSIX directory-descriptor operations are required"
        )


def _create_temporary(dir_fd: int) -> tuple[str, int]:
    flags = os.O_WRONLY | os.O_CREAT | os.O_EXCL
    flags |= getattr(os, "O_CLOEXEC", 0)
    for _ in range(_TEMPORARY_NAME_ATTEMPTS):
        name = f".publish-{secrets.token_hex(16)}.tmp"
        try:
            descriptor = os.open(
                name,
                flags,
                0o600,
                dir_fd=dir_fd,
            )
        except FileExistsError:
            continue
        try:
            os.fchmod(descriptor, 0o600)
            return name, descriptor
        except BaseException:
            try:
                os.close(descriptor)
            except BaseException:
                pass
            try:
                os.unlink(name, dir_fd=dir_fd)
            except BaseException:
                pass
            raise
    raise FileExistsError("could not allocate an exclusive temporary name")


def _write_all(descriptor: int, payload: bytes) -> None:
    remaining = memoryview(payload)
    while remaining:
        written = os.write(descriptor, remaining)
        if written <= 0:
            raise OSError("file write made no progress")
        remaining = remaining[written:]


def publish_new_posix_file(
    directory: Path,
    name: str,
    payload: bytes,
) -> PublishResult:
    _check_posix_directory_operations()
    if not isinstance(directory, Path):
        raise TypeError("directory must be a Path")
    if type(name) is not str:
        raise TypeError("name must be exact text")
    if _TARGET_NAME.fullmatch(name) is None:
        raise ValueError("name is outside the conservative target profile")
    if type(payload) is not bytes:
        raise TypeError("payload must be exact bytes")
    if len(payload) > _MAX_PAYLOAD_BYTES:
        raise ValueError("payload exceeds the supported byte length")

    directory_flags = os.O_RDONLY | os.O_DIRECTORY
    directory_flags |= getattr(os, "O_CLOEXEC", 0)
    dir_fd = os.open(directory, directory_flags)
    temporary_name: str | None = None
    descriptor = -1
    publication_may_exist = False
    try:
        temporary_name, descriptor = _create_temporary(dir_fd)
        _write_all(descriptor, payload)
        os.fsync(descriptor)
        try:
            os.close(descriptor)
        finally:
            descriptor = -1

        target_existed = False
        try:
            os.link(
                temporary_name,
                name,
                src_dir_fd=dir_fd,
                dst_dir_fd=dir_fd,
            )
            publication_may_exist = True
        except FileExistsError:
            target_existed = True
        except OSError:
            raise
        except BaseException as error:
            raise PublicationIndeterminateError(
                "target publication may have succeeded"
            ) from error

        if target_existed:
            os.unlink(temporary_name, dir_fd=dir_fd)
            temporary_name = None
            os.fsync(dir_fd)
            result = PublishResult.EXISTS
        else:
            os.fsync(dir_fd)
            os.unlink(temporary_name, dir_fd=dir_fd)
            temporary_name = None
            os.fsync(dir_fd)
            result = PublishResult.CREATED

        try:
            os.close(dir_fd)
        finally:
            dir_fd = -1
        return result
    except PublicationIndeterminateError:
        raise
    except BaseException as error:
        if publication_may_exist:
            raise PublicationIndeterminateError(
                "target publication may have succeeded"
            ) from error
        raise
    finally:
        if descriptor >= 0:
            try:
                os.close(descriptor)
            except BaseException:
                pass
        if temporary_name is not None:
            try:
                os.unlink(temporary_name, dir_fd=dir_fd)
            except BaseException:
                pass
        if dir_fd >= 0:
            try:
                os.close(dir_fd)
            except BaseException:
                pass
```

## Example

```python
with TemporaryDirectory() as temporary_directory:
    root = Path(temporary_directory)
    target = root / "snapshot.bin"

    first = publish_new_posix_file(root, target.name, b"first version")
    second = publish_new_posix_file(root, target.name, b"second version")
    observed = (
        first,
        second,
        target.read_bytes(),
        stat.S_IMODE(target.stat().st_mode),
        tuple(root.glob(".publish-*.tmp")),
    )

assert observed == (
    PublishResult.CREATED,
    PublishResult.EXISTS,
    b"first version",
    0o600,
    (),
)
```

## Trade-offs and Limitations

The function holds the caller's bounded payload and performs a linear write.
The temporary and target are in one directory, so hard-link publication is an
atomic no-replacement name operation on supported local filesystems. Exactly
one of several cooperating concurrent creators can link the target; a loser
observes `exists` and does not alter that target.

An asynchronous exception raised by the link call is conservatively reported as
indeterminate because the target may have become visible before control returns
to Python. Once the call returns, every failure through directory-descriptor
close receives the same classification. Definite `FileExistsError` and other
ordinary system errors reported directly by the link call remain pre-publication
outcomes.

Best-effort failure cleanup cannot remove temporaries left by process
termination or a machine crash. The recipe does not claim portable power-loss
durability, operate on Windows, cross filesystems, preserve ownership, ACLs,
extended attributes, or timestamps, publish multiple files, or protect an
untrusted directory from entry replacement and permission races.

## Related Snippets

<!-- catalog:related:start -->
- [Replace a File Atomically with a Sibling Temporary File](replace-a-file-atomically-with-a-sibling-temporary-file.md)
- [Split a Binary Stream into Exclusively Created Numbered Parts](split-a-binary-stream-into-exclusively-created-numbered-parts.md)
- [Store Bytes by Their Content Digest](store-bytes-by-their-content-digest.md)
<!-- catalog:related:end -->
