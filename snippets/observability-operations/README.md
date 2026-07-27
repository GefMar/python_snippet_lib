# Observability and Operations

Telemetry, logging, metrics, monitoring or operational automation is the
primary behavior.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Capture a Bounded Pickle-Friendly Exception Report](capture-a-bounded-pickle-friendly-exception-report.md) | recipe | interoperability, observability, serialization | Flatten one exception and its native cause or context chain into bounded immutable diagnostic data instead of trying to serialize live exception state. |
| [Classify a Finite Value with Warning and Critical Limits](classify-a-finite-value-with-warning-and-critical-limits.md) | recipe | observability, validation | Classify one finite float as ok, warning, or critical using an explicit direction and strictly ordered inclusive limits. |
| [Classify Required Health Stamps by Freshness](classify-required-health-stamps-by-freshness.md) | recipe | observability, validation | Classify every required source as fresh, stale, or missing from its latest successful timestamp instead of running expensive checks in the read path. |
| [Compute a Process CPU Rate from Two Linux procfs Samples](compute-a-process-cpu-rate-from-two-linux-procfs-samples.md) | algorithm | data-transformation, observability, validation | Compute a Linux process CPU rate from two explicit procfs stat samples without reading a file, consulting a clock, or maintaining a cache. |
| [Count Values in Fixed Upper-Bound Bins](count-values-in-fixed-upper-bound-bins.md) | algorithm | data-transformation, observability | Count finite observations in stable right-closed bins defined by a strictly increasing sequence of upper bounds. |
| [Format Log Records as JSON with Explicit Extra Fields](format-log-records-as-json-with-explicit-extra-fields.md) | recipe | observability, serialization | Serialize each log record to one strict JSON object while keeping caller-supplied extra fields separate from standard logging metadata. |
| [Group Metric Samples by Their Exact Label-Key Shape](group-metric-samples-by-their-exact-label-key-shape.md) | algorithm | data-transformation, observability, validation | Partition immutable metric samples into table plans whose columns are defined by exactly the same label names. |
| [Measure and Freeze Elapsed Time in a Context](measure-and-freeze-elapsed-time-in-a-context.md) | idiom | observability, performance-optimization | Measure intermediate and final elapsed durations from one monotonic clock while freezing the final value when a context exits. |
| [Measure Cache Hit Ratios from Monotonic Counter Snapshots](measure-cache-hit-ratios-from-monotonic-counter-snapshots.md) | algorithm | observability, validation | Derive one cache hit ratio from two immutable cumulative-counter snapshots without coupling the calculation to a cache or metrics system. |
| [Process Log Records in a Background Thread with QueueListener](process-log-records-in-a-background-thread-with-queuelistener.md) | recipe | concurrency-control, lifecycle-management, observability | Move target-handler work from producer threads to one standard-library QueueListener thread with an explicit scoped lifecycle. |
| [Read a Bounded Log Delta Across One Rename-Based Rotation](read-a-bounded-log-delta-across-one-rename-based-rotation.md) | recipe | observability, resource-management, retry-recovery | Resume a binary log from a device, inode, and offset checkpoint while recovering unread bytes across at most one rename-based rotation. |
| [Report Partition Offsets Behind a Fixed Checkpoint](report-partition-offsets-behind-a-fixed-checkpoint.md) | algorithm | observability, validation | Compare current partition offsets with one fixed target checkpoint and report every positive gap in deterministic order. |
| [Resolve the Latest Status with an Explicit Mapping](resolve-the-latest-status-with-an-explicit-mapping.md) | pattern | interoperability, observability, validation | Resolve a current normalized status from a finite unordered event log without leaking external status names into the caller's state model. |
| [Scope Structured Log Fields with Context Variables](scope-structured-log-fields-with-context-variables.md) | pattern | concurrency-control, observability | Attach nested structured fields to log events without passing the same identifiers through every function call. |
<!-- catalog:category:end -->
