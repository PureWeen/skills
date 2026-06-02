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

## Version-Specific Bug History

These are bugs that were fixed. If you encounter them, upgrade to the version indicated.

### Fixed in v0.77.5
- **ET guardrail requires explicit configuration** — The daily effective-workflow (ET) guardrail and its `@actions/artifact` client setup are now only activated when explicitly configured in the workflow. Previously, the guardrail could run overhead even in workflows that did not use it. Workflows that rely on the ET guardrail must now add explicit configuration to opt in.
- **`@actions/artifact` install for ET guardrail** — Resolved a missing dependency that caused failures when the daily effective-workflow guardrail was enabled. If you saw artifact-client errors after opting into the ET guardrail, recompile to pick up the fix.

### Fixed in v0.76.1
- **`push-to-pull-request-branch` push failures on merge history** — Branches with merge commits previously caused signed-push failures. The safe output now auto-linearizes merge commits before pushing. No workflow-source change is needed; recompile to pick up the behavior.

### Fixed in v0.74.8
- **`patch-diff.githubusercontent.com` blocked in `network.allowed: [github]`** — Workflows that needed to fetch PR diffs from `patch-diff.githubusercontent.com` had to add it as a separate custom domain. It is now included in the `github` named domain group automatically. Recompile to remove any manual entries.
- **Fuzzy validation errors** — Compiler validation errors now include `file:line:col:` positioning and "Did you mean?" suggestions for mistyped engine names, events, permissions, and MCP types. No workflow change needed.

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
