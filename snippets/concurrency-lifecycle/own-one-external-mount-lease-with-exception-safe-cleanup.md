---
title: "Own One External Mount Lease with Exception-Safe Cleanup"
snippet_type: pattern
use_cases:
  - lifecycle-management
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - elect-one-final-releaser-from-bounded-named-leases.md
  - run-one-async-operation-with-a-bounded-resource-stack.md
---

# Own One External Mount Lease with Exception-Safe Cleanup

## Idea and Problem

Own one externally acquired mount lease through a single-use context manager without embedding subprocess, network, or filesystem operations.

The owner derives one logical mount path from an absolute `PurePosixPath` root
and one validated child name. An injected acquisition callback returns an
opaque handle, which is exposed only inside a frozen lease after acquisition
succeeds. Cleanup always receives that exact lease. If cleanup fails, the
owner retains unresolved ownership and permits an explicit retry.

## When to Use

Use this pattern around a trusted adapter that already knows how to claim and
release one external mount. The adapter must reject a pre-existing claim
atomically, make acquisition failure-atomic, and make release retry-safe for
the same lease after an ambiguous cleanup failure.

Use the external system's native context manager when it already provides the
same ownership and error semantics. Use a transaction or durable reconciler
when lease recovery must survive process termination.

## Implementation

```python
import re
from collections.abc import Callable
from dataclasses import dataclass
from enum import Enum, auto
from pathlib import PurePosixPath
from types import TracebackType


_MAX_ROOT_BYTES = 4_096
_CHILD_NAME = re.compile(r"[a-z][a-z0-9_-]{0,63}", re.ASCII)


class MountLeaseState(Enum):
    NEW = auto()
    ACTIVE = auto()
    RELEASED = auto()
    CLEANUP_FAILED = auto()


class MountLeaseStateError(RuntimeError):
    pass


class MountPathClaimedError(RuntimeError):
    pass


@dataclass(frozen=True, slots=True)
class ExternalMountLease:
    path: PurePosixPath
    handle: object


AcquireMount = Callable[[PurePosixPath], object]
ReleaseMount = Callable[[ExternalMountLease], None]


def _logical_mount_path(root: object, child: object) -> PurePosixPath:
    if type(root) is not PurePosixPath:
        raise TypeError("root must be an exact PurePosixPath")
    if not root.is_absolute():
        raise ValueError("root must be an absolute logical path")
    if ".." in root.parts or "\x00" in str(root):
        raise ValueError("root must not contain parent traversal or NUL")
    if len(str(root).encode("utf-8")) > _MAX_ROOT_BYTES:
        raise ValueError("root exceeds the supported byte length")
    if type(child) is not str:
        raise TypeError("child must be an exact string")
    if _CHILD_NAME.fullmatch(child) is None:
        raise ValueError("child must be one conservative path component")
    return root / child


def _combined_failures(
    body_error: BaseException,
    cleanup_error: BaseException,
) -> BaseException:
    message = "body and external mount cleanup both failed"
    if isinstance(body_error, Exception) and isinstance(cleanup_error, Exception):
        return ExceptionGroup(message, [body_error, cleanup_error])
    return BaseExceptionGroup(message, [body_error, cleanup_error])


class ExternalMountLeaseOwner:
    def __init__(
        self,
        root: PurePosixPath,
        child: str,
        *,
        acquire: AcquireMount,
        release: ReleaseMount,
    ) -> None:
        if not callable(acquire) or not callable(release):
            raise TypeError("acquire and release must be callable")
        self._path = _logical_mount_path(root, child)
        self._acquire = acquire
        self._release = release
        self._state = MountLeaseState.NEW
        self._lease: ExternalMountLease | None = None

    @property
    def path(self) -> PurePosixPath:
        return self._path

    @property
    def state(self) -> MountLeaseState:
        return self._state

    @property
    def lease(self) -> ExternalMountLease:
        if self._lease is None:
            raise MountLeaseStateError("no owned mount lease is available")
        return self._lease

    def __enter__(self) -> ExternalMountLease:
        if self._state is not MountLeaseState.NEW:
            raise MountLeaseStateError(
                f"cannot acquire from {self._state.name}"
            )
        handle = self._acquire(self._path)
        lease = ExternalMountLease(self._path, handle)
        self._lease = lease
        self._state = MountLeaseState.ACTIVE
        return lease

    def _release_owned_lease(self) -> None:
        if self._state not in {
            MountLeaseState.ACTIVE,
            MountLeaseState.CLEANUP_FAILED,
        }:
            raise MountLeaseStateError(
                f"cannot release from {self._state.name}"
            )
        lease = self.lease
        try:
            self._release(lease)
        except BaseException:
            self._state = MountLeaseState.CLEANUP_FAILED
            raise
        else:
            self._state = MountLeaseState.RELEASED
            self._lease = None

    def retry_cleanup(self) -> None:
        if self._state is not MountLeaseState.CLEANUP_FAILED:
            raise MountLeaseStateError("cleanup retry requires CLEANUP_FAILED")
        self._release_owned_lease()

    def __exit__(
        self,
        _exc_type: type[BaseException] | None,
        body_error: BaseException | None,
        _traceback: TracebackType | None,
    ) -> bool:
        if self._state is not MountLeaseState.ACTIVE:
            raise MountLeaseStateError(
                f"cannot exit from {self._state.name}"
            )
        try:
            self._release_owned_lease()
        except BaseException as cleanup_error:
            if body_error is None:
                raise
            raise _combined_failures(body_error, cleanup_error) from None
        return False
```

## Example

```python
from dataclasses import FrozenInstanceError


class BodyProbeError(Exception):
    pass


class CleanupProbeError(Exception):
    pass


class FakeMountAdapter:
    def __init__(self, *, fail_release_once: bool = False) -> None:
        self.claims: dict[PurePosixPath, str] = {}
        self.acquire_calls: list[PurePosixPath] = []
        self.release_calls: list[ExternalMountLease] = []
        self.fail_release_once = fail_release_once

    def acquire(self, path: PurePosixPath) -> object:
        self.acquire_calls.append(path)
        if path in self.claims:
            raise MountPathClaimedError(f"path is already claimed: {path}")
        handle = f"lease-{len(self.acquire_calls)}"
        self.claims[path] = handle
        return handle

    def release(self, lease: ExternalMountLease) -> None:
        self.release_calls.append(lease)
        if self.claims.get(lease.path) != lease.handle:
            raise AssertionError("release did not receive the owned lease")
        if self.fail_release_once:
            self.fail_release_once = False
            raise CleanupProbeError("cleanup was interrupted")
        del self.claims[lease.path]


root = PurePosixPath("/logical/workspaces")
success_adapter = FakeMountAdapter()
success_owner = ExternalMountLeaseOwner(
    root,
    "job-one",
    acquire=success_adapter.acquire,
    release=success_adapter.release,
)
with success_owner as successful_lease:
    active_state = success_owner.state

try:
    successful_lease.path = PurePosixPath("/changed")
except FrozenInstanceError:
    frozen = True
else:
    frozen = False

retry_adapter = FakeMountAdapter(fail_release_once=True)
retry_owner = ExternalMountLeaseOwner(
    root,
    "job-two",
    acquire=retry_adapter.acquire,
    release=retry_adapter.release,
)
try:
    with retry_owner as unresolved_lease:
        raise BodyProbeError("operation failed")
except ExceptionGroup as failures:
    combined_types = tuple(type(error) for error in failures.exceptions)
else:
    combined_types = ()

failed_state = retry_owner.state
same_unresolved_lease = retry_owner.lease is unresolved_lease
retry_owner.retry_cleanup()

try:
    retry_owner.__enter__()
except MountLeaseStateError:
    reentry_rejected = True
else:
    reentry_rejected = False

claimed_adapter = FakeMountAdapter()
claimed_path = root / "already-owned"
claimed_adapter.claims[claimed_path] = "external-lease"
claimed_owner = ExternalMountLeaseOwner(
    root,
    "already-owned",
    acquire=claimed_adapter.acquire,
    release=claimed_adapter.release,
)
try:
    with claimed_owner:
        pass
except MountPathClaimedError:
    existing_claim_rejected = True
else:
    existing_claim_rejected = False

assert (
    successful_lease,
    active_state,
    success_owner.state,
    success_adapter.claims,
    frozen,
    combined_types,
    failed_state,
    same_unresolved_lease,
    retry_owner.state,
    len(retry_adapter.acquire_calls),
    len(retry_adapter.release_calls),
    reentry_rejected,
    existing_claim_rejected,
    claimed_owner.state,
) == (
    ExternalMountLease(root / "job-one", "lease-1"),
    MountLeaseState.ACTIVE,
    MountLeaseState.RELEASED,
    {},
    True,
    (BodyProbeError, CleanupProbeError),
    MountLeaseState.CLEANUP_FAILED,
    True,
    MountLeaseState.RELEASED,
    1,
    2,
    True,
    True,
    MountLeaseState.NEW,
)
```

## Trade-offs and Limitations

The paths are logical values only. Validation does not inspect, create,
remove, authorize, or resolve anything on a real filesystem. The callbacks
are trusted adapters: acquisition must either return one handle or fail
without leaving a claim, and it must reject a pre-existing claim atomically.
Release must be safe to retry with the same frozen lease because an exception
can leave external ownership ambiguous.

The owner is single-use after successful acquisition and is not thread-safe.
After a cleanup failure it retains the lease in `CLEANUP_FAILED`; only
`retry_cleanup()` can advance it to `RELEASED`. A process crash can still lose
that unresolved ownership. Combining body and cleanup failures preserves both
causes, but callers may need `except*` handling for the resulting exception
group.

This pattern deliberately contains no credentials, subprocess calls, network
requests, mount detection, timeout enforcement, checkout behavior, real
filesystem mutation, or recursive deletion. Those belong in the injected
adapter and its separately tested operational boundary.

## Related Snippets

<!-- catalog:related:start -->
- [Elect One Final Releaser from Bounded Named Leases](elect-one-final-releaser-from-bounded-named-leases.md)
- [Run One Async Operation with a Bounded Resource Stack](run-one-async-operation-with-a-bounded-resource-stack.md)
<!-- catalog:related:end -->
