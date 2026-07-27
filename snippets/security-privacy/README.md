# Security and Privacy

A threat model, cryptographic, authentication, authorization, redaction,
privacy or hostile-input invariant is central.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Audit Symlinks in a Bounded Directory Tree](audit-symlinks-in-a-bounded-directory-tree.md) | recipe | resource-management, security, validation | Inspect a stable directory without following links and reject any symbolic link whose resolved target is missing or outside the inspected root. |
| [Authenticate Bounded Payloads with Versioned HMAC Keys](authenticate-bounded-payloads-with-versioned-hmac-keys.md) | pattern | security, serialization, validation | Authenticate a bounded payload with the key assigned to the current time epoch while retaining older versioned keys for verification. |
| [Create and Verify a Short-Lived HMAC Download URL](create-and-verify-a-short-lived-hmac-download-url.md) | pattern | networking, security | Issue a replayable HTTPS capability URL whose exact endpoint, resource, audience, issuance time, expiry, and key ID are authenticated together. |
| [Encrypt a Bounded Value with a Versioned AES-GCM Key Envelope](encrypt-a-bounded-value-with-a-versioned-aes-gcm-key-envelope.md) | integration | security, serialization, validation | Encrypt one bounded byte value in a canonical envelope that identifies a retained key while authenticating the format version, purpose, and key ID as associated data. |
| [Load an Authenticated Legacy Pickle with Restricted Globals](load-an-authenticated-legacy-pickle-with-restricted-globals.md) | recipe | interoperability, security, serialization, validation | Never unpickle untrusted or attacker-modifiable data; for an unavoidable authenticated legacy channel, verify a bounded envelope before restricting every global lookup to an immutable exact allowlist. |
| [Plan a Bounded Notebook Storage Key with Collision Suggestions](plan-a-bounded-notebook-storage-key-with-collision-suggestions.md) | recipe | persistence, security, validation | Validate one raw relative POSIX notebook path and plan a namespaced storage key against an immutable snapshot of occupied keys. |
| [Redact a Printable ASCII Secret with a Bounded Visible Tail](redact-a-printable-ascii-secret-with-a-bounded-visible-tail.md) | recipe | observability, security, validation | Mask every character of a short secret and reveal only a bounded suffix of longer values while keeping at least eight characters hidden. |
| [Redact Explicit Paths in Bounded JSON-Like Data](redact-explicit-paths-in-bounded-json-like-data.md) | recipe | data-transformation, observability, security, validation | Copy a bounded JSON-like value and replace leaves at explicitly tokenized field paths without retaining any part of the original values. |
| [Separate Executable and Redacted Views of a Command Argument Vector](separate-executable-and-redacted-views-of-a-command-argument-vector.md) | pattern | observability, security, validation | Build one immutable command plan that keeps the shell-free execution vector hidden from representation while exposing a separately rendered view with fixed secret placeholders. |
| [Validate a Conservative Unicode Filename Component](validate-a-conservative-unicode-filename-component.md) | recipe | interoperability, validation | Normalize and validate one user-facing Unicode filename component without treating it as an authorized storage path. |
| [Verify an RFC 7636 S256 PKCE Challenge](verify-an-rfc-7636-s256-pkce-challenge.md) | recipe | interoperability, security, validation | Validate an OAuth PKCE verifier and compare its RFC 7636 S256 transformation with an exact unpadded challenge. |
<!-- catalog:category:end -->
