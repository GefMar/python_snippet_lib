---
title: "Attach Best-Effort Cleanup with weakref.finalize Without Retaining the Owner"
snippet_type: pattern
use_cases:
  - lifecycle-management
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - cache-one-zero-argument-method-result-per-weakly-referenced-instance.md
  - ../concurrency-lifecycle/own-one-external-mount-lease-with-exception-safe-cleanup.md
  - ../concurrency-lifecycle/run-one-async-operation-with-a-bounded-resource-stack.md
---

# Attach Best-Effort Cleanup with weakref.finalize Without Retaining the Owner

## Idea and Problem

Attach one at-most-once best-effort cleanup callback without letting the finalizer's callback or arguments retain its weakly referenced owner.

The owner stores only a `weakref.finalize` object. That finalizer stores one
external synchronous callback and one bounded immutable resource identifier;
neither is a bound owner method or contains the owner. Explicit `close()` calls
the same finalizer used by collection fallback, so both paths share one
at-most-once cleanup attempt.

## When to Use

Use this pattern as a fallback for a small noncritical in-process resource that
already has a preferred explicit cleanup path. The cleanup callback must be
short, synchronous, safe to call without the owner, and able to identify the
external resource from the immutable identifier alone.

This version deliberately sets `finalizer.atexit` to `False`. It does not run
cleanup during interpreter shutdown, when module globals, event loops, threads,
or external adapters may already be unavailable. Use a durable external
reconciler when abandoned resources must be recovered after process exit.

## Implementation

```python
import gc
import re
import weakref
from collections.abc import Callable

_RESOURCE_ID = re.compile(r"[a-z][a-z0-9._-]{0,63}", re.ASCII)

Cleanup = Callable[[str], None]


class BestEffortCleanupOwner:
    __slots__ = ("__weakref__", "_finalizer")

    def __init__(self, resource_id: str, cleanup: Cleanup) -> None:
        if type(resource_id) is not str:
            raise TypeError("resource_id must be an exact string")
        if _RESOURCE_ID.fullmatch(resource_id) is None:
            raise ValueError("resource_id has invalid bounded ASCII syntax")
        if not callable(cleanup):
            raise TypeError("cleanup must be callable")
        if getattr(cleanup, "__self__", None) is self:
            raise ValueError("cleanup must not be a bound owner method")

        finalizer = weakref.finalize(self, cleanup, resource_id)
        finalizer.atexit = False
        self._finalizer = finalizer

    @property
    def cleanup_pending(self) -> bool:
        return self._finalizer.alive

    def close(self) -> None:
        self._finalizer()
```

## Example

```python
cleanup_events: list[str] = []


def release_external_resource(resource_id: str) -> None:
    cleanup_events.append(resource_id)


explicit = BestEffortCleanupOwner(
    "resource-explicit",
    release_external_resource,
)
explicit_reference = weakref.ref(explicit)
pending_before_close = explicit.cleanup_pending
explicit.close()
pending_after_close = explicit.cleanup_pending
explicit.close()
del explicit
gc.collect()

fallback = BestEffortCleanupOwner(
    "resource-fallback",
    release_external_resource,
)
fallback_reference = weakref.ref(fallback)
fallback_pending = fallback.cleanup_pending
del fallback
gc.collect()

assert (
    cleanup_events,
    pending_before_close,
    pending_after_close,
    fallback_pending,
    explicit_reference(),
    fallback_reference(),
) == (
    ["resource-explicit", "resource-fallback"],
    True,
    False,
    True,
    None,
    None,
)
```

## Trade-offs and Limitations

Collection timing is nondeterministic, and a callback or any callback argument
that retains the owner prevents the fallback from becoming eligible. The
implementation rejects the obvious bound-owner-method case, but it cannot
inspect closures, callable objects, or indirect object graphs for hidden owner
references. The immutable identifier must not contain a secret that should not
remain reachable until cleanup.

`weakref.finalize` guarantees an at-most-once callback attempt, not successful
release. An exception from explicit `close()` propagates after the finalizer is
already dead; an exception from collection-triggered cleanup cannot be returned
to application code. There is no retry state, thread or task affinity, async
awaiting, timeout, cancellation handling, or interpreter-exit cleanup. Use an
explicit context manager, `try`/`finally`, or an async resource stack for
critical resources and operations whose cleanup outcome must be observed.

## Related Snippets

<!-- catalog:related:start -->
- [Cache One Zero-Argument Method Result per Weakly Referenced Instance](cache-one-zero-argument-method-result-per-weakly-referenced-instance.md)
- [Own One External Mount Lease with Exception-Safe Cleanup](../concurrency-lifecycle/own-one-external-mount-lease-with-exception-safe-cleanup.md)
- [Run One Async Operation with a Bounded Resource Stack](../concurrency-lifecycle/run-one-async-operation-with-a-bounded-resource-stack.md)
<!-- catalog:related:end -->
