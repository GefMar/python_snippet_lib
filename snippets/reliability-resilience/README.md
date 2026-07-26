# Reliability and Resilience

Failure policy such as retry, deadline, idempotency, fallback, recovery, cache
correctness or lease behavior is the primary invariant.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Assign Stable Schedule Slots with a Digest](assign-stable-schedule-slots-with-a-digest.md) | algorithm | performance-optimization, resource-management | Assign a string key reproducibly to one of a fixed number of schedule slots using a domain-separated digest instead of Python's process-randomized hash. |
| [Hold a Switch Active Through a Monotonic Cooldown](hold-a-switch-active-through-a-monotonic-cooldown.md) | pattern | observability, resource-management | Keep a threshold-activated switch on until a full quiet cooldown has elapsed, without embedding clocks or side effects in the state transition. |
| [Wait for a Predicate Until a Monotonic Deadline](wait-for-a-predicate-until-a-monotonic-deadline.md) | recipe | retry-recovery | Poll synchronous state within one monotonic time budget instead of scattering unbounded sleep loops through calling code. |
<!-- catalog:category:end -->
