---
name: "Agentic Workflow Auditor"
description: |
  Audits this repo's gh-aw agentic workflows (.github/workflows/*.md) against the
  gh-aw-guide skill: security anti-patterns, deprecated frontmatter, and lock-file
  freshness vs. the latest stable gh-aw release. Deterministic scanners run first;
  the agent only interprets results in context and emits a single digest.

on:
  # Manual-only by default so the workflow has NO autonomous side effects until you
  # opt in. Uncomment the schedule / pull_request triggers below to make it active.
  # schedule: weekly on monday
  # pull_request:
  #   types: [opened, synchronize, reopened]
  #   paths:
  #     - '.github/workflows/**.md'
  #     - '.github/workflows/**.lock.yml'
  workflow_dispatch:

# read-only agent; every write goes through the sandboxed safe-output job.
permissions:
  contents: read
  issues: read
  pull-requests: read
  actions: read

engine:
  id: copilot
  model: claude-sonnet-4.6

network:
  allowed:
    - defaults
    - github            # api.github.com for the latest-stable-release lookup

tools:
  github:
    toolsets: [default]
  bash: ["find", "ls", "cat", "grep", "head", "tail", "wc", "jq", "sed", "awk", "echo", "test", "basename"]

safe-outputs:
  # Schedule/dispatch mode: one rolling tracker issue.
  create-issue:
    max: 1
    title-prefix: "[workflow-audit] "
    labels: [workflow-audit, automated]
    close-older-issues: true
  # PR mode: a single review comment on the PR that touched a workflow.
  add-comment:
    max: 1
    target: "*"
    hide-older-comments: true
  noop:
    report-as-issue: false
  missing-tool:
    create-issue: false
  report-incomplete:
    create-issue: false
  report-failure-as-issue: false

concurrency:
  group: "workflow-auditor-${{ github.run_id }}"
  # When you enable the pull_request trigger above, scope by PR instead:
  # group: "workflow-auditor-${{ github.event.pull_request.number || github.run_id }}"
  cancel-in-progress: false

timeout-minutes: 20

steps:
  - name: Checkout repository
    uses: actions/checkout@v4
    with:
      persist-credentials: false

  - name: Run deterministic audits (security + deprecation + freshness)
    env:
      GH_TOKEN: ${{ github.token }}
    run: |
      set -euo pipefail
      OUT=/tmp/gh-aw/agent/audit
      mkdir -p "$OUT"

      # The gh-aw-guide skill ships the scanner + knowledge base. Support both the
      # top-level layout (skills-distribution repos) and the .github/skills layout.
      SKILL=""
      for cand in gh-aw-guide .github/skills/gh-aw-guide; do
        if [ -d "$cand" ]; then SKILL="$cand"; break; fi
      done
      echo "skill_dir=${SKILL:-<not found>}" > "$OUT/context.txt"

      # 1) Security scanner (from the gh-aw-guide skill).
      if [ -n "$SKILL" ] && [ -f "$SKILL/scripts/Test-GhAwWorkflowSecurity.ps1" ]; then
        pwsh "$SKILL/scripts/Test-GhAwWorkflowSecurity.ps1" \
          -WorkflowDir .github/workflows \
          > "$OUT/security.txt" 2>&1 || true
      else
        echo "gh-aw-guide scanner not found — skipping security scan" > "$OUT/security.txt"
      fi

      # 2) Deprecated-frontmatter grep (mirrors references/migrations.md).
      grep -rnE 'max-effective-tokens|max-daily-effective-tokens|network\.firewall|features\.(inline-agents|copilot-requests|user-invokable|disable-model-invocation)|engine\.max-turns|cli-proxy|disable-xpia-prompt|tools\.serena' \
        .github/workflows/*.md > "$OUT/deprecated.txt" 2>/dev/null || echo "none" > "$OUT/deprecated.txt"

      # 3) Lock freshness: compare each lock's gh-aw setup version to latest stable.
      LATEST=$(gh api repos/github/gh-aw/releases/latest --jq '.tag_name' 2>/dev/null || echo "unknown")
      echo "latest_stable=$LATEST" > "$OUT/freshness.txt"
      for lock in .github/workflows/*.lock.yml; do
        [ -f "$lock" ] || continue
        VER=$(sed -n '2p' "$lock" | sed 's/^# gh-aw-manifest: //' \
          | jq -r '[.actions[]? | select(.repo | test("gh-aw-actions/setup")) | .version][0] // "?"' 2>/dev/null \
          || echo "?")
        echo "$(basename "$lock"): compiled_with=${VER:-?}  latest=$LATEST" >> "$OUT/freshness.txt"
      done

      echo "=== context ===";    cat "$OUT/context.txt"
      echo "=== security ===";   cat "$OUT/security.txt"
      echo "=== deprecated ==="; cat "$OUT/deprecated.txt"
      echo "=== freshness ===";  cat "$OUT/freshness.txt"
---

# Agentic Workflow Auditor

You are auditing the gh-aw agentic workflows in **this** repository. Authoritative
knowledge lives in the **gh-aw-guide** skill — its directory is reported as
`skill_dir=` in `/tmp/gh-aw/agent/audit/context.txt`. Read that skill's `SKILL.md`
and `references/migrations.md` before judging anything.

## Inputs (already generated for you)

Reports are in `/tmp/gh-aw/agent/audit/`:

- `context.txt` — which skill directory was found.
- `security.txt` — output of `Test-GhAwWorkflowSecurity.ps1` (regex heuristics).
- `deprecated.txt` — grep hits for deprecated frontmatter fields.
- `freshness.txt` — each `*.lock.yml`'s compiled gh-aw version vs. the latest stable.

Read all four with `cat`.

## Your job

1. **Triage the security findings in context — do not parrot the scanner.** The
   scanner is heuristic and over-reports. In particular, modern gh-aw auto-injects
   `restore_base_github_folders.sh` after a PR-branch checkout, which restores
   `.github/` (workflow defs, skills, prompts) from the **base** branch. So a
   `pull_request_target` + checkout, or a pre-agent step that runs a script under
   `.github/`, is **mitigated** when (a) the job token is read-only, (b) writes go
   only through capped safe-outputs, and (c) the trigger is `roles:`-gated. Confirm
   each by reading the workflow's frontmatter and (if needed) its `.lock.yml`. Only
   escalate a finding if the mitigation is genuinely absent (e.g. a script executed
   from **outside** `.github/` after a fork checkout, or a write-capable token on an
   unrestricted trigger).

2. **Confirm deprecations** against `references/migrations.md` and give the exact
   `gh aw fix --write` codemod or manual replacement.

3. **Flag stale locks.** Any lock compiled more than one minor behind the latest
   stable should be recompiled (`gh aw compile`). Call out the security-relevant
   deltas it is missing (AWF firewall version, digest pinning, safe-output integrity
   fixes) using the skill's version history.

4. **Note auth/cost posture.** If workflows authenticate Copilot with a
   `COPILOT_GITHUB_TOKEN` PAT and this is an org-owned repo, mention the optional
   migration to `permissions: copilot-requests: write` (org billing, no PAT) and the
   caveat that user inference budgets don't apply to org billing — pair with
   `max-ai-credits`/cost centers.

## Output

- **On a pull request** (`github.event_name == 'pull_request'`): call `add-comment`
  once with a concise per-workflow verdict table for the workflow files changed in
  this PR. Lead with a one-line PASS/CHANGES-REQUESTED.
- **On schedule / dispatch**: call `create-issue` once with the full fleet digest
  grouped by severity. `close-older-issues: true` keeps a single rolling tracker.
- **If everything is clean and current**: call `noop` with a one-line reason. Do not
  open an issue or comment.

Keep it high signal. Every item must cite the skill rule or migration it comes from.
Never include secrets, tokens, or raw untrusted PR text in the output.
