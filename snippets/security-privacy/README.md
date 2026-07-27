# Security and Privacy

A threat model, cryptographic, authentication, authorization, redaction,
privacy or hostile-input invariant is central.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Audit Symlinks in a Bounded Directory Tree](audit-symlinks-in-a-bounded-directory-tree.md) | recipe | resource-management, security, validation | Inspect a stable directory without following links and reject any symbolic link whose resolved target is missing or outside the inspected root. |
| [Authenticate Bounded Payloads with Versioned HMAC Keys](authenticate-bounded-payloads-with-versioned-hmac-keys.md) | pattern | security, serialization, validation | Authenticate a bounded payload with the key assigned to the current time epoch while retaining older versioned keys for verification. |
| [Load an Authenticated Legacy Pickle with Restricted Globals](load-an-authenticated-legacy-pickle-with-restricted-globals.md) | recipe | interoperability, security, serialization, validation | Never unpickle untrusted or attacker-modifiable data; for an unavoidable authenticated legacy channel, verify a bounded envelope before restricting every global lookup to an immutable exact allowlist. |
| [Redact a Printable ASCII Secret with a Bounded Visible Tail](redact-a-printable-ascii-secret-with-a-bounded-visible-tail.md) | recipe | observability, security, validation | Mask every character of a short secret and reveal only a bounded suffix of longer values while keeping at least eight characters hidden. |
| [Redact Explicit Paths in Bounded JSON-Like Data](redact-explicit-paths-in-bounded-json-like-data.md) | recipe | data-transformation, observability, security, validation | Copy a bounded JSON-like value and replace leaves at explicitly tokenized field paths without retaining any part of the original values. |
| [Validate a Conservative Unicode Filename Component](validate-a-conservative-unicode-filename-component.md) | recipe | interoperability, validation | Normalize and validate one user-facing Unicode filename component without treating it as an authorized storage path. |
| [Verify an RFC 7636 S256 PKCE Challenge](verify-an-rfc-7636-s256-pkce-challenge.md) | recipe | interoperability, security, validation | Validate an OAuth PKCE verifier and compare its RFC 7636 S256 transformation with an exact unpadded challenge. |
<!-- catalog:category:end -->
