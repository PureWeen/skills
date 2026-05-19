# MSBuild Review Rules

MSBuild target and task guidance applicable to any .NET repository with MSBuild
build infrastructure. Load when `.targets`, `.props`, `.projitems`, or `.csproj`
files change.

---

## MSBuild Tasks (C#)

| Check | What to look for |
|-------|-----------------|
| **Return `!Log.HasLoggedErrors`** | `RunTask()` / `Execute()` should return `!Log.HasLoggedErrors`, not hardcoded `true`/`false` — hardcoded values skip the centralized error-tracking mechanism. |
| **Use coded errors and warnings** | Errors and warnings should use coded methods (e.g., `Log.LogError("XY1234", ...)`) — never bare `Log.LogError` without a code. Error messages should come from resource files for localizability. |
| **`[Required]` properties need defaults** | `[Required]` properties must be non-nullable with a default: `public string Foo { get; set; } = "";` or `public ITaskItem[] Bar { get; set; } = [];`. Non-`[Required]` and `[Output]` properties should be nullable. |
| **Caching with `RegisterTaskObject`** | Use `BuildEngine4.RegisterTaskObject()` instead of `static` variables for sharing data between tasks or across builds. Use `as` for casts to avoid `InvalidCastException`. |
| **Use appropriate log levels** | Use `MessageImportance.Low` for verbose diagnostics, `Normal` for progress, `High` for important status. Don't spam high-importance messages. |

---

## Process Management in Tasks

| Check | What to look for |
|-------|-----------------|
| **Don't redirect stdout/stderr without draining** | Background processes with `RedirectStandardOutput = true` must have async readers draining the output. Otherwise the OS pipe buffer fills and the child process deadlocks. For fire-and-forget processes, set `Redirect* = false`. |
| **Include stdout in error diagnostics** | When a task captures stdout, pass it to error reporting so failure messages include all output, not just stderr. |

---

## MSBuild Targets & XML

| Check | What to look for |
|-------|-----------------|
| **Underscore prefix for private names** | Internal targets, properties, and item groups should be prefixed with `_` (e.g., `_CompileJava`, `$(_JarFile)`). MSBuild has no visibility — the underscore signals "internal, may be renamed." |
| **Incremental builds (`Inputs`/`Outputs`)** | Every target that *writes files* must have `Inputs` and `Outputs` so MSBuild can skip it when nothing changed. Targets that only read, set properties, or populate item groups do NOT need them. |
| **Stamp files for unknown outputs** | When outputs aren't known ahead of time, use a stamp file. Create it with `<Touch Files="..." AlwaysCreate="True" />`. |
| **`FileWrites` for intermediate files** | Intermediate files must be added to `@(FileWrites)` so `IncrementalClean` doesn't delete them. Use an `<ItemGroup>` block inside the target (it evaluates even when the target is skipped). Do NOT use `<Output TaskParameter="TouchedFiles" ItemName="FileWrites" />` — it won't run when the target is skipped. |
| **Don't duplicate item group transforms** | If a target uses the same transform more than once, compute it into a local item group first and reuse it. Duplicated transforms allocate the same array twice. |
| **Use `->Count()` for empty checks** | Prefer `'@(Items->Count())' != '0'` over `'@(Items)' != ''`. The latter does a string join of all items, producing enormous log messages. |
| **Avoid `BeforeTargets`/`AfterTargets`** | Prefer `$(XDependsOn)` properties to order targets. `AfterTargets` runs even if the predecessor *failed*, causing confusing cascading errors. Use `BeforeTargets`/`AfterTargets` only when no `DependsOn` property exists. |
