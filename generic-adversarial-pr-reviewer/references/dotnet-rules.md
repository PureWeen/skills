# .NET Review Rules

C# and managed/native interop guidance. Load when `.cs` files are changed or
when the diff contains interop markers (`DllImport`, `LibraryImport`,
`StructLayout`, `MarshalAs`, `extern`).

---

## Nullable Reference Types

| Check | What to look for |
|-------|-----------------|
| **`#nullable enable`** | New files should have `#nullable enable` at the top — unless nullable is already enabled at the project level via MSBuild properties. |
| **Scrutinize `!` (null-forgiving operator)** | The postfix `!` null-forgiving operator (e.g., `foo!.Bar`) should be justified. If the value can be null, add a proper null check. If it can't be null, make the type non-nullable. AI-generated code frequently sprinkles `!` to silence warnings — this turns compile-time safety into runtime `NullReferenceException`s. Note: this is about the postfix `!`, not the logical negation `!` (e.g., `if (!someBool)`). |
| **`ArgumentNullException.ThrowIfNull`** | .NET 6+ code should use `ArgumentNullException.ThrowIfNull(param)` for parameter validation. |

---

## Async, Cancellation & Thread Safety

| Check | What to look for |
|-------|-----------------|
| **CancellationToken propagation** | Every `async` method that accepts a `CancellationToken` must pass it to ALL downstream async calls. A token that's accepted but never used is a broken contract. |
| **OperationCanceledException** | Catch-all blocks (`catch (Exception)`) must NOT swallow `OperationCanceledException`. Catch it explicitly first and rethrow, or use a type filter. |
| **Honor the token** | If a method accepts `CancellationToken`, it must observe it — register callbacks, check `IsCancellationRequested` in loops, pass it downstream. Don't accept it just for API completeness. |
| **Thread safety of shared state** | If a new field or property can be accessed from multiple threads (e.g., static caches, event handlers), verify thread-safe access: `ConcurrentDictionary`, `Interlocked`, or explicit locks. A `Dictionary<K,V>` read concurrently with a write is undefined behavior. |
| **Avoid double-checked locking — use `Lazy<T>`** | Double-checked locking is error-prone. Prefer `Lazy<T>` or `LazyInitializer.EnsureInitialized()` — they handle thread-safe initialization correctly. |
| **Singleton initialization completeness** | When a singleton is initialized behind a lock, ensure ALL setup steps complete before publishing the instance. Another thread can see `instance != null` and use it before setup runs. |

---

## Error Handling

| Check | What to look for |
|-------|-----------------|
| **No empty catch blocks** | Every `catch` must capture the `Exception` and log it (or rethrow). No silent swallowing. |
| **Validate parameters** | Enum parameters and string-typed "mode" values must be validated — throw `ArgumentException` or `NotSupportedException` for unexpected values. |
| **Fail fast on critical ops** | If a critical operation fails, throw immediately. Silently continuing leads to confusing downstream failures. |
| **Check process exit codes** | If one operation checks the process exit code, ALL similar operations must too. Inconsistent error checking creates a false sense of safety. |
| **Log messages must have context** | A bare `"GetModuleHandle failed"` could be anything. Include what you were doing and what value was unexpected. |
| **Differentiate similar error messages** | Two messages saying `"X failed"` for different operations are impossible to debug. Make each unique. |
| **Include actionable details in exceptions** | Use `nameof` for parameter names. Include the unsupported value or unexpected type. Never throw empty exceptions. |
| **Challenge exception swallowing** | When a PR adds `catch { continue; }` or `catch { return null; }`, question whether the exception is truly expected or masking a deeper problem. |

---

## Performance

| Check | What to look for |
|-------|-----------------|
| **Avoid unnecessary allocations** | Don't create intermediate collections when LINQ chaining or a single list would do. Char arrays for `string.Split()` should be `static readonly` fields. |
| **ArrayPool for large buffers** | Buffers ≥ 1 KB should use `ArrayPool<byte>.Shared.Rent()` with `try`/`finally` return. |
| **`HashSet.Add()` already handles duplicates** | Calling `.Contains()` before `.Add()` does the hash lookup twice. Just call `.Add()`. |
| **Don't wrap a value in an interpolated string** | `$"{someString}"` creates an unnecessary `string.Format` call when `someString` is already a string. |
| **Pre-allocate collections when size is known** | Use `new List<T>(capacity)` or `new Dictionary<TK, TV>(count)` when the size is known or estimable. |
| **Avoid closures in hot paths** | Lambdas that capture local variables allocate a closure object on every call. In loops or frequently-called methods, extract the lambda to a static method or cache the delegate. |
| **Place cheap checks before expensive ones** | In validation chains, test simple conditions (null checks, boolean flags) before allocating strings or doing I/O. |
| **Watch for O(n²)** | Nested loops over the same collection, repeated `.Contains()` on a `List<T>`, or LINQ `.Where()` inside a loop. Switch to `HashSet<T>` or `Dictionary<TK, TV>` for lookups. |

---

## Code Organization

| Check | What to look for |
|-------|-----------------|
| **One type per file** | Each public class, struct, enum, or interface should be in its own file named after the type. |
| **Use `record` for data types** | Immutable data-carrier types should be `record` types — they get value equality, `ToString()`, and deconstruction for free. |
| **Remove unused code** | Dead methods, speculative helpers, and code "for later" should be removed. No commented-out code — Git has history. |
| **New helpers default to `internal`** | New utility methods should be `internal` unless a confirmed external consumer needs them. |
| **Reduce indentation with early returns** | Invert logic for the common case with `continue`/`return` so complex cases have less nesting. |
| **Well-named constants over magic numbers** | `if (retryCount > 3)` should be `if (retryCount > MaxRetries)`. Constants document intent. |

---

## Managed ↔ Native Interop

Load this section when the diff contains `DllImport`, `LibraryImport`,
`StructLayout`, `MarshalAs`, `UnmanagedCallersOnly`, or `extern "C"` markers.

| Check | What to look for |
|-------|-----------------|
| **`static_cast` over C-style casts** | `static_cast<int>(val)` is checked at compile time. `(int)val` can silently reinterpret bits. Always use C++ casts in interop boundaries. |
| **`nullptr` over `NULL`** | `NULL` is `0` in C++, which can silently convert to integral types. `nullptr` has proper pointer semantics. |
| **Struct field ordering for padding** | When defining structs shared between managed and native code, order fields largest-to-smallest to minimize padding. Keep `[StructLayout(LayoutKind.Sequential)]` and matching C struct in sync. |
| **Bool marshalling** | C++ `bool` is 1 byte, Windows `BOOL` is 4 bytes. When P/Invoking, explicitly specify `[MarshalAs(UnmanagedType.U1)]` or `[MarshalAs(UnmanagedType.Bool)]`. |
| **String marshalling charset** | P/Invoke string parameters should specify `CharSet.Unicode` (UTF-16) or use `[MarshalAs(UnmanagedType.LPUTF8Str)]` for UTF-8. Don't rely on the default (ANSI on Windows). |
