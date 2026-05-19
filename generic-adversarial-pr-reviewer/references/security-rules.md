# Security Review Rules

Security checklist for code reviews. Applicable to any repository handling file
I/O, archives, process execution, or user input.

---

## Archive & Path Safety

| Check | What to look for |
|-------|-----------------|
| **Zip Slip protection** | Archive extraction must validate that every entry path, after `Path.GetFullPath()` or equivalent normalization, resolves under the destination directory. Never use bulk extraction APIs for untrusted archives without entry-by-entry validation. |
| **Path traversal** | `StartsWith()` checks on paths must normalize with full path resolution first. A path like `C:\Program Files\..\Users\evil` bypasses naive prefix checks. Also check for directory boundary issues (`C:\Program FilesX` matching `C:\Program Files`). |

---

## Process & Command Safety

| Check | What to look for |
|-------|-----------------|
| **Command injection** | Arguments passed to process execution must be sanitized. Use argument list APIs (not string interpolation into command strings). Never interpolate user/external input into command strings. |
| **Environment variable injection** | Don't trust environment variables for security decisions. They can be set by the caller. |

---

## Input Validation

| Check | What to look for |
|-------|-----------------|
| **Untrusted input flows** | Trace user/external input through the code. If it reaches file operations, database queries, process execution, or network calls without validation, flag it. |
| **Deserialization** | Deserializing untrusted data (JSON, XML, binary) can lead to code execution or DoS. Verify that deserialization uses safe settings and validates types. |

---

## Secrets & Credentials

| Check | What to look for |
|-------|-----------------|
| **Secrets in source** | API keys, tokens, passwords, connection strings, or private keys in committed code. Check string literals, config files, and test fixtures. |
| **Secrets in logs** | Logging statements that might output tokens, passwords, or PII. Redact sensitive values before logging. |
