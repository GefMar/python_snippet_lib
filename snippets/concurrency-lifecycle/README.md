# Concurrency and Lifecycle

Scheduling, synchronization, cancellation, task ownership or
thread/process/resource lifecycle is the primary invariant.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Collect Thread-Pool Results and Errors as Futures Complete](collect-thread-pool-results-and-errors-as-futures-complete.md) | recipe | concurrency-control, resource-management | Run independent blocking calls in a thread pool while retaining both successful values and item-specific failures as each future finishes. |
| [Drain Bounded Deferred Writes Outside the Queue Lock](drain-bounded-deferred-writes-outside-the-queue-lock.md) | pattern | concurrency-control, networking, resource-management | Queue response bytes under a short lock, then perform potentially blocking writes without holding that queue lock. |
| [Gather Async Results with Bounded Concurrency](gather-async-results-with-bounded-concurrency.md) | recipe | concurrency-control, resource-management | Apply an asynchronous worker to a finite iterable while limiting active calls and preserving input order in the returned results. |
| [Guard an Async Resource with Explicit Lifecycle States](guard-an-async-resource-with-explicit-lifecycle-states.md) | pattern | concurrency-control, lifecycle-management, resource-management | Serialize asynchronous start and stop transitions so callers can observe and enforce one explicit resource lifecycle. |
| [Guard Readers with a Writer-Priority Read-Write Lock](guard-readers-with-a-writer-priority-read-write-lock.md) | pattern | concurrency-control, resource-management | Allow several threads to read shared state concurrently while ensuring that later readers cannot continually bypass a writer that is already waiting. |
| [Initialize One Shared Resource Lazily with Serialized Retries](initialize-one-shared-resource-lazily-with-serialized-retries.md) | pattern | concurrency-control, lifecycle-management, retry-recovery | Serialize first access to a shared synchronous resource, retry only approved creation failures, and publish the value only after construction succeeds. |
| [Prevent Overlapping POSIX Jobs with a Nonblocking File Lock](prevent-overlapping-posix-jobs-with-a-nonblocking-file-lock.md) | recipe | automation, concurrency-control | Hold a nonblocking advisory lock for one job run so a concurrent local invocation can fail immediately instead of overlapping it. |
| [Refresh an Async Value Within a Bounded Stale Window](refresh-an-async-value-within-a-bounded-stale-window.md) | pattern | caching, concurrency-control, lifecycle-management, retry-recovery | Keep serving one immutable async-produced value while a single background task refreshes it, but never serve that stale value beyond an explicit monotonic age limit. |
| [Run an Async Worker on Clock-Aligned Ticks Without Catch-Up](run-an-async-worker-on-clock-aligned-ticks-without-catch-up.md) | pattern | automation, concurrency-control, lifecycle-management | Run one asynchronous worker on shared wall-clock boundaries while skipping missed slots instead of replaying a burst after an overrun. |
| [Run Bounded Thread Work by Priority and Submission Order](run-bounded-thread-work-by-priority-and-submission-order.md) | pattern | concurrency-control, lifecycle-management, resource-management | Run blocking callables on fixed worker threads while bounding queued work and ordering pending jobs by priority, then FIFO submission order. |
| [Stop a Polling Worker Cooperatively with an Event](stop-a-polling-worker-cooperatively-with-an-event.md) | pattern | concurrency-control, resource-management | Run a bounded polling operation until another thread requests shutdown through a shared event that also interrupts idle waiting. |
| [Stream Bounded stdout and stderr Lines from a POSIX Process](stream-bounded-stdout-and-stderr-lines-from-a-posix-process.md) | integration | automation, concurrency-control, lifecycle-management, resource-management | Drain stdout and stderr from one shell-free POSIX child without pipe deadlock while bounding runtime, raw bytes, line length, and emitted records. |
| [Submit a Callable with a Snapshot of the Current Context](submit-a-callable-with-a-snapshot-of-the-current-context.md) | recipe | concurrency-control, interoperability, observability | Capture the current context for each thread-pool submission so the worker sees the caller's context variables without custom globals or exception wrappers. |
<!-- catalog:category:end -->
