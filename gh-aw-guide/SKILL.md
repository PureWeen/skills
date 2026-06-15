---
name: gh-aw-guide
description: >-
  Reviewer overlay for GitHub Agentic Workflows (gh-aw): security boundaries,
  fork handling, safe-outputs hardening, concurrency footguns, integrity-filter
  traps, and version-pinned bug history. Use when reviewing or editing
  .github/workflows/*.md or .lock.yml files. For canonical gh-aw schema and
  authoring scaffolds, see github/gh-aw upstream prompts (see preamble).
---

# gh-aw Reviewer Overlay

> **This skill is the primary gh-aw reference for this workspace.**
> For the canonical schema, see https://github.com/github/gh-aw/tree/main/.github/aw/ — these upstream prompts are usually **not installed locally** in consumer repos, so do not assume an `applyTo:` baseline is available. This guide focuses on **security boundaries, footguns, and review-critical patterns** that the schema docs don't emphasize. For migration help with older workflows, see [`references/migrations.md`](references/migrations.md). For deep-dive execution model, fork handling, and threat model, see [`references/architecture.md`](references/architecture.md).

## CLI Essentials

```bash
gh aw compile <name>          # Compile .md → .lock.yml (always commit both)
gh aw run <name> --ref main   # Trigger a workflow_dispatch run on a branch
gh aw trial ./<name>.md --clone-repo owner/repo  # Test before merging to main
gh aw lint                    # Validate .lock.yml without recompile
gh aw audit <run-id>          # Analyze a completed run
gh aw replay <run-id-or-url>  # Render a unified timeline from run artifacts/logs
gh aw logs --format markdown  # Cross-run audit/trend report
gh aw fix --write             # Apply supported source migrations
gh aw compile --approve       # Approve safe-update manifest changes (also on `run`, `upgrade`)
```

`.lock.yml` is auto-generated — **never edit manually**. On merge conflict, resolve in the source `.md`, accept either side for `.lock.yml`, then `gh aw compile` to regenerate. For deprecated flags, see [`references/migrations.md`](references/migrations.md). Full CLI reference: `gh aw --help`.

## 🚨 Before You Build: Prefer Built-in gh-aw Features

**CRITICAL RULE:** Before implementing any trigger, output, scheduling, or interaction mechanism in a gh-aw workflow, check whether gh-aw has a built-in feature. Manual reimplementations are always worse — they miss platform integration (emoji reactions, sanitized inputs, noise reduction) and accumulate bugs.

### Anti-Patterns: Manual Reimplementations to Avoid

> ⏱ **Staleness note (last reviewed: 2026-06-15 against gh-aw v0.79.8 — first **public-preview** release wave).** gh-aw ships new built-ins frequently. If you don't see what you need here, check the canonical [safe-outputs reference](https://github.github.com/gh-aw/reference/safe-outputs/), [triggers reference](https://github.github.com/gh-aw/reference/triggers/), [frontmatter reference](https://github.github.com/gh-aw/reference/frontmatter/), and [cost-management reference](https://github.github.com/gh-aw/reference/cost-management/) before reimplementing.

| If you're about to implement... | Use this built-in instead |
|---------------------------------|--------------------------|
| `issue_comment` + `startsWith(comment.body, '/cmd')` | `slash_command:` trigger |
| Manual emoji reaction on triggering comment | `reaction:` field under `on:` |
| Posting "workflow started/completed" status comments | `status-comment: true` under `on:` |
| Fixed cron schedule for non-critical timing | `schedule: weekly on monday around 9:00` (fuzzy) |
| Manual `if:` to skip bot-authored PRs | `skip-bots:` under `on:` |
| Manual `if:` to skip by author role | `skip-roles:` under `on:` |
| Manual label check + removal for one-shot commands | `label_command:` trigger |
| Editing old comments to collapse them | `hide-older-comments: true` on `add-comment:` |
| Creating no-op report issues | `noop: report-as-issue: false` |
| Auto-closing older issues from same workflow | `close-older-issues: true` on `create-issue:` |
| Disabling workflow after a date | `stop-after:` under `on:` |
| Manual approval gating | `manual-approval:` under `on:` |
| Search-based skip logic in `steps:` | `skip-if-match:` / `skip-if-no-match:` under `on:` |
| Locking issues to prevent concurrent edits | `lock-for-agent: true` under trigger |
| Manually hiding agent comments | `hide-comment:` safe output |
| Custom post-processing jobs for agent output | `safe-outputs.jobs:` (MCP tool access) |
| Wrapping GitHub Actions as agent-callable tools | `safe-outputs.actions:` action wrappers |
| Triggering CI on agent-created PRs | `github-token-for-extra-empty-commit:` on `create-pull-request` |
| No guard against agent approving PRs | `allowed-events: [COMMENT]` on `submit-pull-request-review`; or `[COMMENT, REQUEST_CHANGES]` with `supersede-older-reviews: true` |
| Stale blocking reviews from previous `/review` runs | `supersede-older-reviews: true` on `submit-pull-request-review` |
| Merging PRs via shell `gh pr merge` in post-steps | `merge-pull-request` safe output |
| Keeping PR branch up-to-date with base manually | `update-branch: true` on `update-pull-request` |
| Configuring the GitHub CLI proxy mode | `tools.github.mode: gh-proxy` (deprecated `cli-proxy` removed) |
| `slash_command:` without `events:` filter | `events: [pull_request_comment]` or `events: [issue_comment]` |
| `cancel-in-progress: true` on `slash_command:` workflows | `cancel-in-progress: false` |
| `pull_request` trigger for agentic workflows | `slash_command:`, `label_command:`, or `schedule` |

## Common Patterns

### Pre-Agent Data Prep (`steps:`)

Use `steps:` for GitHub API access that the agent needs to consume:

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

Comment bodies, issue titles, and PR descriptions are **user-controlled untrusted input**. In pre-agent `steps:`, always use `steps.<id>.outputs.text` (sanitized) instead of raw `${{ github.event.comment.body }}`. ❌ **Never reference `${{ github.event.* }}` content fields directly in agent prompts.** Container sandboxing limits the write surface but does **not** prevent prompt injection (XPIA) — pair sanitization with tight `safe-outputs:` as defense-in-depth.

> 🛑 **Recursive workflow triggering**: Actions via `GITHUB_TOKEN` do **NOT** fire new workflow events (prevents infinite loops). Actions via GitHub App installation tokens or PATs **DO** fire events. This is why `github-token-for-extra-empty-commit:` requires a PAT — `GITHUB_TOKEN` pushes won't trigger CI on agent-created PRs.

### Safe Outputs — `max:` Semantics

```yaml
safe-outputs:
  add-comment:
    max: 1                      # counts comments — 1 comment per run (blast-radius cap)
    hide-older-comments: true
    target: "*"    # Required for workflow_dispatch (no triggering PR context)
```

> **🚨 `max:` is type-specific — it does NOT uniformly mean "max tool calls".** Setting `max: 1` on `add-labels` thinking "one tool call per run" silently drops every label beyond the first (the agent batches multiple labels per call, but `max:` counts the total labels). Always check the unit before setting it.

> ⏱ **Defaults table (last verified 2026-06-15 against gh-aw v0.79.8 — see [safe-outputs reference](https://github.github.com/gh-aw/reference/safe-outputs/)):** Treat as non-authoritative — verify upstream for types not listed or when reviewing a workflow on an older gh-aw version.
>
> | Type | What `max:` counts | Default |
> |---|---|---|
> | `add-labels` / `remove-labels` | **labels** (sum across calls) | 3 |
> | `add-reviewer` | **reviewers** | 3 |
> | `hide-comment` | **comments** | 5 |
> | `add-comment` | comments (≈1 call) | 1 |
> | `create-pull-request-review-comment` / `reply-to-pull-request-review-comment` / `resolve-pull-request-review-thread` | inline review items | 10 |
> | `upload-asset` / `autofix-code-scanning-alert` | items | 10 |
> | `dispatch-workflow` | dispatches (calls) | 1 |
> | `set-issue-type` / `set-issue-field` | operations | 5 |
> | `update-project` / `close-pull-request` | operations | 10 |
> | `create-issue` / `update-issue` / `close-issue` | issues / updates / closures | 1 |
> | `create-pull-request` / `update-pull-request` / `push-to-pull-request-branch` | PRs / updates / pushes | 1 |
> | `submit-pull-request-review` | reviews | 1 |
> | `assign-*` / `unassign-*` / `link-sub-issue` | individual ops | 1 |
> | `create-discussion` / `update-discussion` / `close-discussion` | items | 1 |
> | `update-release` / `create-project` / `create-project-status-update` | items | 1 |
> | `upload-artifact` | uses **`max-uploads:`**, not `max:` | 1 |
> | `create-agent-session` / `call-workflow` / `noop` | sessions / calls / messages | 1 |
> | `create-code-scanning-alert` / `missing-tool` / `missing-data` | items | unlimited |
>
> **Sizing principle**: `max:` is a blast-radius cap, not a retry budget. The safe-outputs infrastructure handles HTTP 429/5xx retries independently. Raising `max:` doesn't help retries; **lowering it below legitimate item count silently drops items beyond the cap**. For multi-item types (`add-labels`, `add-reviewer`, all PR-review-comment types), set `max:` to the realistic per-run maximum — e.g., `add-labels: max: 1` on a labeler that emits `area-*` + `platform/*` will drop one of them every time.

### Add Labels — Security Hardening

`add-labels:` accepts `allowed:` (glob allow-list) and `blocked:` (glob deny-list) — infrastructure-level filters that run **before** the agent's chosen labels are applied. **Always set `allowed:`** when the workflow has `roles: all` or otherwise accepts untrusted triggers. Otherwise a prompt-injected agent can apply any label including ones that trigger downstream automation (`approved-for-merge`, `needs-backport`, `label_command:` triggers).

```yaml
safe-outputs:
  add-labels:
    max: 3                                # counts LABELS, not calls — see table above
    allowed: [area-*, platform/*, t/*]    # restrict to expected label families
    blocked: ["~*", "*[bot]"]             # deny patterns regardless of allowed
```

### Auto-Injected `create-issue` Opt-Out

**🛑 Frequent surprise:** If you omit `safe-outputs:` entirely (or only declare system types `noop` / `missing-tool` / `missing-data`), gh-aw **silently auto-enables `create-issue`** with `max: 1`, the workflow ID as the label, and the workflow ID as the title prefix. The first time an agent run completes with content, an issue gets created. To opt out, declare an explicit `safe-outputs:` block — an empty block is not sufficient:

```yaml
safe-outputs:
  noop:
    report-as-issue: false   # Also suppress noop → comment behavior
```

### Concurrency

Include all trigger-specific PR number sources. **Use `cancel-in-progress: false` for `slash_command:` and `label_command:` workflows** — a non-matching event (ordinary comment, benign label change) in the same group can cancel an in-progress matching run, killing the agent mid-execution:

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

> ⚠️ **`workflow_dispatch` concurrency footgun.** If your workflow accepts an `inputs.pr_number`, **include it in the concurrency group** — otherwise `github.ref` alone (typically `refs/heads/main` for every dispatch) puts all simultaneous PR-targeted dispatches into the same group, and `cancel-in-progress: true` silently cancels a maintainer's in-flight `/review` of PR #100 when another maintainer dispatches against PR #200 seconds later.

> ⚠️ **Pre-cancellation race**: Cancellation is asynchronous — `SIGTERM` then 7500ms then `SIGKILL`. Already-running steps may complete. An agent that already posted a comment cannot un-post it; a `create-pull-request` that already ran cannot un-create the PR. **Concurrency is not a substitute for idempotency.**

### `slash_command:` Event Subscription

`slash_command:` compiles to broad comment-event subscriptions by default. On busy repos this can mean hundreds of skipped pre-activation runs per day (runner cost + UI noise + concurrency collisions). **Always narrow `events:`** to the minimum needed:

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

- **`label_command:`** — One-shot command triggered by applying a label. Auto-removed after the workflow fires (self-resetting).
- **`names:` filtering** — Filter label events to specific label names for persistent state.
- **`remove_label: false`** — Keep the label after triggering (persistent state markers).

See the [LabelOps pattern guide](https://github.github.com/gh-aw/patterns/label-ops/) for examples.

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

Manual triggers should bypass the gate. Use step outputs rather than `exit 1` — failing the job normalizes failures and masks real errors.

### Documentation Maintenance Workflows

Upstream's stable documentation-maintenance workflows (`glossary-maintainer`, `daily-doc-updater`, `daily-doc-healer`, `unbloat-docs`) are useful review exemplars. They do not introduce new runtime semantics; use them as pattern guidance when reviewing glossary, documentation updater, or docs cleanup workflows:

| Pattern | Review check |
|---|---|
| Bounded scan scope | Scheduled docs/glossary agents should prefetch a bounded window, cache processed work, and cap detailed PR/commit reads. Avoid prompts that tell the agent to scan the whole repo every run. |
| Deterministic preflight | Use `steps:` or `pre-agent-steps:` to gather candidate commits, issues, or docs files into `/tmp/gh-aw/agent/*`, then instruct the agent to consume those files instead of re-querying blindly. |
| Documentation-only scope | Constrain candidate terms or edits to user-facing docs and code changes. Glossary workflows should ignore generic terms, internal-only implementation details, and issue discussion terminology unless corroborated by docs/commits/PRs. |
| Safe write path | Prefer `create-pull-request` with short `expires`, doc labels/title prefix, and `protected-files: fallback-to-issue` for broad docs updaters. Use `noop` when no doc change is warranted. |
| Least privilege | Keep frontmatter `permissions:` read-only and let safe-outputs own writes. Scope GitHub MCP `toolsets`, use `min-integrity: approved` when reading community content, and avoid global repository search tools unless the task truly needs them. |
| History-dependent scans | If the workflow relies on `git log --since` or weekly/daily commit windows, set `checkout.fetch-depth: 0`; shallow checkout makes the scan silently incomplete. |

### Fork PR Checkout (`workflow_dispatch`)

For `workflow_dispatch` workflows that need to evaluate a PR branch, the platform's `checkout_pr_branch.cjs` is **skipped** — you must implement checkout manually. Required checklist:

- [ ] Reject cross-repository (fork) PRs via `gh pr view --json isCrossRepository`
- [ ] Verify PR author has write/maintain/admin access (not just triage)
- [ ] Check out the PR branch via `gh pr checkout`
- [ ] Restore `.github/` and `.agents/` from the **base branch SHA** after checkout — defense-in-depth even though the platform also restores

Full reference script in [`references/architecture.md`](references/architecture.md#safe-pattern-checkout--restore).

For `pull_request` + fork support (not `workflow_dispatch`): add `forks: ["*"]` to the trigger frontmatter. The platform automatically preserves `.github/` and `.agents/` as a base-branch artifact in the activation job (gh-aw#23769, resolved).

### Operating Within a Fork

When you fork a repository, all workflow files come with it. Events inside your fork fire the workflows inside your fork, with your fork's secrets. This is separate from cross-fork PRs and is frequently a surprise.

**Guard pattern** — prevent workflows from running in forks (with a manual escape hatch):

```yaml
jobs:
  guard:
    if: ${{ github.event_name == 'workflow_dispatch' || !github.event.repository.fork }}
```

> ⚠️ **YAML gotcha**: Don't start a bare `if:` value with `!` — it's a YAML tag indicator. Always wrap in `${{ }}`.

> **Note:** This guard is for workflows running *inside* a forked repo — not for blocking cross-fork PRs. For fork PR protection, see Defense #5 under [Read-Only Contributor Write Surface](#read-only-contributor-write-surface).

### Security-Critical Patterns

These patterns are the most commonly missed when building secure workflows. Use all where applicable.

**1. Role-based access control** — `roles:` controls who can trigger the workflow. Without it, any user (including the PR author) can trigger `/review` on a malicious PR designed to prompt-inject the reviewer. The default `[admin, maintain, write]` is injected automatically for "unsafe" events (issues, comments, PRs, discussions):

```yaml
on:
  slash_command:
    name: review
    events: [pull_request_comment]
    roles: [admin, maintain, write]  # Only committers can trigger — NEVER use 'all' unless you've audited every safe-output
```

> ⚠️ **`triage` role footgun**: `triage` is excluded from the default allowlist. A `label_command:` workflow (which requires triage to apply the label) will _fire_ but the activation job will _deny_ a triage user unless `roles:` is broadened.

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

> **`supersede-older-reviews: true`** — When using `REQUEST_CHANGES`, set this to automatically dismiss older blocking reviews from the same workflow. Without it, a `REQUEST_CHANGES` review persists even after the author fixes everything and re-runs `/review`, because gh-aw has no `dismiss-pull-request-review` safe output. With `supersede-older-reviews`, the new review replaces the old one (best-effort).

**3. Integrity filtering** — controls what content the agent can **see** (vs. `roles:` which controls who can **trigger**). The MCP gateway intercepts GitHub tool calls and filters content by author trust level before the AI engine sees it. Filtered items are logged as `DIFC_FILTERED` events — inspect with `gh aw logs --filtered-integrity`.

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
    min-integrity: approved
    allowed-repos: "myorg/*"       # Scope to specific repos (optional; default "all")
    toolsets: [pull_requests, repos]
    trusted-users: [contractor-1]  # Elevate specific users to 'approved'
    blocked-users: [spam-bot]      # Unconditionally block (always denied)
    approval-labels: [human-reviewed]  # Labels that promote items to 'approved'
    # integrity-proxy: false       # Disables DIFC proxy for pre-agent gh CLI — use only for non-agentic steps
```

**Effective integrity computation order** (highest wins): `blocked-users` → `trusted-users` → `approval-labels` → endorsement/disapproval reactions → author association default.

**Centralized management:** `trusted-users`, `blocked-users`, and `approval-labels` accept GitHub Actions expressions (e.g., `${{ vars.TRUSTED_CONTRACTORS }}`). Organization-wide defaults can be distributed via Actions env vars prefixed `GH_AW_GITHUB_` (e.g., `GH_AW_GITHUB_TRUSTED_USERS`).

**Reaction-based integrity (`features.integrity-reactions: true`)** — Reactions on issues/PRs/comments can dynamically promote or demote integrity. The reactor's own integrity must meet `endorser-min-integrity` for their reaction to count, otherwise an unapproved user could promote themselves with a 👍:

```yaml
features:
  integrity-reactions: true
tools:
  github:
    min-integrity: approved                         # Baseline threshold for filtered content (paired with reactions)
    endorsement-reactions: [THUMBS_UP, HEART]       # Promote item to 'approved' (default when feature enabled)
    disapproval-reactions: [THUMBS_DOWN, CONFUSED]  # Demote item integrity (default when feature enabled)
    endorser-min-integrity: approved                # Reactor's own integrity required for reaction to count (default: approved)
    disapproval-integrity: none                     # Integrity level assigned on qualifying disapproval (default: none)
```

**`integrity-proxy: false`** — Disables the DIFC proxy for pre-agent `gh` CLI calls in workflow steps. Only use when you deliberately want to bypass integrity checks for a non-agentic step. Does NOT affect the MCP gateway filtering.

**Interaction with `roles:`:**

| `roles:` | `min-integrity` | Effect |
|----------|----------------|--------|
| Default `[admin, maintain, write]` | `approved` | **Most restrictive.** Only trusted actors trigger; agent sees only trusted content |
| Default | `unapproved`/`none` | Trusted actors only, agent reads community content. Good for post-merge scans |
| `all` | `approved` | **Two-layer defense.** Any actor triggers, but agent only sees trusted content |
| `all` | `none` | **Widest exposure.** Must pair with minimal `safe-outputs:` — only remaining constraint |

**4. CI triggering + protected file safety** — `GITHUB_TOKEN` pushes don't trigger CI; a PAT/App token is required. `protected-files` controls what happens when the agent modifies package manifests, top-level dot-directories, root governance docs, or `.github/`:

```yaml
safe-outputs:
  create-pull-request:
    github-token-for-extra-empty-commit: ${{ secrets.PAT_OR_APP_TOKEN }}  # Required to trigger CI
    protected-files: request_review      # Default: create PR + REQUEST_CHANGES review listing protected files
    # protected-files: blocked | fallback-to-issue | allowed (disables protection)
    # object form supports policy + exclude list
```

**5. Fork PR checkout for `workflow_dispatch`** — see the [Fork PR Checkout](#fork-pr-checkout-workflow_dispatch) pattern above. The platform's `checkout_pr_branch.cjs` is skipped for `workflow_dispatch`, so manual restoration of `.github/` from base is required.

**6. XPIA hardening** — Cross-prompt injection (XPIA) sanitization is enforced at compile time. `disable-xpia-prompt` is **rejected in strict mode** — do not use it. The runtime handles XPIA by default.

### Idempotency and the Edited-Comment Time-Bomb

**Slash command workflows MUST be idempotent.** Treat every activation as if the same command might already be running for the same target. Check before acting, claim a lock, no-op if already in progress or done.

gh-aw provides `lock-for-agent: true` to automatically lock/unlock the issue during execution, but use sparingly — it prevents genuine users from interacting on the issue/PR while the workflow runs. In public repos, locking is visible and may be perceived as moderation.

> 🛑 **The edited-comment time-bomb**: An attacker can edit a 6-month-old comment on a closed issue or PR, injecting `/command` or any payload — `issue_comment.edited` fires TODAY against today's secrets, today's `permissions:`, today's `safe-outputs:`. The workflow has no concept of "this comment was created when our security model was different." **For raw `issue_comment`, use `types: [created]`** — add `edited` only if you've explicitly designed for this attack vector.

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

### ✅ Recommended

| Trigger | Reviewer audit question | Key caveat |
|---------|------------------------|------------|
| `workflow_dispatch` | Does the workflow assume a specific ref? Branch selection is user-controlled — a write user can dispatch against a stale branch with weaker `permissions:`, different `safe-outputs:`, or a friendlier prompt | Write+ required |
| `schedule` | Is the cron isolated per workload? | Best concurrency story; no event spamming; no approval gate |
| `labeled` / `label_command:` | Can a triage user fire something that needs write to be safe? Verify label permissions before relaxing integrity filtering | Triage+ required; one-shot with auto-remove |
| `issues` | Are safe-outputs limited to read-reply (`add-comment`, `add-labels` with `allowed:`, `update-issue`, `close-issue`)? | `roles: all` acceptable **only** with read-reply outputs. **Never** pair `roles: all` with `dispatch-workflow`, `create-pull-request`, `push-to-pull-request-branch`, `create-agent-session`, `merge-pull-request`, or any output that creates persistent artifacts or triggers downstream pipelines |
| `release` / `milestone` | Trusted trigger; usually safe | Write+ required |

### ⚠️ Use with Caution

| Trigger | Headline risk |
|---------|--------------|
| `push` | **Always** use explicit `branches:` — bare `on: push` fires on every branch including bot/dependency/codeflow branches. Rapid pushes stack runs unless `cancel-in-progress: true` |
| `issue_comment` / `slash_command:` | Broad underlying subscription; concurrency catastrophe; edited-comment time-bomb |
| `pull_request_review` | Fires for ALL review types including COMMENT from any user, not just approvals |
| `discussion` / `discussion_comment` | Most-open untrusted-input surface; no approval gate; lower visibility than issues |

### `pull_request.synchronize` Gotchas

`synchronize` fires once per push to a PR branch (not per commit). Things that do **NOT** fire `synchronize`:

- **Draft → ready-for-review**: Fires `ready_for_review`, not `synchronize`. Default `types: [opened, synchronize, reopened]` won't re-run CI when a draft is marked ready
- **Base-ref edits**: Changing the PR's base branch fires `edited` (with `changes.base`), not `synchronize`
- **Pushes to the base branch**: Someone merging to `main` while your PR targets `main` does NOT fire `synchronize` on your PR — it fires `push` on `main`. Your CI won't re-run against the new base unless you push to your branch
- **Approval dismissal**: Branch protection's "Dismiss stale approvals on new commits" fires on the same head-SHA-changed event. A force-push that doesn't change file contents still invalidates all prior approvals

### ⛔ Avoid

| Trigger | Why |
|---------|-----|
| `pull_request` | Causes "Approve and run" gate for ALL workflows; clicking approves everything including `pull_request_target` with full secrets. Prefer `slash_command:`, `schedule`, or `label_command:` |
| `pull_request_target` | Runs on base ref with full secrets and write token — most exploited vulnerability class. Never check out PR head SHA |
| `workflow_run` | `pull_request_target`'s quieter sibling — launders untrusted fork artifacts into privileged context with no approval gate. Classic pwn: sandboxed `pull_request` workflow uploads artifacts (e.g., `coverage.json`), then `workflow_run` downloads and acts on them with full secrets. **Treat all downloaded artifacts as untrusted** |

## Design Principles

1. **Deterministic by default.** Use deterministic Actions and reusable workflows; agentic workflows only when the input is unstructured or AI unlocks a capability deterministic code cannot provide.
2. **Limitations ARE the security model.** Don't engineer bypasses (`pull_request_target` for write access, PAT pools to evade bot attribution, `workflow_run` to escape approval gates, `roles: all` to widen the actor pool). When a boundary blocks a legitimate goal, escalate to platform owners.
3. **Limit the agent job to agent-suitable work.** Keep filtering/skipping in pre-agent steps. Execute deterministic scripts before and after the agent job.
4. **Apply least privilege on every dimension.** Minimum `permissions:`, `safe-outputs:`, `network.allowed:`, secrets, `tools:`. The agent sandbox limits the write surface (prevents process escape) but does not neutralise prompt injection — untrusted input must still be treated as adversarial. The same operation in pre/post-agent steps runs on the runner host with full secret access.

## Engine & Runtime Updates

**Current engines:** `copilot` (default), `claude`, `codex`, `antigravity`, `gemini` (deprecated but still functional), `crush`, `opencode`, and `pi`. `engine: gemini` now emits a non-fatal deprecation warning that points authors to `engine: antigravity`; don't rewrite existing Gemini workflows blindly unless you've also updated secrets and runtime assumptions (`ANTIGRAVITY_API_KEY` vs `GEMINI_API_KEY`).

**Claude permission mode:** `engine.permission-mode` is first-class and can be `auto`, `acceptEdits`, `plan`, or `bypassPermissions`. gh-aw no longer derives `bypassPermissions` implicitly from unrestricted bash. In `bypassPermissions`, Claude's `--allowed-tools` is not an effective boundary; rely on MCP gateway `allowed:` filters and tight `tools:`.

```yaml
engine:
  id: claude
  permission-mode: auto
```

**Copilot inference via `github.token`:** The old `features.copilot-requests` flag migrated to an explicit permission. Use `gh aw fix --write` on old workflows, then verify the compiled lock file:

```yaml
permissions:
  contents: read
  copilot-requests: write
```

**AI Credits (AIC) guardrails:** `max-ai-credits` defaults to `1000` (1 AIC = $0.01 USD; pricing sourced from [models.dev](https://models.dev/)); steering messages are injected near 80/90/95/99% of budget. Use plain integers or `K`/`M` suffixes (e.g., `100M`); a negative value disables both enforcement and steering. `max-daily-ai-credits` adds an opt-in 24-hour per-workflow cap aggregated across recent runs; the guardrail is skipped for `workflow_call`, `repository_dispatch`, and `workflow_dispatch` runs carrying internal `aw_context` dispatch metadata. Threat-detection has its own `safe-outputs.threat-detection.max-ai-credits` budget (default `400`, override via `GH_AW_DEFAULT_DETECTION_MAX_AI_CREDITS`). The legacy `max-effective-tokens` / `max-daily-effective-tokens` fields are deprecated — run `gh aw fix --write` to apply the `effective-tokens-to-ai-credits` codemod (including the `-1` sentinel).

**Provider/auth knobs:** Copilot supports `engine.copilot-sdk: true` for SDK sidecar workflows. Claude supports Anthropic WIF through `engine.auth.provider: anthropic` plus `federation-rule-id`, `organization-id`, `service-account-id`, and `workspace-id`. BYOK Azure/OpenAI deployments can disable AWF model-name rewriting with `sandbox.agent.model-fallback: false`; API gateways that require nonstandard headers can set `sandbox.agent.targets.<provider>.authHeader`.

## Frontmatter Features (Selected)

**`on.needs:`** — Express dependencies on custom `pre_activation`/`activation` jobs, enabling GitHub App credentials to be sourced from upstream job outputs. See also `safe-outputs.needs` for credential-supply dependencies in the safe-outputs job.

**`checkout.clean-git-credentials`** — Remove cached git credentials from the workspace after checkout, preventing credential leaks when subsequent steps or build tools use submodules. Required for repositories where `persist-credentials: false` alone was insufficient (e.g., compiled lock files that use submodule checkout patterns):

```yaml
checkout:
  clean-git-credentials: true
```

> ⚠️ **Submodule credential leak (pre-v0.74.4):** Compiled lock files previously used `persist-credentials: false` on checkout steps, but this setting was not respected when submodules were present, allowing credentials to persist in git config. `clean-git-credentials: true` resolves this.

**`pre-steps:`** — Inject steps that run _before_ checkout and the agent, inside the same job. Recommended for token-minting actions (e.g., `actions/create-github-app-token`, `octo-sts`) for cross-repo checkout. The minted token stays in the same job, avoiding the masking issue when crossing job boundaries.

**`run-name:` / `runs-on:` / `runs-on-slim:`** — `runs-on` applies to the main agent job; `runs-on-slim` applies to generated framework jobs (`activation`, `safe_outputs`, unlock, etc.). `safe-outputs.runs-on` overrides `runs-on-slim` for safe-output jobs.

**`redirect:`** — Workflow authors can move/rename a published workflow and have `gh aw add`, `add-wizard`, and `update` follow the new location. Use `gh aw update --no-redirect` when auditing or when redirects should fail closed.

**`tracker-id:`** — Adds hidden `<!-- gh-aw-tracker-id: ... -->` markers to workflow-created issues, PRs, discussions, and comments so operators can search for all assets associated with a workflow.

**Project UTC offset:** `.github/workflows/aw.json` can set `{ "utc": "-08:00" }`; `GH_AW_DEFAULT_UTC` is the enterprise fallback. This affects rendered timestamps and expiration messages, not scheduling.

**`models:` (custom model pricing overlay):** Top-level frontmatter field that mirrors the built-in `models.json` structure and merges over it at runtime so `gh aw audit` / cost reporting see correct AIC values for private or non-catalog models. Imported overlays merge first; the main workflow's overlay wins on conflict at model granularity (whole model entry replaced; unmentioned models/providers preserved). Only the `providers` structure is accepted — the legacy alias-map form is rejected:

```yaml
models:
  providers:
    anthropic:
      models:
        my-custom-claude:
          cost:
            input: "3e-06"
            output: "1.5e-05"
            cache_read: "3e-07"
            cache_write: "3.75e-06"
```

**`safe-outputs.timeout-minutes:`** — Override the consolidated `safe_outputs` job timeout (default 45 minutes; was 15 minutes before this knob existed). Raise it for workflows running many sequential safe-output operations against large monorepos (e.g., several `push-to-pull-request-branch` cycles):

```yaml
safe-outputs:
  timeout-minutes: 120
  push-to-pull-request-branch: {}
```

**`features.dangerously-disable-sandbox-agent:` requires a literal justification string.** Boolean `true` and expressions like `${{ inputs.reason }}` are rejected by the compiler. The value must be a **static literal of at least 20 characters** explaining why the agent firewall is being removed; the string is stored for audit and diagnostics. Only use this in combination with `sandbox.agent: false` when the workflow genuinely runs in a controlled environment:

```yaml
features:
  dangerously-disable-sandbox-agent: "controlled environment with no internet access"
sandbox:
  agent: false
```

**`tools.github.allowed-repos: current`** — Use this in reusable workflows to scope the GitHub MCP guard policy to the repository where the workflow is running without hard-coding `owner/repo`.

**`network.allowed-input: true`** — Reusable `workflow_call` workflows can expose a `network_allowed` input so callers can union extra domains/ecosystems into the compiled baseline allowlist. Keep the source workflow's baseline least-privileged.

**Inline skills:** Workflow markdown can define engine-native skills inline using `## skill: \`name\`` blocks, parallel to inline sub-agents. The compiler extracts them out of the main prompt and materializes them into the engine-specific skill directory at runtime.

For exhaustive frontmatter reference (`source:`, `private:`, `resources:`, `labels:`, `runtimes:`, `imports:`, `engine.*`, etc.), see [github/gh-aw frontmatter docs](https://github.github.com/gh-aw/reference/frontmatter/).

## OpenTelemetry / OTLP Observability

Use `observability.otlp` instead of hand-rolled telemetry scripts when a workflow needs distributed traces:

```yaml
observability:
  otlp:
    endpoint:
      - url: ${{ secrets.OTLP_ENDPOINT_PRIMARY }}
        headers:
          Authorization: ${{ secrets.OTLP_TOKEN_PRIMARY }}
      - url: ${{ secrets.OTLP_ENDPOINT_BACKUP }}
    if-missing: warn
    attributes:
      langfuse.session.id: ${{ github.run_id }}
      langfuse.user.id: ${{ github.actor }}
```

**Review checks:**
- `endpoint` accepts a string, a single `{url, headers}` object, or an array of endpoint objects for fan-out. Top-level `headers:` only applies to string endpoints; object/array endpoints should carry per-endpoint headers.
- `if-missing` can be `error` (default), `warn`, or `ignore`. It controls OTLP-dependent MCP gateway setup; it is not a general "skip all telemetry" switch.
- Static endpoint hosts are automatically added to the network allowlist. Secret/expression endpoints are not statically resolvable, so ensure the collector host is otherwise allowed.
- `attributes:` adds custom span attributes to gh-aw job spans. Non-empty values are masked from runner logs; still avoid putting secrets or high-cardinality/raw user payloads in attributes.
- OTLP headers are passed via `OTEL_EXPORTER_OTLP_HEADERS` for the primary endpoint; multi-endpoint configurations also use `GH_AW_OTLP_ALL_HEADERS` so all endpoint headers are covered by masking/header handling. The MCP gateway config only receives endpoint/trace IDs. Do **not** embed collector credentials in custom gateway JSON, shell traces, or checked-in imports.
- Custom imports can emit spans with `/tmp/gh-aw/actions/otlp.cjs`; export failures are non-fatal and mirrored to `/tmp/gh-aw/otel.jsonl`. Treat `otel.jsonl` / `copilot-otel.jsonl` artifacts as observability data, not as content to paste into public docs.

## Safe Outputs You May Not Know About

The official safe-outputs reference covers 30+ output types — the ones below are commonly missed even though they materially change workflow design:

- **`create-check-run`** — Creates a first-class GitHub Check Run that appears in the PR Checks UI. `name:` is configured in frontmatter (not accepted from the agent); when `name` equals the workflow name it auto-suffixes with `(Result)` to avoid being collapsed into the workflow's own check entry. **`target:`** controls SHA resolution: `triggering` (default — resolves the current PR head via the Pulls API so the check follows force-pushes), `"*"` (the agent must include `pull_request_number` / `pr_number` / `pr` / `pull_number` per call), or an explicit PR number expression like `"${{ github.event.inputs.pr }}"`. Same-repo only.
- **`set-issue-type:` / `set-issue-field:`** — Set GitHub Issues type or any single field by name/value (default `max: 5`). Useful for triage workflows that classify issues without using labels.
- **`upload-artifact:`** — Upload files as run-scoped GitHub Actions artifacts (configured via `max-uploads:` not `max:`). Prefer over `upload-asset:` for most cases.
- **`dispatch_repository:`** *(experimental)* — Trigger `repository_dispatch` in **external** repositories. **Audit carefully:** pairing with `roles: all` lets untrusted triggers reach other repos.
- **Custom safe-output `jobs:` and `actions:`** — Register post-processing jobs as MCP tools (`safe-outputs.jobs:`) or mount any public GitHub Action as an agent-callable tool (`safe-outputs.actions:`).
- **`add-comment.discussions: false`** — Opts the workflow out of `discussions:write` permission. **Set this when the workflow only comments on issues/PRs** — otherwise the safe-outputs job carries an unnecessary write scope.
- **`add-comment.allowed-mentions:`** — Permit specific `@team` or `@user` mentions (others are escaped). The author of the parent issue/PR/discussion is auto-preserved.
- **`create-pull-request` / `push-to-pull-request-branch` protected-file policies:** Default is `request_review` (PR is preserved, plus a `REQUEST_CHANGES` review). Use `blocked` to fail closed, `fallback-to-issue` to avoid pushing and create a review issue, or object form `{ policy, exclude }` when a workflow intentionally edits a specific protected file.
- **`allowed-files:`** — Exclusive allowlist for code-push safe outputs. It is not an "extra allow" list; files outside the list are refused even if normally unprotected. To modify protected files, the path must match `allowed-files` and `protected-files` must allow it.
- **`allow-workflows: true`** — Required when a code-push safe output is allowed to modify `.github/workflows/**`; requires `safe-outputs.github-app` because `workflows: write` is GitHub App-only.
- **`push-to-pull-request-branch` cross-repo:** `target-repo` requires checking out the target repository with a distinct `path:`. The safe-output handler now respects `target-repo` across handlers; don't assume root workspace is the target repo.

### Issue / Comment Lifecycle Options (often missed)

- **`create-issue.group-by-day: true`** — Posts subsequent same-day runs as comments on the existing issue created earlier that UTC day, instead of creating duplicate issues. Pairs well with `close-older-issues: true` for daily/weekly report workflows.
- **`create-issue.deduplicate-by-title:`** — Drop duplicate issues by title match (`true` for exact, integer for Levenshtein edit distance). Eliminates the "agent re-creates the same triage issue every run" pattern.
- **`messages.append-only-comments: true`** — Disables the default behavior of editing the activation comment with final status; each run posts a fresh comment for an append-only timeline. Useful when audit-trail visibility matters more than UI tidiness.

## Security Hardening

**Token injection hardening** — Secrets are injected via `env:` blocks rather than inline `run:` interpolation. **The compiler (v0.74.4+) now automatically rewrites `${{ … }}` expressions inside `run:` blocks _and_ `safe_jobs:` step env vars** into `env:` bindings as part of compile — authors no longer need to manually rewrite expressions to clear the run-script guardrail; recompiling picks up the transform automatically. Older compiled lock files retain the manual form.

**NFKC normalization + homoglyph detection** — SafeOutputs detects Unicode homoglyph attacks (e.g., Cyrillic characters disguised as Latin) via NFKC normalization and homoglyph character mapping, preventing safe-output key spoofing.

**Safe-output file-reference hardening** — Safe-output MCP tool calls reject `@filepath` local-file references. Agents must pass explicit structured arguments or uploaded artifact/asset references; don't rely on local runner paths surviving into safe-output execution.

**Validation ergonomics:** Compiler/validation errors include `file:line:col` context and fuzzy "Did you mean?" suggestions for common typos in engine names, events, permission scopes, and MCP tool types.

## Breaking Changes & Migrations

Deprecated frontmatter fields are rejected by the compiler. Run `gh aw fix --write` to auto-fix supported patterns. Some migrations (e.g., `cli-proxy`, `.mcp.json`) require manual edits — see [`references/migrations.md`](references/migrations.md) for the full table and version-pinned bug history.

Supported runtimes: `node`, `python`, `go`, `uv`, `bun`, `deno`, `ruby`, `java`, `dotnet`, `elixir`.

## Further Reading

- **Execution model, fork handling, threat model, troubleshooting** — [`references/architecture.md`](references/architecture.md)
- **Version-pinned bug history and migration commands** — [`references/migrations.md`](references/migrations.md)
- **Unreleased/main-branch watchlist** — [`references/watchlist.md`](references/watchlist.md) (not stable behavior)
- **Official OpenTelemetry reference** — [`gh-aw OpenTelemetry docs`](https://github.github.com/gh-aw/reference/open-telemetry/)
- **Canonical schema reference** — [`github/gh-aw/.github/aw/`](https://github.com/github/gh-aw/tree/main/.github/aw/) (usually not installed locally — fetch directly when needed)
- **Official user-facing docs** — [`gh.io/gh-aw`](https://gh.io/gh-aw)
