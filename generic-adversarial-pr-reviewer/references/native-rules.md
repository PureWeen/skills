# Native Code (C/C++) Review Rules

C and C++ guidance for any repository with native code. Load when `.c`, `.cpp`,
`.h`, or `.hpp` files change.

---

## Memory Management

| Check | What to look for |
|-------|-----------------|
| **Every `new` needs a `delete` or justification** | If a `new` has no matching cleanup, document *why* the leak is acceptable and its worst-case size. "Small leak" is not a justification without quantifying "how small" and "how often." |
| **Quantify leaks** | Is the leaked path hit once at startup (acceptable) or once per request/invocation (not acceptable)? The answer determines whether a leak matters. |
| **Document known leaks in commit messages** | If a small leak is deliberately accepted, say so in the commit message so reviewers don't rediscover it later. |
| **Watch for leaks in external APIs** | Some external functions allocate memory that the caller must free. Check the docs for every external API call that returns a pointer. |
| **Use RAII (`std::unique_ptr`, etc.)** | If an object has a clear owner, use smart pointers or RAII to ensure cleanup. Don't rely on manual `delete`. |

---

## C++ Best Practices

| Check | What to look for |
|-------|-----------------|
| **Virtual destructor on base classes** | Any base class with virtual methods must have a public virtual destructor. Without one, `delete`-through-base-pointer is undefined behavior. |
| **Delete copy/move constructors when inappropriate** | Types holding non-copyable resources (file handles, OS handles, native refs) must use `= delete` on copy constructor and assignment operator. |
| **Prefer `private` over `protected`** | Unless the type is explicitly designed for subclassing, use `private`. Don't speculatively make things `protected`. |
| **Use `const` where possible** | If a parameter or function argument isn't modified, declare it `const`. |
| **Handle `EINTR` for system calls** | `read()`, `write()`, and other syscalls can return `EINTR` when interrupted by a signal. Retry in a loop. |
| **Use `sizeof()` not magic numbers** | `16` should be `sizeof(some_type)` or equivalent. Magic numbers make code fragile and unreadable. |
| **No commented-out code** | If it's not needed, delete it. Git has history. |
| **Don't use compiler-reserved identifiers** | Double-underscore `__` prefixed names are reserved by the C/C++ standard. |

---

## Symbol Visibility

| Check | What to look for |
|-------|-----------------|
| **Use `-fvisibility=hidden` by default** | Only export symbols that are explicitly needed. If a native function isn't called from managed code or another library, it shouldn't be exported. |
| **Question every exported symbol** | Search for actual usage before keeping an exported function. If nothing outside the module calls it, make it internal. |
| **Document cross-references for exports** | Add comments with links to callers. When the caller changes, it's clear the export can be removed. |
| **Remove dead symbols proactively** | When an upstream consumer no longer uses a function, remove it now. Don't wait. |

---

## Platform-Specific Code

| Check | What to look for |
|-------|-----------------|
| **Prefer wide (W) Win32 functions** | Use `GetModuleHandleExW` not `GetModuleHandleEx` (the macro). Avoid the ANSI (`A`) variants entirely. |
| **Don't change platform-guarded code unnecessarily** | If a change is in a `#if defined(WINDOWS)` block, verify it's actually needed on that platform. |
| **Check return codes on all platform APIs** | Even APIs that "shouldn't fail" have return values. Check them. |
