---
name: gh-aw-guide
description: >-
  Comprehensive guide for building and maintaining GitHub Agentic Workflows (gh-aw).
  Covers architecture, security boundaries, fork handling, safe outputs, anti-patterns,
  compilation, and troubleshooting. Use when creating or editing gh-aw workflow .md files,
  writing safe-outputs configurations, configuring fork PR handling, setting up integrity
  filtering, debugging "why doesn't my workflow trigger", or any task involving
  .github/workflows/*.md or .lock.yml files, resolving .lock.yml merge conflicts. Also use when asked about
  slash commands, pre-agent-steps, protected files, or agentic workflow security.
---

# gh-aw (GitHub Agentic Workflows) Guide

This skill provides a complete reference for building, securing, and maintaining GitHub Agentic Workflows. It covers the gh-aw platform's architecture, security model, and all available features.

> **Assumes latest stable release.** Run `gh aw --version` to verify. For migration help with older workflows, see [`references/migrations.md`](references/migrations.md).

## Quick Start

gh-aw workflows are authored as `.md` files with YAML frontmatter, compiled to `.lock.yml` via `gh aw compile`. The lock file is auto-generated — **never edit it manually**. If `.lock.yml` has merge conflicts, resolve conflicts in the source `.md` first, then accept either side for `.lock.yml` and run `gh aw compile` to regenerate both `.lock.yml` and `.github/aw/actions-lock.json`.

```bash
# Compile after every change to the .md source
gh aw compile .github/workflows/<name>.md

# This updates:
# - .github/workflows/<name>.lock.yml (auto-generated)
# - .github/aw/actions-lock.json
```

**Always commit the compiled lock file alongside the source `.md`.**

### CLI Commands

```bash
gh aw compile <name>          # Compile .md → .lock.yml
gh aw run <name>              # Trigger a workflow_dispatch run
gh aw run <name> --ref main   # Run on a specific branch
gh aw status                  # List all workflows and their status
gh aw trial ./<name>.md --clone-repo owner/repo  # Test a workflow before merging to main
gh aw audit <run-id>          # Analyze a completed workflow run
gh aw logs <name>             # View execution logs
gh aw logs --filtered-integrity  # Inspect DIFC_FILTERED integrity events
gh aw lint                        # Validate .lock.yml via actionlint (no recompile)
gh aw lint --shellcheck --pyflakes  # Lint with extra checks
gh aw upgrade                 # Upgrade gh-aw CLI extension
gh aw experiments             # Read A/B experiment state
gh aw experiments analyze     # Statistical analysis of experiment results
gh aw compile --approve       # Approve safe-update manifest changes (also on `run`, `upgrade`)
# For deprecated flags (e.g., --safe-update → --approve), see references/migrations.md
```

**`gh aw trial`** — Test workflows that aren't on main yet. Creates a temporary private repo, installs the workflow, and runs it. Essential for validating new workflows before merge, since `workflow_dispatch` requires the lock file on the default branch.

> ⚠️ **`imports:` gotcha**: Trial only has files available from the cloned repo at HEAD. If your workflow uses `imports:` referencing shared files (e.g., `shared/review-shared.md`) that haven't been committed to the source repo yet, trial will fail because those files don't exist in the trial context. **Workaround:** commit shared files to the source repo first, or test on a real branch with `gh aw run --ref` instead.

**`gh aw run --ref`** — Trigger a workflow on a specific branch. The workflow must already exist on that branch (registered by GitHub after the lock file is pushed).

## 🚨 Before You Build: Prefer Built-in gh-aw Features

**CRITICAL RULE:** Before implementing any trigger, output, scheduling, or interaction mechanism in a gh-aw workflow, check whether gh-aw has a built-in feature that does it. gh-aw extends GitHub Actions with many convenience features — manually reimplementing them is always worse (more code, more bugs, missing platform integration like emoji reactions, sanitized inputs, and noise reduction).

### Step 1: Check the anti-patterns table below
### Step 2: If not listed, check the [triggers reference](https://github.github.com/gh-aw/reference/triggers/), [frontmatter reference](https://github.github.com/gh-aw/reference/frontmatter/), and [safe-outputs reference](https://github.github.com/gh-aw/reference/safe-outputs/)
### Step 3: If a built-in exists, use it. If not, proceed with manual implementation.

### Anti-Patterns: Manual Reimplementations to Avoid

| If you're about to implement... | Use this built-in instead | Docs |
|---------------------------------|--------------------------|------|
| `issue_comment` + `startsWith(comment.body, '/cmd')` | `slash_command:` trigger | [Command Triggers](https://github.github.com/gh-aw/reference/command-triggers/) |
| Manual emoji reaction on triggering comment | `reaction:` field under `on:` | [Frontmatter](https://github.github.com/gh-aw/reference/frontmatter/) |
| Posting "workflow started/completed" status comments | `status-comment: true` under `on:` | [Frontmatter](https://github.github.com/gh-aw/reference/frontmatter/) |
| Fixed cron schedule (`0 9 * * 1`) for non-critical timing | `schedule: weekly on monday around 9:00` (fuzzy) | [Triggers](https://github.github.com/gh-aw/reference/triggers/) |
| Manual `if:` to skip bot-authored PRs | `skip-bots:` under `on:` | [Triggers](https://github.github.com/gh-aw/reference/triggers/) |
| Manual `if:` to skip by author role | `skip-roles:` under `on:` | [Triggers](https://github.github.com/gh-aw/reference/triggers/) |
| Manual label check + removal for one-shot commands | `label_command:` trigger | [Triggers](https://github.github.com/gh-aw/reference/triggers/) |
| Editing old comments to collapse them | `hide-older-comments: true` on `add-comment:` | [Safe Outputs](https://github.github.com/gh-aw/reference/safe-outputs/) |
| Creating no-op report issues | `noop: report-as-issue: false` | [Safe Outputs / Monitoring](https://github.github.com/gh-aw/patterns/monitoring/) |
| Auto-closing older issues from same workflow | `close-older-issues: true` on `create-issue:` | [Safe Outputs](https://github.github.com/gh-aw/reference/safe-outputs/) |
| Disabling workflow after a date | `stop-after:` under `on:` | [Triggers](https://github.github.com/gh-aw/reference/triggers/) |
| Manual approval gating | `manual-approval:` under `on:` | [Triggers](https://github.github.com/gh-aw/reference/triggers/) |
| Search-based skip logic in `steps:` | `skip-if-match:` / `skip-if-no-match:` under `on:` | [Triggers](https://github.github.com/gh-aw/reference/triggers/) |
| Locking issues to prevent concurrent edits | `lock-for-agent: true` under trigger | [Triggers](https://github.github.com/gh-aw/reference/triggers/) |
| Manually hiding agent comments | `hide-comment:` safe output | [Safe Outputs](https://github.github.com/gh-aw/reference/safe-outputs/) |
| Custom post-processing jobs for agent output | `safe-outputs.jobs:` custom jobs with MCP tool access | [Custom Safe Outputs](https://github.github.com/gh-aw/reference/custom-safe-outputs/) |
| Wrapping GitHub Actions as agent-callable tools | `safe-outputs.actions:` action wrappers | [Custom Safe Outputs](https://github.github.com/gh-aw/reference/custom-safe-outputs/) |
| Triggering CI on agent-created PRs | `github-token-for-extra-empty-commit:` on `create-pull-request` | [Triggering CI](https://github.github.com/gh-aw/reference/triggering-ci/) |
| No guard against agent approving PRs | `allowed-events: [COMMENT]` on `submit-pull-request-review`; or `[COMMENT, REQUEST_CHANGES]` with `supersede-older-reviews: true` to auto-dismiss stale blocking reviews | [Safe Outputs](https://github.github.com/gh-aw/reference/safe-outputs-pull-requests/) |
| Stale blocking reviews from previous `/review` runs | `supersede-older-reviews: true` on `submit-pull-request-review` — dismisses older same-workflow `REQUEST_CHANGES` reviews after posting replacement | [Safe Outputs](https://github.github.com/gh-aw/reference/safe-outputs-pull-requests/) |
| Merging PRs via shell `gh pr merge` in post-steps | `merge-pull-request` safe output — executes in the safe-outputs job with proper permissions | [Safe Outputs](https://github.github.com/gh-aw/reference/safe-outputs/) |
| Manually updating existing bot comments (delete + repost) | `hide-older-comments: true` on `add-comment` — collapses previous comments before posting new | [Safe Outputs](https://github.github.com/gh-aw/reference/safe-outputs/) |
| Keeping PR branch up-to-date with base manually | `update-branch: true` on `update-pull-request` — merges latest base into PR branch | [Safe Outputs](https://github.github.com/gh-aw/reference/safe-outputs-pull-requests/) |
| Configuring the GitHub CLI proxy mode | `tools.github.mode: gh-proxy` — official config; old `cli-proxy` feature flag is deprecated | [Engines](https://github.github.com/gh-aw/reference/engines/) |
| `slash_command:` without `events:` filter (subscribes to ALL comment events) | `events: [pull_request_comment]` or `events: [issue_comment]` | [Command Triggers](https://github.github.com/gh-aw/reference/command-triggers/) |
| `cancel-in-progress: true` on `slash_command:` workflows | `cancel-in-progress: false` — non-matching events cancel in-progress agent runs | [Concurrency](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/using-concurrency) |
| Using `pull_request` trigger for agentic workflows | `slash_command:`, `label_command:`, or `schedule` — `pull_request` causes the "Approve and run" gate for ALL workflows | [Triggers](https://github.github.com/gh-aw/reference/triggers/) |

**Note:** gh-aw is actively developed. If a capability feels like something a framework would provide natively, check the reference docs — it probably exists even if it's not in this table yet.

For full architecture, security, fork handling, safe outputs, and troubleshooting details, see the [official gh-aw docs](https://gh.io/gh-aw).

## Common Patterns

### Pre-Agent Data Prep (the `steps:` pattern)

Use `steps:` for any operation requiring GitHub API access that the agent needs:

```yaml
steps:
  - name: Fetch PR data
    env:
      GH_TOKEN: ${{ github.token }}
      PR_NUMBER: ${{ github.event.pull_request.number || github.event.issue.number || inputs.pr_number }}
    run: |
      gh pr view "$PR_NUMBER" --json title,body > pr-metadata.json
      gh pr diff "$PR_NUMBER" --name-only > changed-files.txt
```

### Payload Sanitization

Comment bodies, issue titles, and PR descriptions are **user-controlled untrusted input**. In pre-agent `steps:`, always use `steps.<id>.outputs.text` (sanitized) instead of raw `${{ github.event.comment.body }}`. Within the agent job itself, container sandboxing limits the write surface (process escape), but does **not** prevent prompt injection (XPIA) — the model still reads untrusted content and may act on it. Rely on the platform's built-in XPIA sanitization and pair with tight `safe-outputs:` as defense-in-depth.

> 🛑 **Recursive workflow triggering**: Actions performed via `GITHUB_TOKEN` do **NOT** fire new workflow events (prevents infinite loops). Actions via GitHub App installation tokens or PATs **DO** fire events. This is why `github-token-for-extra-empty-commit:` requires a PAT — `GITHUB_TOKEN` pushes won't trigger CI on agent-created PRs.

### Safe Outputs (Posting Comments)

```yaml
safe-outputs:
  add-comment:
    max: 1                      # counts comments — 1 comment per run (blast-radius cap)
    hide-older-comments: true
    target: "*"    # Required for workflow_dispatch (no triggering PR context)
```

> **🚨 `max:` is type-specific — it does NOT uniformly mean "max tool calls".** What `max:` counts varies by safe-output type. Setting `max: 1` on `add-labels` thinking "one tool call per run" silently drops every label beyond the first (the agent batches multiple labels per call, but `max:` counts the total labels, not the call count). Always check the unit before setting it.

> **`max:` unit by safe-output type** (verify in [official docs](https://github.github.com/gh-aw/reference/safe-outputs/)):
>
> | Type | What `max:` counts | Default |
> |---|---|---|
> | `add-labels` | **labels** (sum across calls) | 3 |
> | `remove-labels` | labels | 3 |
> | `add-reviewer` | **reviewers** | 3 |
> | `hide-comment` | **comments** | 5 |
> | `add-comment` | comments (≈1 call) | 1 |
> | `create-pull-request-review-comment` | **inline review comments** | 10 |
> | `reply-to-pull-request-review-comment` | replies | 10 |
> | `resolve-pull-request-review-thread` | threads | 10 |
> | `upload-asset` | assets | 10 |
> | `autofix-code-scanning-alert` | autofixes | 10 |
> | `dispatch-workflow` | dispatches (calls) | 1 |
> | `set-issue-type` / `set-issue-field` | operations | 5 |
> | `update-project` | operations | 10 |
> | `close-pull-request` | closures | 10 |
> | `create-issue` / `update-issue` / `close-issue` | issues / updates / closures | 1 |
> | `create-pull-request` / `update-pull-request` / `push-to-pull-request-branch` | PRs / updates / pushes | 1 |
> | `submit-pull-request-review` | reviews | 1 |
> | `assign-*` / `unassign-*` / `link-sub-issue` | individual ops | 1 |
> | `create-discussion` / `update-discussion` / `close-discussion` | items | 1 |
> | `update-release` / `create-project` / `create-project-status-update` | items | 1 |
> | `upload-artifact` | uses **`max-uploads:`**, not `max:` | 1 |
> | `create-agent-session` | sessions | 1 |
> | `call-workflow` | tool invocations | 1 |
> | `noop` | messages | 1 |
> | `create-code-scanning-alert` / `missing-tool` / `missing-data` | items | unlimited |
>
> **Sizing principle**: `max:` is a blast-radius cap — it limits how many items of each type the agent can produce per run. The safe-outputs infrastructure handles HTTP 429/5xx retries independently; raising `max:` does not help retries, but lowering it below the legitimate item count silently drops items beyond the cap. **For multi-item types (`add-labels`, `add-reviewer`, all PR-review-comment types), do NOT set `max:` lower than the maximum item count any single run might emit — `add-labels: max: 1` on a labeler that emits `area-*` + `platform/*` will drop one of them every time.**

### Add Labels — Security Hardening

`add-labels:` accepts `allowed:` (glob allow-list) and `blocked:` (glob deny-list) — these are infrastructure-level filters that run before the agent's chosen labels are applied. **Always set `allowed:`** when the workflow has `roles: all` or otherwise accepts untrusted triggers, otherwise a prompt-injected agent can apply any label in the repo (including labels that trigger downstream automation like `approved-for-merge`, `needs-backport`, or `label_command:` triggers).

```yaml
safe-outputs:
  add-labels:
    max: 3                                # counts LABELS, not calls — see table above
    allowed: [area-*, platform/*, t/*]    # restrict to expected label families
    blocked: ["~*", "*[bot]"]             # deny patterns regardless of allowed
```

### Concurrency

Include all trigger-specific PR number sources. **Use `cancel-in-progress: false` for `slash_command:` and `label_command:` workflows** — a non-matching event (ordinary comment or benign label change) in the same concurrency group can cancel an in-progress matching run (the actual `/command`), killing the agent mid-execution:

```yaml
# For slash_command/label_command workflows — never cancel in-progress
concurrency:
  group: "my-workflow-${{ github.event.issue.number || github.event.pull_request.number || inputs.pr_number || github.run_id }}"
  cancel-in-progress: false

# For schedule/workflow_dispatch — include PR number when present
concurrency:
  group: "my-workflow-${{ inputs.pr_number || github.ref || github.run_id }}"
  cancel-in-progress: true
```

> ⚠️ **`workflow_dispatch` concurrency footgun.** If your workflow accepts an `inputs.pr_number` input, **include it in the concurrency group** — otherwise `github.ref` alone (typically `refs/heads/main` for every dispatch) puts all simultaneous PR-targeted dispatches into the same group, and `cancel-in-progress: true` will silently cancel a maintainer's in-flight `/review` of PR #100 when another maintainer dispatches against PR #200 seconds later. The example above isolates per-PR.

> ⚠️ **Pre-cancellation race**: Cancellation is asynchronous — GitHub sends `SIGTERM`, waits up to 7500ms, then `SIGKILL`. Already-running steps may complete. An agent that already posted a comment cannot un-post it. A `create-pull-request` that already ran cannot un-create the PR. **Concurrency is not a substitute for idempotency.**

### `slash_command:` Event Subscription

`slash_command:` compiles to broad event subscriptions — by default it listens to **all** comment-related events (issue open/edit, PR open/edit, every comment, every review comment, every discussion comment), then filters post-activation. This means:

- **Runner cost**: The pre-activation job runs on every matching event (~5-30s each), even when skipped. On busy repos this can be hundreds of skipped runs per day.
- **Actions UI noise**: Operators learn to ignore "skipped" runs and may miss real failures.
- **Concurrency collisions**: Non-matching events in the same concurrency group can cancel matching ones (see above).

**Always narrow `events:`** to the minimum needed:

```yaml
on:
  slash_command:
    name: review
    events: [pull_request_comment]  # Only PR comments, not issues/discussions
```

### The "Approve and Run Workflows" Gate

The `pull_request` trigger causes an "Approve and run workflows" button for first-time fork contributors. **This gate is dangerous, not protective**:

1. **Alert fatigue** — After clicking through dozens of legitimate first-time PRs, the click becomes muscle memory
2. **No per-workflow granularity** — A single click approves ALL gated workflows, including any `pull_request_target` workflows with full secrets
3. **No diff preview** — The UI shows no preview of what will execute or which secrets are exposed

**Design rule**: Assume the approval gate will always be clicked. The only safe workflows are ones that produce the same outcome whether the actor is trusted or untrusted. Prefer `issue_comment`/`slash_command:` (not subject to the gate) or `schedule`/`workflow_dispatch` over `pull_request` when possible.

### LabelOps

gh-aw provides label-based triggering patterns for both one-shot commands and persistent state tracking:

- **`label_command:`** — One-shot command triggered by applying a label. The label is auto-removed after the workflow fires, making it self-resetting. Use for operations like "apply this label to trigger a review".
- **`names:` filtering** — Filter label events to specific label names for persistent label-state awareness.
- **`remove_label: false`** — Keep the label after triggering (for persistent state markers rather than one-shot commands).

See the [LabelOps pattern guide](https://github.github.com/gh-aw/patterns/label-ops/) for detailed examples and best practices.

### Noise Reduction

Filter `pull_request` triggers to relevant paths and add a gate step:

```yaml
on:
  pull_request:
    paths:
      - 'src/**/tests/**'

steps:
  - name: Gate — skip if no relevant files
    id: gate
    if: github.event_name == 'pull_request'
    env:
      PR_NUMBER: ${{ github.event.pull_request.number }}
    run: |
      FILES=$(gh pr diff "$PR_NUMBER" --name-only | grep -E '\.cs$' || true)
      if [ -z "$FILES" ]; then
        echo "skip=true" >> "$GITHUB_OUTPUT"
      fi
  - name: Agent work
    if: steps.gate.outputs.skip != 'true'
```

Manual triggers (`workflow_dispatch`, `issue_comment`) should bypass the gate. Use step outputs rather than `exit 1` to avoid red ❌ checks on non-matching PRs — `exit 1` fails the job, which normalizes failures and masks real errors.

### Fork PR Checkout (workflow_dispatch)

For `workflow_dispatch` workflows that need to evaluate a PR branch, implement a checkout step that: (1) verifies the PR author has write access and rejects fork PRs, (2) checks out the PR branch, and (3) restores `.github/` and agent infrastructure from the base branch SHA — defense-in-depth even though the platform also does this restore automatically.

```yaml
steps:
  - name: Checkout PR and restore agent infrastructure
    env:
      GH_TOKEN: ${{ github.token }}
      PR_NUMBER: ${{ inputs.pr_number }}
    run: |
      set -euo pipefail
      # Reject cross-repository (fork) PRs
      IS_FORK=$(gh pr view "$PR_NUMBER" --json isCrossRepository --jq '.isCrossRepository')
      if [[ "$IS_FORK" == "true" ]]; then
        echo "::error::Fork PRs are not supported in this workflow"
        exit 1
      fi
      # Verify PR author has write access
      AUTHOR=$(gh pr view "$PR_NUMBER" --json author --jq '.author.login')
      PERM=$(gh api "repos/$GITHUB_REPOSITORY/collaborators/$AUTHOR/permission" --jq '.permission')
      if [[ "$PERM" != "admin" && "$PERM" != "write" && "$PERM" != "maintain" ]]; then
        echo "::error::PR author $AUTHOR has $PERM access — requires write+"
        exit 1
      fi
      gh pr checkout "$PR_NUMBER"
      # Restore trusted .github/ from base branch — must succeed or abort
      BASE_SHA=$(gh pr view "$PR_NUMBER" --json baseRefOid --jq '.baseRefOid')
      git fetch origin "$BASE_SHA" --depth=1
      git checkout "$BASE_SHA" -- .github/
      # .agents/ may not exist at base — guard separately to avoid aborting the script
      git checkout "$BASE_SHA" -- .agents/ 2>/dev/null || true
```

For `pull_request` + fork support (not `workflow_dispatch`): add `forks: ["*"]` to the trigger frontmatter. The platform automatically preserves `.github/` and `.agents/` as a base-branch artifact in the activation job, then restores them after `checkout_pr_branch.cjs` — fork PRs cannot overwrite agent infrastructure (gh-aw#23769, resolved).

### Operating Within a Fork

When you fork a repository, all workflow files come with it. Events inside your fork fire the workflows inside your fork, with your fork's secrets. This is separate from cross-fork PRs and is frequently a surprise.

**Guard pattern** — prevent workflows from running in forks (with a manual escape hatch):

```yaml
jobs:
  guard:
    if: ${{ github.event_name == 'workflow_dispatch' || !github.event.repository.fork }}
```

> ⚠️ **YAML gotcha**: Don't start a bare `if:` value with `!` — it's a YAML tag indicator. Always wrap in `${{ }}` to be safe.

> **Note:** This guard is for workflows running *inside* a forked repo — not for blocking cross-fork PRs. For fork PR protection, see Defense #5 under [Read-Only Contributor Write Surface](#read-only-contributor-write-surface): `if: github.event.pull_request.head.repo.fork == false`.

### Security-Critical Patterns

These patterns are the most commonly missed when building secure workflows. Use all where applicable.

**1. Role-based access control** — `roles:` controls who can trigger the workflow. Without it, any user (including the PR author) can trigger `/review` on a malicious PR designed to prompt-inject the reviewer. The default `[admin, maintain, write]` is injected automatically for workflows with "unsafe" events (issues, comments, PRs, discussions):

```yaml
on:
  slash_command:
    name: review
    events: [pull_request_comment]
    roles: [admin, maintain, write]  # Only committers can trigger — NEVER use 'all' unless you've audited every safe-output
```

> ⚠️ **`triage` role footgun**: `triage` is excluded from the default allowlist. A `label_command:` workflow (which requires triage to apply the label) will _fire_ but the activation job will _deny_ a triage user unless `roles:` is broadened. Add `triage` explicitly when triage users are the primary operators.

**2. Prevent accidental PR approvals** — always restrict review workflows; otherwise the agent can approve PRs and bypass branch protection rules (gh-aw#25439):

```yaml
safe-outputs:
  submit-pull-request-review:
    # COMMENT-only: no stale blocking reviews, safe for iterative /review re-runs
    allowed-events: [COMMENT]
    # Or allow REQUEST_CHANGES with supersede to auto-dismiss stale blocking reviews:
    # allowed-events: [COMMENT, REQUEST_CHANGES]
    # supersede-older-reviews: true
```

> **`supersede-older-reviews: true`** — When using `REQUEST_CHANGES`, set this to automatically dismiss older blocking reviews from the same workflow after posting a replacement. This solves the stale-review problem: without it, a `REQUEST_CHANGES` review persists even after the author fixes everything and re-runs `/review`, because gh-aw has no `dismiss-pull-request-review` safe output and the compiler rejects `pull-requests: write`. With `supersede-older-reviews`, the new review replaces the old one (best-effort). This makes `[COMMENT, REQUEST_CHANGES]` a viable option alongside `[COMMENT]`-only.

**3. Integrity filtering** — controls what content the agent can **see** (vs. `roles:` which controls who can **trigger**). The MCP gateway intercepts GitHub tool calls and filters content by author trust level before the AI engine sees it. Filtered items are logged as `DIFC_FILTERED` events — inspect them with `gh aw logs --filtered-integrity`.

| Level | Who qualifies |
|-------|--------------|
| `merged` | Merged PRs; commits on default branch (any author) |
| `approved` | `OWNER`, `MEMBER`, `COLLABORATOR`; non-fork PRs on public repos; all items in private repos; platform bots; `trusted-users` |
| `unapproved` | `CONTRIBUTOR`, `FIRST_TIME_CONTRIBUTOR` |
| `none` | All content including `FIRST_TIMER` and users with no association |
| `blocked` | Users in `blocked-users` — always denied, cannot be promoted |

**Defaults:** When `min-integrity` is omitted, the runtime's `determine-automatic-lockdown` step computes the level per event/actor/repo (see [`references/architecture.md`](references/architecture.md)). As a rough heuristic, public repos tend to land on `approved` and private repos on `none`, but this is **not a static guarantee** — always set it explicitly for security-sensitive workflows.

> ⚠️ **Private repos default to a permissive level.** "It's private so it's trusted" is a frequent misread — automatic lockdown on a private repo can resolve to `none`, allowing the agent to see content from any user with repo access (including read-only contractors and external collaborators). For private workflows with write-capable safe-outputs, **always set `min-integrity: approved` explicitly.**

```yaml
tools:
  github:
    min-integrity: approved        # Default for public repos — only trusted author content
    allowed-repos: "myorg/*"       # Scope to specific repos (optional; default "all")
    toolsets: [pull_requests, repos]
    trusted-users: [contractor-1]  # Elevate specific users to 'approved'
    blocked-users: [spam-bot]      # Unconditionally block specific users (always denied)
    approval-labels: [human-reviewed]  # Labels on issues/PRs that promote items to 'approved'
    # integrity-proxy: false        # Disable DIFC proxy for pre-agent gh CLI calls (default: true)
```

**`allowed-repos` scoping:** Restricts which repositories' content the agent can see. Accepts `"all"` (default), `"public"`, or an array of patterns: `["myorg/*", "partner/repo"]`. When specified alongside `min-integrity`, both fields must be present.

**`approval-labels`:** Label names that promote an issue or PR to `approved` integrity regardless of author association. Useful as a human-review gate: a maintainer applies the label, and the agent can then see and act on it. Accepts a static list or a GitHub Actions expression.

**`trusted-users` / `blocked-users` / `approval-labels` — GitHub Actions expressions:** All three fields accept GitHub Actions expressions (e.g., `${{ vars.TRUSTED_CONTRACTORS }}`) in addition to static arrays. This enables centralized management via repository variables or environment variables (`GH_AW_GITHUB_*`).

**Centralized management via `GH_AW_GITHUB_*` environment variables:** Organization-wide integrity settings can be distributed via Actions environment variables prefixed with `GH_AW_GITHUB_` (e.g., `GH_AW_GITHUB_TRUSTED_USERS`). These are merged with per-workflow frontmatter.

**`integrity-proxy: false`:** Disables the DIFC proxy for pre-agent `gh` CLI calls in workflow steps. Only use when you deliberately want to bypass integrity checks for a non-agentic step. Does not affect the MCP gateway filtering.

**Effective integrity computation order:** An item's final integrity level is determined by (highest wins): `blocked-users` (forced deny) → `trusted-users` (forced `approved`) → `approval-labels` (promote to `approved`) → endorsement/disapproval reactions (see below) → author association default.

**Reaction-based integrity (`features.integrity-reactions: true`):** Reactions on issues/PRs/comments can dynamically promote or demote integrity:

```yaml
features:
  integrity-reactions: true
tools:
  github:
    min-integrity: approved
    endorsement-reactions: [THUMBS_UP, HEART]   # Promote item to 'approved' (default when feature enabled)
    disapproval-reactions: [THUMBS_DOWN, CONFUSED]  # Demote item integrity (default when feature enabled)
    endorser-min-integrity: approved            # Reactor's own integrity required for reaction to count (default: approved)
    disapproval-integrity: none                 # Integrity level assigned on qualifying disapproval (default: none)
```

**Interaction with `roles:`:**

| `roles:` | `min-integrity` | Effect |
|----------|----------------|--------|
| Default `[admin, maintain, write]` | `approved` | **Most restrictive.** Only trusted actors trigger; agent sees only trusted content |
| Default | `unapproved`/`none` | Trusted actors only, but agent reads community content. Good for post-merge scans |
| `all` | `approved` | **Two-layer defense.** Any actor triggers, but agent only sees trusted content |
| `all` | `none` | **Widest exposure.** Must pair with minimal `safe-outputs` — only remaining constraint |

**4. CI triggering + protected file safety** for agent-created PRs — `GITHUB_TOKEN` pushes don't trigger CI; a PAT/App token is required. `protected-files` controls what happens when the agent modifies package manifests or `.github/`:

```yaml
safe-outputs:
  create-pull-request:
    github-token-for-extra-empty-commit: ${{ secrets.PAT_OR_APP_TOKEN }}  # Required to trigger CI
    protected-files: fallback-to-issue   # Create issue instead of failing if agent touches .github/ or package manifests
    # protected-files: blocked (default) | allowed (disables protection)
```

**5. Fork PR checkout for `workflow_dispatch`** — the platform's `checkout_pr_branch.cjs` is skipped for `workflow_dispatch`, so you must implement a checkout step that verifies write access, rejects fork PRs, and restores trusted `.github/` from the base branch. See the [Fork PR Checkout](#fork-pr-checkout-workflow_dispatch) pattern above for a complete example.

**6. XPIA hardening** — Cross-prompt injection (XPIA) sanitization paths are hardened. `disable-xpia-prompt` is **rejected at compile time in strict mode** — do not use it. The runtime handles XPIA protection by default.

### Idempotency and the Edited-Comment Time-Bomb

**Slash command workflows MUST be idempotent.** Treat every activation as if the same command might already be running for the same target. Check before acting, claim a lock, no-op if already in progress or done.

gh-aw provides `lock-for-agent: true` to automatically lock/unlock the issue during execution, but use with caution — it prevents genuine users from interacting on the issue/PR while the workflow runs. In public repos, locking is visible to all watchers and may be perceived as moderation — use sparingly and consider a status comment explaining the temporary lock purpose.

**State tracking for scheduled pollers:** Use comment-based state to track what's been processed. Edit a comment's visible markdown to reflect status (⏳ in progress / ✅ done), and append invisible `<!-- state-machine -->` HTML comments as an append-only audit trail. This gives human-readable status and machine-parseable history in one artifact.

> 🛑 **The edited-comment time-bomb**: An attacker can edit a 6-month-old comment on a closed issue or PR, injecting `/command` or any payload — `issue_comment.edited` fires TODAY against today's secrets, today's `permissions:`, today's `safe-outputs:`. The workflow has no concept of "this comment was created when our security model was different." For raw `issue_comment`, use `types: [created]` — add `edited` only if you've designed for this attack vector.

### Read-Only Contributor Write Surface

> **What the agent can do is determined by `permissions:` and `safe-outputs:` — NOT by the actor who fired it.** When a workflow accepts a read-only contributor as the trigger (`roles: all`), that contributor effectively gets bot-level write access to anything the workflow grants the agent.

**What a read-only user can fire:**

| Action | Can fire? |
|--------|----------|
| Open an issue, comment, react with emoji | ✅ |
| `/slash-command` in any comment/body they author | ✅ |
| Open a PR (from fork) | ✅ |
| Apply a label | ❌ (requires triage) |
| Invoke `workflow_dispatch` | ❌ (requires write) |
| Click "Approve and run workflows" | ❌ (requires write) |

**Defenses, in priority order:**
1. Leave `roles:` at its default `[admin, maintain, write]`
2. Minimize `permissions:` to the smallest set the agent needs
3. Minimize `safe-outputs:` to only the mutations the workflow needs
4. For PR-touching workflows: never check out the PR head SHA in a job that has secrets
5. Add an explicit fork guard: `if: github.event.pull_request.head.repo.fork == false`
6. Configure `min-integrity` to control what content the agent can see

## Trigger Selection Guide

Choose the right trigger for your workflow. Triggers are grouped by recommended usage level.

### ✅ Recommended

| Trigger | Best for | Key advantage |
|---------|----------|---------------|
| `workflow_dispatch` | Manual escape hatch, debugging, ad-hoc runs | Write+ required; auto-paired with most triggers. ⚠️ Branch selection is user-controlled — a write user can dispatch against a stale branch with weaker `permissions:`, different `safe-outputs:`, or a friendlier prompt |
| `schedule` | Periodic housekeeping, polling-based PR operations | Best concurrency story; no event spamming; no approval gate |
| `labeled` / `label_command:` | Human-in-the-loop gate via label application | Triage+ required to apply label; one-shot with auto-remove. Consider `min-integrity: none` only when labels can be applied exclusively by write+ users — the label gate controls *who triggers*, but integrity controls *what content the agent sees*. Verify label permissions before relaxing integrity filtering |
| `issues` | Community-facing issue workflows | Immediate; `roles: all` acceptable **only with read-reply safe-outputs** (`add-comment`, `add-labels` with `allowed:`, `update-issue`, `close-issue`). **Never** pair `roles: all` with `dispatch-workflow`, `create-pull-request`, `push-to-pull-request-branch`, `create-agent-session`, `merge-pull-request`, or any output that creates persistent artifacts or triggers downstream pipelines |
| `release` / `milestone` | Post-release/milestone automation | Trusted trigger (write+) |

### ⚠️ Use with Caution

| Trigger | Headline risk |
|---------|--------------|
| `push` | **Always** use explicit `branches:` — bare `on: push` fires on every branch including bot/dependency/codeflow branches. The trigger most likely to turn a PoC into a billing surprise. Rapid pushes (rebasing, force-pushing) stack runs unless `cancel-in-progress: true` |
| `issue_comment` / `slash_command:` | Broad underlying subscription; concurrency catastrophe; edited-comment time-bomb |
| `pull_request_review` | Fires for ALL review types including COMMENT from any user, not just approvals |
| `discussion` / `discussion_comment` | Most-open untrusted-input surface; no approval gate; lower visibility than issues |

### `pull_request.synchronize` Gotchas

`synchronize` fires once per push to a PR branch (not per commit). Things that do **NOT** fire `synchronize`:

- **Draft → ready-for-review**: Fires `ready_for_review`, not `synchronize`. Workflows using default `types: [opened, synchronize, reopened]` won't re-run CI when a draft is marked ready
- **Base-ref edits**: Changing the PR's base branch fires `edited` (with `changes.base`), not `synchronize`
- **Pushes to the base branch**: Someone merging to `main` while your PR targets `main` does NOT fire `synchronize` on your PR — it fires `push` on `main`. Your CI won't re-run against the new base unless you push to your branch
- **Approval dismissal**: Branch protection's "Dismiss stale approvals on new commits" fires on the same head-SHA-changed event. A force-push that doesn't change file contents (rebase onto current `main`) still invalidates all prior approvals

### ⛔ Avoid

| Trigger | Why |
|---------|-----|
| `pull_request` | Causes "Approve and run" gate for ALL workflows; clicking approves everything including `pull_request_target` with full secrets. Prefer `slash_command:`, `schedule`, or `label_command:` |
| `pull_request_target` | Runs on base ref with full secrets and write token — most exploited vulnerability class. Never check out PR head SHA |
| `workflow_run` | `pull_request_target`'s quieter sibling — launders untrusted fork artifacts into privileged context with no approval gate. Classic pwn: sandboxed `pull_request` workflow uploads artifacts (e.g., `coverage.json`), then `workflow_run` downloads and acts on them with full secrets. Artifact may contain shell-injection or prompt-injection payloads. No UI signal connects the upstream PR to the downstream run. **Treat all downloaded artifacts as untrusted** |

### Design Principles

1. **Deterministic by default.** Use deterministic Actions and reusable workflows; agentic workflows only when the input is unstructured or AI unlocks a capability deterministic code cannot provide.
2. **Limitations ARE the security model.** Don't engineer bypasses (`pull_request_target` for write access, PAT pools to evade bot attribution, `workflow_run` to escape approval gates, `roles: all` to widen the actor pool). When a boundary blocks a legitimate goal, escalate to platform owners.
3. **Limit the agent job to agent-suitable work.** Keep filtering/skipping in pre-agent steps. Execute deterministic scripts before and after the agent job.
4. **Apply least privilege on every dimension.** Minimum `permissions:`, `safe-outputs:`, `network.allowed:`, secrets, `tools:`. The agent sandbox limits the write surface (prevents process escape) but does not neutralise prompt injection — untrusted input must still be treated as adversarial. The same operation in pre/post-agent steps runs on the runner host with full secret access.
5. **Mind the signal-to-noise ratio.** Convenience triggers compile to broad subscriptions. Every event spawns a workflow run consuming a runner slot. The activation step must be cheap, and the worst-case invocation rate must be estimated and acceptable.
6. **Understand the `GITHUB_TOKEN` recursion boundary.** Actions via `GITHUB_TOKEN` do NOT fire new workflow events (prevents infinite loops). GitHub App tokens and PATs DO fire events. This is by design — it's why `github-token-for-extra-empty-commit:` needs a PAT, and why bot comments via `GITHUB_TOKEN` don't trigger `issue_comment` workflows.

### Frontmatter Features

```yaml
source: "githubnext/agentics/workflows/ci-doctor.md@v1.0.0"  # Track workflow origin
private: true                                                    # Prevent installation via gh aw add
resources:                                                       # Companion files fetched with gh aw add
  - triage-issue.md
  - shared/helper-action.yml
labels: ["automation", "ci"]                                     # For gh aw status --label filtering
checkout: false                                                  # Skip repo checkout (for workflows that only use MCP/API, no source needed)

runtimes:                    # Override default runtime versions
  dotnet:
    version: "9.0"
  node:
    version: "22"

imports:                     # APM package dependencies
  - uses: shared/apm.md
    with:
      packages:
        - microsoft/apm-sample-package

on:
  needs: pre_activation       # Declare dependency on a custom pre_activation job for credential supply
                              # Enables sourcing GitHub App credentials from upstream job outputs
```

**`on.needs:`** — Express dependencies on custom `pre_activation`/`activation` jobs, enabling GitHub App credentials to be sourced from upstream job outputs. See also `safe-outputs.needs` for credential-supply dependencies in the safe-outputs job.

**`merge-pull-request` safe output** — Merge a PR directly as a safe output. Executes in the safe-outputs job with proper write permissions, not inside the agent container.

**Automatic `pull-requests: read` permission inference** — The compiler now automatically infers `pull-requests: read` for activation jobs that include Vale pre-steps using `gh pr diff`. Previously this required a manual `permissions:` block; workflows using Vale will pick it up on recompile.

**`tools.github.mode: gh-proxy`** — Configure the GitHub CLI proxy feature. The deprecated `cli-proxy` feature flag is scheduled for removal; migrate to this form:

```yaml
tools:
  github:
    mode: gh-proxy
```

**Claude engine** — The Claude engine has two permission modes: `acceptEdits` (default — agent proposes edits that the safe-outputs layer validates) and `bypassPermissions` (activated when unrestricted bash `bash: "*"` is granted — agent executes directly).

**Reasoning message rendering** — Agent reasoning/thinking content (Claude extended thinking blocks, Codex `thinking` sections, Copilot `reasoning_text`) renders with a distinct `◯` open circle icon and italic styling in the workflow UI. This makes agent thought processes visible and distinguishable from regular output — useful when diagnosing why the agent made a particular decision.

**`engine.bare` frontmatter field** — Disable automatic context loading for supported engines, giving full control over what the AI agent sees. Use `bare: true` with `copilot` (suppresses `AGENTS.md` and user instructions) or `claude` (suppresses `CLAUDE.md` memory files). Also supported by `codex` and `gemini`. Unsupported engines emit a compiler warning.

**`engine.mcp` import support** — `engine.mcp.tool-timeout` and `session-timeout` can be imported from shared workflows. Shared workflows wrapping slow MCP servers can declare these once, and consumers inherit the values automatically. Consumer-declared values take precedence.

**`small` model alias for sub-agents** — Inline sub-agent blocks use the `small` model alias by default, reducing cost and latency for lightweight agent tasks.

**A/B experimentation infrastructure** — Full experiment lifecycle: define variants, run round-robin, collect per-run state, and analyze results statistically. New commands: `gh aw experiments` (read state from storage branches) and `gh aw experiments analyze` (significance testing, sample-size tracking). Experiment state can be stored in cache or a dedicated repo branch.

**`pre-steps:` frontmatter field** — Inject steps that run _before_ checkout and the agent, inside the same job. Recommended for token-minting actions (e.g., `actions/create-github-app-token`, `octo-sts`) for cross-repo checkout. The minted token stays in the same job, avoiding the masking issue when crossing job boundaries.

**Model-not-supported detection** — When a model is unavailable or not supported by your Copilot plan, the workflow surfaces a clear error without retrying.

**`checkout` field in shared imports** — Shared importable workflows support a `checkout` field for controlling which ref is checked out during import.

**`env:` field in shared imports** — Shared importable workflows support an `env:` field for passing environment variables to imported workflows.

**`push-to-pull-request-branch` target-repo** — The compiler honors the `target-repo` field in shared PR checkout steps for `push-to-pull-request-branch`.

**Bundle mode for safe-output patches** — Bundle mode is the default for safe-output patch packaging.

**`gh aw lint`** — Fast lock-file validation using [actionlint](https://github.com/rhysd/actionlint) directly against existing `.lock.yml` files — no recompile needed. Supports `--dir`, explicit file paths, and optional `--shellcheck`/`--pyflakes` checks. Ideal as a lightweight CI gate.

**Fuzzy "Did You Mean?" suggestions in validation errors** — When `gh aw compile` detects a mistyped identifier (engine name, event name, permission scope, or MCP type), the error now includes a suggestion based on Levenshtein distance matching. For example: `invalid engine: copiliot → Did you mean: copilot?`. No action required — this improves the compile-error experience automatically.

**File/line/column context in validation errors** — Validation errors now include `file:line:col:` positioning, enabling IDE tooling to jump directly to the problematic field. Useful when compiling large multi-import workflows where the offending field is not obvious from the error message alone.

**Inline sub-agents** — Sub-agent artifact staging and restoration are automatically emitted in compiled workflows. No feature flag needed.

**Shared `skip-if-match` dedup component** — The common "open issue/PR by title prefix" deduplication query is a shared compiler-imported component, eliminating copy-paste across workflows.

**Playwright `mode: cli`** — The compiler correctly accepts `mode: cli` in Playwright tool configuration.

**First-party coding-agent skill** — gh-aw ships a router skill that gives coding agents (Copilot, Claude, etc.) structured guidance on creating, debugging, and updating agentic workflows via the `gh aw` CLI.

**`allow-bot-authored-trigger-comment`** — For bots that don't follow the standard `[bot]` naming convention, opt into the confused-deputy bypass explicitly:

```yaml
on:
  issue_comment:
    types: [edited]
  allow-bot-authored-trigger-comment: true
```

Standard `[bot]`-authored comments are auto-detected and bypass the confused-deputy guard without this flag.

**`engine.env` handling** — Multi-line block-scalar `engine.env` values (written with `>-`) are supported. Custom job references in `engine.env` values (e.g., `${{ needs.my_job.outputs.value }}`) are wired into the agent job's `needs` list.

**MCP progress notifications** — The `logs`, `audit`, and `audit-diff` MCP tools stream real-time progress updates to AI clients during long-running operations.

**`report_incomplete` safe output** — Report incomplete work as a safe output when the agent runs out of turns or budget before finishing.

**`checks` MCP tool** — `checks` is available as a first-class MCP tool in the gh-aw MCP server, alongside `pull_requests`, `repos`, `issues`, etc.

**`checkout: false`** — Skip the default repository checkout when the workflow doesn't need source code (e.g., ChatOps commands that only call APIs via `web-fetch`). Saves ~10-30s of runner time.

**`checkout.clean-git-credentials`** — Remove cached git credentials from the workspace after checkout, preventing credential leaks when subsequent steps or build tools use submodules. Required for repositories where `persist-credentials: false` alone was insufficient (e.g., compiled lock files that use submodule checkout patterns):

```yaml
checkout:
  clean-git-credentials: true   # Scrub git credentials post-checkout; required for submodule-using workflows
```

> ⚠️ **Submodule credential leak (pre-v0.74.4):** Compiled lock files previously used `persist-credentials: false` on checkout steps, but this setting was not respected when submodules were present, allowing credentials to persist in the git config. `clean-git-credentials: true` resolves this by explicitly scrubbing credentials after checkout.

**`engine.max-turns`** — Limit the number of turns the agent can take. Set in the engine block: `engine: { id: claude, max-turns: 15 }`. Preserved through shared imports. **Supported on Claude only** — the Copilot engine uses `max-continuations` instead; other engines do not support a turn cap (see Engine feature comparison below).

**Available engines:** `copilot` (default), `claude`, `codex`, `gemini`, `crush` (experimental), `opencode` (experimental). See [Engines reference](https://github.github.com/gh-aw/reference/engines/).

**Engine feature comparison:**

| Feature | Copilot | Claude | Codex | Gemini | Crush | OpenCode |
|---------|---------|--------|-------|--------|-------|----------|
| `max-turns` | ✗ | ✓ | ✗ | ✗ | ✗ | ✗ |
| `max-continuations` | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ |
| `engine.agent` (custom agent) | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ |
| `engine.bare` (disable context) | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ |
| `engine.harness` (custom harness) | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ |
| Tools allowlist | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ |

**Available tools:** `web-fetch` (fetch URLs), `bash` (shell commands), `checks` (CI check runs), GitHub MCP toolsets (`pull_requests`, `repos`, `issues`, etc.). Use `tools: [web-fetch]` for workflows that call external APIs.

**MCP config location:** `.github/mcp.json` (previously `.mcp.json` at repo root). Migrate existing configs manually.

**`vulnerability-alerts` permission** — Available as a `GITHUB_TOKEN` permission scope for workflows that need to read security alerts.

### Safe-Outputs You May Not Know About

The official safe-outputs reference covers 30+ output types — the ones below are commonly missed even though they materially change workflow design:

- **`set-issue-type:` / `set-issue-field:`** — Set GitHub Issues type or any single field by name/value (default `max: 5`). Useful for triage workflows that classify issues without using labels.
- **`upload-artifact:`** — Upload files as run-scoped GitHub Actions artifacts (default `max: 1`, configured via `max-uploads:` not `max:`). Prefer over `upload-asset:` (orphan-branch storage) for most cases.
- **`dispatch_repository:`** *(experimental)* — Trigger `repository_dispatch` events in **external** repositories (cross-repo). Audit carefully: pairing this with `roles: all` lets untrusted triggers reach other repos.
- **Custom safe-output `jobs:` and `actions:`** — Register your own post-processing jobs as MCP tools (`safe-outputs.jobs:`) or mount any public GitHub Action as an agent-callable tool (`safe-outputs.actions:`). See [Custom Safe Outputs](https://github.github.com/gh-aw/reference/custom-safe-outputs/).

### Issue / Comment Lifecycle Options

- **`create-issue.group-by-day: true`** — Posts subsequent same-day runs as comments on the existing issue created earlier that UTC day, instead of creating duplicate issues. Pairs well with `close-older-issues: true` for daily/weekly report workflows.
- **`create-issue.deduplicate-by-title:`** — Drop duplicate issues by title match (`true` for exact, integer for Levenshtein edit distance). Eliminates the "agent re-creates the same triage issue every run" pattern.
- **`messages.append-only-comments: true`** — Disables the default behavior of editing the activation comment with final status; each run posts a fresh comment for an append-only timeline. Useful when audit-trail visibility matters more than UI tidiness.
- **`add-comment.discussions: false`** — Opts the workflow out of `discussions:write` permission. **Set this when the workflow only comments on issues/PRs** — otherwise the safe-outputs job carries an unnecessary write scope.
- **`add-comment.allowed-mentions:`** — Permit specific `@team` or `@user` mentions (others are escaped). The author of the parent issue/PR/discussion is auto-preserved.

### Auto-Injected `create-issue` Opt-Out

**🛑 Frequent surprise:** If you omit `safe-outputs:` entirely (or only declare system types `noop` / `missing-tool` / `missing-data`), gh-aw **silently auto-enables `create-issue`** with `max: 1`, the workflow ID as the label, and the workflow ID as the title prefix. The first time an agent run completes with content, an issue gets created. To opt out, declare an explicit `safe-outputs:` block with the outputs you actually want — even an empty block is not sufficient.

```yaml
# This workflow has NO safe outputs at all — must declare explicitly
safe-outputs:
  noop:
    report-as-issue: false   # Also suppress noop → comment behavior
```

### Observability (OTLP)

gh-aw supports OpenTelemetry trace export for workflow observability:

```yaml
observability:
  otlp:
    endpoint: ${{ secrets.OTLP_ENDPOINT }}
    headers:
      Authorization: "Bearer ${{ secrets.OTLP_TOKEN }}"
```

Traces include per-job spans with timing, token usage breakdowns, GitHub API rate-limit analytics, and Time Between Turns (TBT) metrics. Use `gh aw audit <run-id>` to view per-run analytics including TBT and API consumption charts. The `gh aw logs` command also surfaces rate-limit data.

### Security Hardening

**NFKC normalization + homoglyph detection** — SafeOutputs infrastructure detects Unicode homoglyph attacks (e.g., Cyrillic characters disguised as Latin) via NFKC normalization, preventing safe-output key spoofing.

**Token injection hardening** — Secrets are injected via `env:` blocks rather than inline `run:` interpolation, reducing exposure to shell injection.

> **v0.74.4+ auto-hoist `${{ … }}` from `run:` to `env:`** — The compiler now automatically rewrites `${{ … }}` expressions inside `run:` blocks (and `safe_jobs:` step env vars) into `env:` bindings as part of compile. Authors no longer need to manually rewrite expressions to clear the run-script guardrail; recompiling an existing workflow picks up the transform automatically.

### Breaking Changes & Migrations

Deprecated frontmatter fields are rejected by the compiler. Run `gh aw fix --write` to auto-fix supported patterns. Some migrations (e.g., `cli-proxy`, `.mcp.json`) require manual edits — see [`references/migrations.md`](references/migrations.md) for the full table and version-specific bug history.

Supported runtimes: `node`, `python`, `go`, `uv`, `bun`, `deno`, `ruby`, `java`, `dotnet`, `elixir`.

## Further Reading

For deep-dive details on execution model, security boundaries, fork handling, safe output types, and known issues, see [`references/architecture.md`](references/architecture.md).

See also the [official gh-aw documentation](https://gh.io/gh-aw) for:
- **Triggers** — complete trigger reference with activity types
- **Frontmatter** — all configuration options
- **Safe outputs** — complete list of 30+ types, key options for each
- **Integrity filtering** — content trust hierarchy and configuration
