# gh-aw Migrations & Version History

Reference for migrating deprecated patterns and version-specific bug history. Only consult this file when editing an older workflow or debugging version-specific weirdness.

## Deprecated Patterns → Current Replacements

| If you see this (old) | Replace with (new) | Migration command |
|---|---|---|
| `network.firewall:` in frontmatter | Remove entirely | `gh aw fix --write` |
| `add-comment.discussion: true` (singular) | `discussions: true` | `gh aw fix --write` |
| `features.inline-agents: true` | Remove (now default-on) | `gh aw fix --write` |
| `cli-proxy` feature flag | `tools.github.mode: gh-proxy` | Manual edit |
| `.mcp.json` at repo root | `.github/mcp.json` | Move file manually |
| `disable-xpia-prompt` in frontmatter | Remove (rejected in strict mode) | Manual edit |
| `--safe-update` CLI flag | `--approve` | Update scripts/docs |
| `features.copilot-requests: true` | `permissions.copilot-requests: write` | `gh aw fix --write` |
| `tools.serena` | Remove; configure an MCP server explicitly if still needed | Manual edit |
| `dangerously-disable-sandbox-agent: true` (Boolean) | String justification ≥ 20 chars (e.g., `"controlled environment with no internet access"`) explaining why the trust boundary is removed — expressions and short strings are rejected by the compiler | Manual edit |
| `user-invokable: true` | Remove entirely — field no longer exists in gh-aw schema and will produce a validation error | Manual edit |
| `disable-model-invocation: true` | Remove entirely — field no longer exists in gh-aw schema and will produce a validation error | Manual edit |

## Version-Specific Bug History

These are bugs that were fixed. If you encounter them, upgrade to the version indicated.

### Fixed in v0.79.6
- **Digest pinning restored** — Container image digest pinning for AWF firewall sidecar images has been restored after a brief regression. The release pipeline now gates on resolved SHA pins before pushing tags, ensuring supply chain integrity for AWF firewall images. No workflow change needed; the compiler picks this up automatically on recompile.
- **AWF firewall security update** — The AWF firewall runtime was updated to incorporate upstream security and stability fixes. No workflow change needed.
- **Windows CLI deadlock** — A process wrapper deadlock in the Windows `gh aw` CLI integration has been resolved, unblocking local development workflows that stalled when spawning child processes on Windows.

### Fixed in v0.79.4
- **`dangerously-disable-sandbox-agent` requires string justification** — Boolean `true` is no longer accepted. Workflows must supply a static literal string of at least 20 characters explaining why the trust boundary is removed (e.g., `"controlled environment with no internet access"`). Expressions and short strings are rejected by the compiler. Update any workflow using `dangerously-disable-sandbox-agent: true` to a descriptive justification string.
- **`user-invokable` and `disable-model-invocation` removed** — These Copilot-specific fields have been removed from the gh-aw schema and now produce a validation error. Remove them from any `.github/workflows/*.md` frontmatter.
- **Milestone cache scoped per owner/repo** — `assign_milestone` lookups in the `safe_outputs` job no longer bleed across repositories in multi-repo runs. No workflow change needed.
- **SHA-pinning for `steps:` workflow setup-cli** — The emitted `setup-cli` step in `steps:`-based compiled workflows now receives a SHA pin, aligning supply chain security posture with standard compiled workflows. Recompile affected workflows to pick up the pin.
- **Failure-issue permission denials handled gracefully** — Workflows that lack `issues: write` no longer crash during failure reporting; permission-denied responses are now caught and reported as non-fatal warnings.
- **Timeout-specific failure messages** — Timeout failures now emit a distinct failure message separate from general failures, making timeout-caused run failures easier to diagnose in failure issues.

### Fixed in v0.77.5
- **Daily effective-token guardrail setup overhead/failures** — `max-daily-effective-tokens` guardrail setup (including `@actions/artifact`) now runs only when explicitly configured, avoiding unnecessary activation work and missing-dependency failures on workflows that do not use the guardrail.
- **Daily effective-token guardrail diagnostics** — Guardrail evaluation now emits structured diagnostics around run discovery, artifact selection/download, and effective-token accumulation, making skipped/blocked runs debuggable.
- **Project UTC offset rendering** — `.github/workflows/aw.json` can provide `utc` (falling back to `GH_AW_DEFAULT_UTC`) so rendered timestamps and expiration messages use a stable project offset instead of the runner's local clock.
- **`features.copilot-requests` migration** — Copilot token mode is now controlled by `permissions.copilot-requests: write`; use `gh aw fix --write` on old workflows that still use the feature flag.
- **`target-repo` respected by safe-output handlers** — Safe-output handlers now honor configured `target-repo` routing instead of assuming the root workspace repository.
- **Protected-files fallback reliability** — `create-pull-request` fallback-to-issue now pushes the branch before creating the review issue, preventing fallback issues from pointing at missing branches.
- **HEAD-only safe-output bundles** — `create_pull_request` fallback logic now handles bundles whose source ref only contains HEAD, avoiding fallback failure on narrow/shallow bundle inputs.
- **Anthropic WIF schema parity** — `engine.auth` JSON schema now includes Anthropic WIF fields, so WIF-configured Claude workflows validate correctly.
- **`assign_to_agent` safe-output resilience** — Assignment failures no longer fail the entire `safe_outputs` job when other outputs can continue.
- **Activation comment targeting** — Activation comments now use the correct repo/client and avoid firing on empty commits.

### Fixed in v0.76.1
- **`push_to_pull_request_branch` signed-push merge history** — The safe output is documented as append-only and now auto-linearizes merge commits before signed push, preventing failures on PR branches with merge history.
- **Codex threat-detection parsing** — Codex response-event logs are parsed correctly in threat-detection result processing.
- **Step name alignment drift** — Direct manifest reads are permitted and agent guidance was tightened so generated step names stay aligned with expected manifests.
- **Duplicate frontmatter scanning** — `ParseWorkflow` no longer scans frontmatter twice, improving compile performance on larger workflows.
- **`gh aw upgrade` source updates** — The upgrade flow now updates `uses:` references in source `.md` files rather than only updating `actions-lock.json`.
- **Branch-equals-base guard for code-push safe outputs** — `create_pull_request` / `push_to_pull_request_branch` now reject pushes when detected branch equals base branch, preventing accidental writes to the base branch under confused event contexts.
- **Incremental push diff sizing** — `push_to_pull_request_branch` excludes merged upstream commits from `diffSize`, avoiding false oversized-diff failures after an agent merges base into a stale PR branch.
- **Safe-output `@filepath` rejection** — Safe-output MCP tool calls now reject local `@filepath` references with a clear error instead of accepting ambiguous runner-local paths.
- **Protected-files default** — The default policy for code-push safe outputs is now `request_review` (preserve PR plus `REQUEST_CHANGES`) rather than hard-blocking by default.

### Fixed in v0.74.8
- **PR review submission 422s** — Safe-output PR review submission no longer fails silently on common 422 cases; errors are surfaced for diagnosis.
- **Deprecated schema fields** — Deprecated frontmatter fields marked in JSON schema now emit warnings via a generic schema walker instead of requiring hand-written validators per field.
- **Validation diagnostics** — Validation errors include source `file:line:col` context and fuzzy "Did you mean?" suggestions for common engine, event, permission, and MCP type typos.
- **`tools.github.allowed-repos: current`** — Reusable/generated workflows can scope GitHub MCP guard policies to the current repository without hard-coding `owner/repo`.
- **HTTP(S) workflow import** — `gh aw add` / `add-wizard` can import workflows from arbitrary HTTP(S) URLs and JSON workflow definitions, with docs for mapping imported fields.
- **`network.allowed-input`** — Reusable workflows can expose a caller-extensible `network_allowed` input that unions caller domains/ecosystems into the compiled baseline.
- **`patch-diff.githubusercontent.com` allowlist** — `network.allowed: [github]` includes the PR diff host, so workflows can fetch PR diffs without custom domain exceptions.
- **OTLP telemetry fields** — OTLP export fills `service.version`, `gen_ai.response.finish_reasons`, total token usage, and distinguishes timeouts from failures.

### Fixed in v0.74.4
- **Submodule credential leak** — Compiled lock files using `persist-credentials: false` on checkout steps failed to scrub credentials when submodules were present. New `checkout.clean-git-credentials: true` option explicitly removes git credentials post-checkout. Workflows with submodules should add this option and recompile.
- **`add_comment` allowed-mentions ignored** — The `allowed-mentions` config was not being passed through to the safe-outputs layer, causing all mentions to be escaped. Now correctly applied.
- **`update_pull_request.update_branch` hard failure** — Workflow-permission errors from branch-update calls previously failed the job. Now treated as warnings; the branch-update step is skipped gracefully.
- **`create_pull_request` spurious chaos fallback** — A branch-already-exists condition was incorrectly triggering the chaos fallback path. Now handled correctly.
- **Repo-memory `MaxFileSize` raised** — Default raised from 10 KB to 100 KB, unblocking repo-memory analysis of real-world source files. No configuration change needed; recompile to pick up the new default.
- **Automatic `pull-requests: read` inference** — The compiler now infers `pull-requests: read` for activation jobs that include Vale pre-steps using `gh pr diff`. Recompile affected workflows to pick up the inferred permission automatically.
- **`@copilot` mention preservation in `add-comment`** — Distinct from the `allowed-mentions` config pass-through fix: `@copilot` is now auto-preserved even when not listed in `allowed-mentions`. Workflows that prefix review summaries with `@copilot ` to trigger Copilot follow-up (e.g., adversarial PR reviewer skills) previously had the mention escaped unless `allowed-mentions` was set explicitly.
- **`conclusion` job static concurrency** — The `conclusion` job used a static concurrency group that caused random cancellations when running parallel `workflow_dispatch` invocations (e.g., batch dispatch loops or matrix-style triggering). Concurrency is now per-run; parallel dispatches no longer cancel each other.
- **Auto-hoist `run:` block expressions** — The compiler now automatically hoists `${{ … }}` expressions in `run:` blocks to `env:` bindings (and applies the same transform to `safe_jobs:` step env vars). Previously, run-script guardrails rejected expressions in `run:` and authors had to rewrite manually. No source change needed — recompile and the codemod applies. See SKILL.md "Token injection hardening" for the security rationale.

### Fixed in v0.72.1
- **`&&` expression corruption** — Compiler HTML-escaped `&&` to `\u0026\u0026` inside `${{ }}` expressions in AWF config JSON, breaking workflow parsing.
- **safe-outputs permission regression** — When `update-project` appeared alongside `add-comment`/`add-labels`, the minted App token was incorrectly downgraded to `issues:read` instead of `issues:write`.
- **Conclusion comment false success** — The `conclusion` job reported ✅ even when `safe_outputs` failed (e.g., 422 on PR review submission).
- **COPILOT_API_KEY over-billing** — The dummy `byok-key` placeholder was causing 10–100x premium request over-billing.
- **Firewall binary 404** — v0.71.x referenced a non-existent `gh-aw-firewall` version. v0.72.1 ships firewall v0.25.29.

### Fixed in v0.71.5
- **Claude engine crash** — `CLAUDE_CODE_DISABLE_FAST_MODE=1` now set automatically (Claude Code 2.1.120+ compatibility).
- **`engine.env` multi-line values** — Block-scalar `engine.env` values (written with `>-`) now compile correctly.
- **`engine.env` `needs` expressions** — Custom job references in `engine.env` values now wired into agent job `needs` list.
- **`gh aw upgrade` false BYOK warning** — No longer strips `COPILOT_PROVIDER_API_KEY`/`COPILOT_PROVIDER_BEARER_TOKEN`.
- **Confused-deputy false positive** — Auto-detects `[bot]`-authored comments and skips the guard.

### Fixed in v0.68.3
- **Model-not-supported detection** — Workflows stop retrying and surface a clear error instead of spinning indefinitely.
- **`engine.max-turns` in shared imports** — Now correctly preserved through import chain.

### Historical (v0.62.2)
- **`min-integrity` compiler bug** — Hardcoded `min-integrity` emitted an incomplete guard policy (missing `repos` field) that crashed the MCP Gateway. Fixed in later versions — verify your lock file includes `determine-automatic-lockdown`.
