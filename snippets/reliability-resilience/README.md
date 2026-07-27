# Reliability and Resilience

Failure policy such as retry, deadline, idempotency, fallback, recovery, cache
correctness or lease behavior is the primary invariant.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Adjust a Bounded Batch Size from Processing Time](adjust-a-bounded-batch-size-from-processing-time.md) | algorithm | performance-optimization, resource-management | Apply one deterministic batch-size adjustment when a completed full batch falls outside an explicit processing-time deadband. |
| [Assign Stable Schedule Slots with a Digest](assign-stable-schedule-slots-with-a-digest.md) | algorithm | performance-optimization, resource-management | Assign a string key reproducibly to one of a fixed number of schedule slots using a domain-separated digest instead of Python's process-randomized hash. |
| [Cache Values with a Monotonic TTL and Early Jitter](cache-values-with-a-monotonic-ttl-and-early-jitter.md) | recipe | caching, resource-management | Bound an in-process cache and spread its expirations before a monotonic freshness ceiling without making cached None values look like misses. |
| [Commit a Source Checkpoint Only After the Sink Accepts a Batch](commit-a-source-checkpoint-only-after-the-sink-accepts-a-batch.md) | pattern | data-transformation, retry-recovery | Advance a source checkpoint only after the sink has durably accepted every prepared event from that exact bounded batch. |
| [Compensate Completed Workflow Steps in Reverse Order](compensate-completed-workflow-steps-in-reverse-order.md) | pattern | automation, resource-management, retry-recovery | Register one compensation after each successful synchronous step so a later failure can unwind completed work without hiding the primary error. |
| [Flush Fixed-Size Batches Through a Scoped Sink](flush-fixed-size-batches-through-a-scoped-sink.md) | pattern | data-transformation, resource-management, retry-recovery | Collect pushed items into fixed-size immutable batches and flush the final partial batch only when the surrounding operation exits successfully. |
| [Hold a Switch Active Through a Monotonic Cooldown](hold-a-switch-active-through-a-monotonic-cooldown.md) | pattern | observability, resource-management | Keep a threshold-activated switch on until a full quiet cooldown has elapsed, without embedding clocks or side effects in the state transition. |
| [Model Independent Blocking Reasons as an Immutable Set](model-independent-blocking-reasons-as-an-immutable-set.md) | pattern | configuration, validation | Represent independent reasons that prevent an operation as immutable identifiers and derive the combined blocked state from whether that set is empty. |
| [Poll a Remote Operation Within Deadline and Failure Budgets](poll-a-remote-operation-within-deadline-and-failure-budgets.md) | integration | automation, networking, resource-management, retry-recovery | Observe a remote operation until it succeeds or fails while bounding elapsed time, status reads, and consecutive transient read failures independently. |
| [Propagate a Monotonic Deadline with ContextVar](propagate-a-monotonic-deadline-with-contextvar.md) | pattern | concurrency-control, lifecycle-management, retry-recovery | Carry one process-local monotonic deadline through nested calls so every outgoing operation consumes the same remaining time budget. |
| [Resolve Incoming Configuration with Last-Known-Good Values](resolve-incoming-configuration-with-last-known-good-values.md) | pattern | configuration, validation | Resolve each incoming configuration entry independently while making every same-key fallback and rejection visible to the caller. |
| [Retry Only Eligible Items in a Bounded Batch](retry-only-eligible-items-in-a-bounded-batch.md) | pattern | networking, retry-recovery | Retry only idempotent items with explicitly retryable responses while retaining every item that has already reached a final outcome. |
| [Schedule the Next Review from Outcome and Bounded Coverage](schedule-the-next-review-from-outcome-and-bounded-coverage.md) | algorithm | automation, retry-recovery, validation | Choose a bounded whole-day review interval from an explicit outcome and the fraction of a planned scan that actually completed. |
| [Wait for a Predicate Until a Monotonic Deadline](wait-for-a-predicate-until-a-monotonic-deadline.md) | recipe | retry-recovery | Poll synchronous state within one monotonic time budget instead of scattering unbounded sleep loops through calling code. |
<!-- catalog:category:end -->
