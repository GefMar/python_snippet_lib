# Networking and Protocols

Wire format, transport, HTTP/RPC/socket behavior or protocol-client semantics
determine correctness.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Collect Matching Cursor Pages with an Explicit Page Budget](collect-matching-cursor-pages-with-an-explicit-page-budget.md) | pattern | networking, resource-management | Collect a requested number of matching items across cursor pages while making every successful stop condition explicit. |
| [Iterate Cursor-Paginated Results Lazily](iterate-cursor-paginated-results-lazily.md) | pattern | networking, resource-management | Hide cursor pagination behind an iterator while keeping page fetches lazy, ordered, and bounded by explicit safety checks. |
| [Parse a Bounded ASCII Media Type Value](parse-a-bounded-ascii-media-type-value.md) | recipe | interoperability, parsing, serialization, validation | Parse one strict media type and its parameters into an immutable value instead of passing an ambiguous raw header string through an application. |
| [Parse a Bounded Host and Port with Bracketed IPv6](parse-a-bounded-host-and-port-with-bracketed-ipv6.md) | recipe | networking, parsing, validation | Parse one conservative host-and-port value without confusing the colons inside an IPv6 literal with a port separator. |
| [Read and Write Size-Capped Varint Frames](read-and-write-size-capped-varint-frames.md) | recipe | networking, serialization, validation | Frame byte payloads on a blocking stream with a canonical unsigned LEB128 length while rejecting oversized declarations before reading their bodies. |
<!-- catalog:category:end -->
