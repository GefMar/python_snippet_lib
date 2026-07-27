---
title: "Elect One Final Releaser from Bounded Named Leases"
snippet_type: pattern
use_cases:
  - concurrency-control
  - lifecycle-management
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - guard-an-async-resource-with-explicit-lifecycle-states.md
  - initialize-one-shared-resource-lazily-with-serialized-retries.md
  - ../networking-protocols/release-a-pooled-response-connection-only-after-clean-eof.md
---

# Elect One Final Releaser from Bounded Named Leases

## Idea and Problem

Elect exactly one caller to perform final cleanup after every named lease has been released once.

A fixed set of names makes ownership explicit. Each successful release returns
an immutable decision with the still-pending names in their original order.
Only the transition that removes the last name requests cleanup, while unknown
or repeated names fail before changing shared state.

## When to Use

Use this pattern when several threads in one process share a resource and all
consumers are known before work starts. Each consumer must stop using the
resource before releasing its name and should place that release in a
`finally` block. The caller receiving the final signal performs cleanup after
the synchronized state transition has completed.

Use durable coordination when consumers run in different processes or can
disappear without executing Python cleanup code. This tracker does not detect
crashes, recover abandoned leases, or prove that a consumer has actually
finished with the resource.

## Implementation

```python
import re
import threading
from dataclasses import dataclass


_MAX_LEASES = 256
_MAX_LEASE_ID_BYTES = 64
_LEASE_ID = re.compile(r"[a-z][a-z0-9._:-]{0,63}", re.ASCII)


class UnknownLeaseError(ValueError):
    pass


class LeaseAlreadyReleasedError(RuntimeError):
    pass


@dataclass(frozen=True, slots=True)
class ReleaseDecision:
    released_id: str
    remaining: tuple[str, ...]
    cleanup_required: bool


def _validated_lease_id(value: object) -> str:
    if type(value) is not str:
        raise TypeError("lease IDs must be exact strings")
    if not 1 <= len(value) <= _MAX_LEASE_ID_BYTES:
        raise ValueError("lease ID length is outside the supported range")
    if _LEASE_ID.fullmatch(value) is None:
        raise ValueError("lease IDs must be canonical printable ASCII")
    return value


class FinalReleaseElection:
    def __init__(self, lease_ids: tuple[str, ...]) -> None:
        if type(lease_ids) is not tuple:
            raise TypeError("lease_ids must be an exact tuple")
        if not 1 <= len(lease_ids) <= _MAX_LEASES:
            raise ValueError("lease count is outside the supported range")

        validated = []
        known: set[str] = set()
        for raw_lease_id in lease_ids:
            lease_id = _validated_lease_id(raw_lease_id)
            if lease_id in known:
                raise ValueError("lease IDs must be unique")
            known.add(lease_id)
            validated.append(lease_id)

        self._ordered_ids = tuple(validated)
        self._known = frozenset(known)
        self._pending = set(known)
        self._lock = threading.Lock()

    def release(self, lease_id: str) -> ReleaseDecision:
        validated_id = _validated_lease_id(lease_id)
        with self._lock:
            if validated_id not in self._known:
                raise UnknownLeaseError(f"unknown lease ID: {validated_id!r}")
            if validated_id not in self._pending:
                raise LeaseAlreadyReleasedError(
                    f"lease was already released: {validated_id!r}"
                )

            self._pending.remove(validated_id)
            remaining = tuple(
                candidate
                for candidate in self._ordered_ids
                if candidate in self._pending
            )
            return ReleaseDecision(
                released_id=validated_id,
                remaining=remaining,
                cleanup_required=not remaining,
            )

    def remaining_lease_ids(self) -> tuple[str, ...]:
        with self._lock:
            return tuple(
                candidate
                for candidate in self._ordered_ids
                if candidate in self._pending
            )
```

## Example

```python
from concurrent.futures import ThreadPoolExecutor
from dataclasses import FrozenInstanceError
from threading import Barrier


lease_ids = ("reader-one", "reader-two", "reader-three")
election = FinalReleaseElection(lease_ids)
start_together = Barrier(len(lease_ids))


def finish_reader(lease_id: str) -> ReleaseDecision:
    try:
        start_together.wait()
    finally:
        decision = election.release(lease_id)
    return decision


with ThreadPoolExecutor(max_workers=len(lease_ids)) as executor:
    decisions = tuple(executor.map(finish_reader, lease_ids))

probe = FinalReleaseElection(("lease-one", "lease-two"))
probe.release("lease-one")
before_failures = probe.remaining_lease_ids()

try:
    probe.release("missing-lease")
except UnknownLeaseError:
    unknown_rejected = True
else:
    unknown_rejected = False

try:
    probe.release("lease-one")
except LeaseAlreadyReleasedError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

after_failures = probe.remaining_lease_ids()
probe_final = probe.release("lease-two")

try:
    decisions[0].remaining = ()
except FrozenInstanceError:
    mutation_rejected = True
else:
    mutation_rejected = False

assert (
    {decision.released_id for decision in decisions} == set(lease_ids)
    and sum(decision.cleanup_required for decision in decisions) == 1
    and next(
        decision.remaining
        for decision in decisions
        if decision.cleanup_required
    )
    == ()
    and all(
        decision.remaining
        == tuple(
            lease_id
            for lease_id in lease_ids
            if lease_id in decision.remaining
        )
        for decision in decisions
    )
    and before_failures == after_failures == ("lease-two",)
    and unknown_rejected
    and duplicate_rejected
    and probe_final.cleanup_required
    and mutation_rejected
)
```

## Trade-offs and Limitations

The lock makes release and inspection atomic among threads using this one
object. Materializing the ordered remainder costs `O(n)` per release, so a full
cycle costs `O(n^2)` time and `O(n)` state; the 256-lease ceiling keeps both
bounded. Lease membership is fixed at construction and cannot grow while work
is in progress.

Exactly one successful transition requests cleanup only when every configured
consumer calls `release()` exactly once. A missed `finally`, process exit, or
crash can leave the resource unreleased. A crash after the final decision can
also lose the cleanup action, and a cleanup failure does not elect a replacement.
The tracker executes no callback, deletion, network operation, or other I/O,
and its lock does not coordinate separate processes or machines.

## Related Snippets

<!-- catalog:related:start -->
- [Guard an Async Resource with Explicit Lifecycle States](guard-an-async-resource-with-explicit-lifecycle-states.md)
- [Initialize One Shared Resource Lazily with Serialized Retries](initialize-one-shared-resource-lazily-with-serialized-retries.md)
- [Release a Pooled Response Connection Only After Clean EOF](../networking-protocols/release-a-pooled-response-connection-only-after-clean-eof.md)
<!-- catalog:related:end -->
