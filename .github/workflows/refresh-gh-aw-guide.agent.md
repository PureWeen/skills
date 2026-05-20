---
name: "Refresh gh-aw-guide"
description: "Nightly check for upstream gh-aw changes. If drift is detected, applies updates to the gh-aw-guide skill and opens a draft PR."

on:
  schedule: "daily"
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: read

# The scraper PowerShell scripts need an authenticated gh CLI. They run as
# regular workflow steps (outside the agent sandbox) and write JSON results
# to disk for the agent to read. This avoids granting the agent direct API
# access for the scraping phase.
steps:
  - name: Run staleness check
    id: staleness
    env:
      GH_TOKEN: ${{ github.token }}
    run: |
      mkdir -p /tmp/drift-results
      pwsh gh-aw-guide-scraper/scripts/Check-Staleness.ps1 \
        -ManifestPath gh-aw-guide/sync.yaml \
        > /tmp/drift-results/staleness.json 2>/tmp/drift-results/staleness-errors.log \
        || echo '{"changes_detected":false,"error":"script failed"}' > /tmp/drift-results/staleness.json
      CHANGES=$(python3 -c "import json; print(str(json.load(open('/tmp/drift-results/staleness.json')).get('changes_detected', False)).lower())" 2>/dev/null || echo "false")
      echo "changes_detected=$CHANGES" >> "$GITHUB_OUTPUT"
      echo "Staleness check complete. changes_detected=$CHANGES"
      head -c 2000 /tmp/drift-results/staleness.json || true

  - name: Run upstream commit scan (if stale)
    if: steps.staleness.outputs.changes_detected == 'true'
    env:
      GH_TOKEN: ${{ github.token }}
    run: |
      pwsh gh-aw-guide-scraper/scripts/Scan-GhAwUpdates.ps1 \
        -MaxCommits 50 \
        -WatermarkFile /tmp/drift-results/gh-aw-watermark.json \
        -DryRun \
        > /tmp/drift-results/upstream.json 2>/tmp/drift-results/upstream-errors.log \
        || echo '{"changes_detected":false,"error":"script failed"}' > /tmp/drift-results/upstream.json
      echo "Upstream scan complete."
      head -c 2000 /tmp/drift-results/upstream.json || true

engine:
  id: copilot
  model: claude-sonnet-4.6

network:
  allowed:
    - defaults

safe-outputs:
  create-pull-request:
    title-prefix: "[gh-aw-guide-sync] "
    labels: [automated, gh-aw-guide-sync]
    draft: true
    expires: 14
    protected-files: allowed
  noop:
    max: 1

concurrency:
  group: "refresh-gh-aw-guide"
  cancel-in-progress: false

timeout-minutes: 30
---

# Refresh the gh-aw-guide skill

Detect drift between this repo's `gh-aw-guide` skill and the upstream
`github/gh-aw` project. When stable-release drift is found, apply updates
to the skill files and open a draft PR for review.

> **🚨 No test or placeholder content.** Never call a safe-output tool with
> filler text. If there's nothing actionable to do, call `noop` and stop.

## Step 1 — Read the pre-computed scraper results

The scraper already ran in `steps:` (before you started) with an authenticated
`gh` CLI. **Do not try to re-run `Check-Staleness.ps1` yourself** — `gh` is
not authenticated inside the agent sandbox.

Inspect the JSON results:

```bash
python3 -c "
import json
d = json.load(open('/tmp/drift-results/staleness.json'))
print('changes_detected:', d.get('changes_detected'))
for m in d.get('manifests', []):
    for s in m.get('sources', []):
        if s.get('type') == 'releases':
            print(f'  releases: last_reviewed={s.get(\"last_reviewed_release\",\"\")} new={len(s.get(\"result\",{}).get(\"releases\",[]))}')
"
```

Then extract any new release notes:

```bash
python3 -c "
import json
d = json.load(open('/tmp/drift-results/staleness.json'))
for m in d.get('manifests', []):
    for s in m.get('sources', []):
        if s.get('type') == 'releases' and s.get('result', {}).get('releases'):
            for r in s['result']['releases']:
                print(f'=== {r[\"tag\"]} ({r.get(\"published_at\",\"\")}) ===')
                print(r.get('release_notes','')[:8000])
                print()
"
```

If `/tmp/drift-results/upstream.json` exists, also inspect it for supplementary
commit signals (only as a tie-breaker — release notes are authoritative).

If anything looks broken, check the error logs:

```bash
cat /tmp/drift-results/staleness-errors.log
cat /tmp/drift-results/upstream-errors.log 2>/dev/null || true
```

## 🚨 Critical — what to do next

### If `changes_detected` is `false`

Call the `noop` tool immediately:

```
noop(message="No drift detected against upstream gh-aw.")
```

Do **not** create a PR. Stop after calling `noop`.

### If `changes_detected` is `true`, continue below

---

## Step 2 — Build a checklist from release notes

Read **every** release note above and build a complete change list.

> **🔍 Scan ALL sections of each release note — not just the curated highlights.**
> Release notes have multiple sections, and author-visible items routinely live in the long-tail PR list:
> - **`✨ What's New`** — curated headline features (always scan)
> - **`🐛 Bug Fixes & Improvements`** — curated bug fixes (always scan)
> - **`Breaking Changes`** — must always be documented
> - **`## What's Changed`** (the full PR list at the bottom) — **also scan this**. PR titles like "fix: surface X guidance", "Add Y codemod", "Preserve Z in W filtering", or "Make E### error actionable" frequently describe author-visible behavior that the curated sections summarize away or omit entirely. Author-visible heuristic: does the PR title reference a config option, an error code, a trigger, a safe-output, or a compile-time transformation? If yes, evaluate it as a candidate P2 item.
> - **`### Community Contributions`** — issue links here describe the user-reported problem each PR resolves. Use them to understand the user-facing impact when a PR title is terse.

For each item, classify:

- **P0** — factually wrong in the current guide (must fix)
- **P1** — security-relevant change (must fix)
- **P2** — new feature, new safe output, new config option, new trigger
  behavior, or workflow-author-visible bug fix (must implement)
- **P3** — internal/cosmetic only (OTLP traces, CI pipeline changes, internal
  refactors, linters, dev tooling). Skip these, but list them in the PR body.

You must implement every P0/P1/P2 item.

## Step 3 — Read the current skill files

Read all of these so you understand what the guide already covers:

```bash
cat gh-aw-guide/SKILL.md
cat gh-aw-guide/references/architecture.md
cat gh-aw-guide/references/migrations.md
cat gh-aw-guide/sync.yaml
```

## Step 4 — Apply updates

For each P0/P1/P2 item:

1. Find the right section in the skill files.
2. Add or update the content (prefer in-place edits to additions where the
   section already exists).
3. Tick the item off your checklist.

Where to put each kind of change:

- **New safe output** → anti-patterns table + Safe Outputs section in `SKILL.md`
- **New frontmatter option** → Frontmatter Features section in `SKILL.md`
- **New trigger behavior** → Trigger Selection Guide in `SKILL.md`
- **Security change** → Security-Critical Patterns in `SKILL.md`
- **Protected files / integrity change** → `references/architecture.md`
- **Migration / breaking change** → `references/migrations.md` (this is the
  only place version numbers should appear)
- **Workflow-author bug fix** → relevant section + Known Issues if applicable

> **🛑 `references/migrations.md` section-header rule (DO NOT VIOLATE).**
> The `## Version-Specific Bug History` block contains one `### Fixed in vX.Y.Z` heading per release. Each heading is a **permanent** record of bugs shipped in that specific version. When adding new fixes for release vN.N.N:
>
> - **ALWAYS add a new `### Fixed in vN.N.N` heading above the existing `### Fixed in v(N-1).x.x` heading.**
> - **NEVER rename, edit, or merge into an existing `### Fixed in …` heading** — doing so retroactively misattributes the prior release's fixes to the new version, corrupting version history.
> - The same rule applies to any other section-by-release table in the guide (e.g., closed-issues tables).

## Step 5 — Update the sync manifest

In `gh-aw-guide/sync.yaml`, update `last_reviewed_release` to the newest stable
tag you incorporated. Add any new tracked URLs or issues that surfaced during
the scrape.

## Step 6 — Verify completeness and verify facts

Re-read your checklist. Every P0/P1/P2 item must be addressed.

> **🧪 Verify factual claims against upstream sources BEFORE finalizing.**
> Release notes describe behavior changes but rarely state exact default values, full option lists, or precise config syntax. When the changes you wrote include any of the following, cross-check against the upstream reference docs (use the web-fetch tool — `gh` is not authenticated inside the agent sandbox):
>
> | Claim type | Cross-check against |
> |---|---|
> | Numeric defaults (e.g., `max:`, `max-uploads:`, `retention-days:`) | `https://raw.githubusercontent.com/github/gh-aw/main/docs/src/content/docs/reference/safe-outputs.md` |
> | New config option names + accepted values | The corresponding `docs/src/content/docs/reference/*.md` file |
> | Version-attribution bullets in `migrations.md` | The actual release notes for that exact tag (`gh api repos/github/gh-aw/releases/tags/vX.Y.Z --jq '.body'` — already in the staleness JSON) |
> | "Available engines: …" list, safe-output enumeration, trigger list | Reference docs |
>
> If any default value or option name you wrote cannot be confirmed from upstream sources, **do not include the unverified claim** — describe the behavior change without the specific number and add a `<!-- TODO: verify default -->` HTML comment so the reviewer knows.

Rules:

1. **Respect `divergence:` sections** in `sync.yaml` — never remove the
   "Security Boundaries", "Safe Pattern: Checkout + Restore", or
   "Common Patterns" sections.
2. **Only document stable-release features.** Anything still in
   pre-release or only visible in the commit scanner stays out of the guide.
3. **Don't pin version numbers in `SKILL.md`.** The guide says "Assumes latest
   stable". Version-specific details go in `references/migrations.md` only.
4. **Include YAML examples** for new config options.

## Step 7 — Open the PR

Use the `create-pull-request` safe output with:

- Title: `Refresh gh-aw-guide for <release-tag(s)>` (e.g.,
  "Refresh gh-aw-guide for v0.73.0").
- Body containing:
  1. The release tag(s) covered.
  2. The complete list of P0/P1/P2 items implemented, each with which file
     and section was updated.
  3. The complete list of P3 items skipped with reasoning.
  4. Links to the upstream release notes you incorporated.

The PR is opened as a **draft** — a human reviewer can promote it.
