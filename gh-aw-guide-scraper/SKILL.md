---
name: gh-aw-guide-scraper
description: >-
  Scrape upstream gh-aw documentation, releases, and repo commits to detect what's changed
  and update the gh-aw-guide skill. Use when asked to "update gh-aw guide", "scrape gh-aw docs",
  "check for gh-aw changes", "refresh gh-aw skill", or "sync gh-aw knowledge". Runs
  Check-Staleness.ps1 (checks tracked sources for drift) and Scan-GhAwUpdates.ps1 (mines
  the github/gh-aw repo for new commits). Only reports stable (non-prerelease) releases.
  Produces a combined report, then updates the gh-aw-guide SKILL.md with the findings.
---

# gh-aw Guide Scraper Skill

Scrape upstream `github/gh-aw` documentation, releases, and repo commits to detect what's changed, then update the `gh-aw-guide` skill files with new knowledge.

**IMPORTANT:** This scraper only tracks **stable releases** (non-prerelease, non-draft). Pre-release features must NOT be added to the guide until they ship in a stable release.

## Prerequisites

- `gh` CLI authenticated with access to `github/gh-aw` (internal repo)
- `pwsh` (PowerShell 7+) installed
- The sync manifest at `.github/instructions/gh-aw-workflows.sync.yaml` must exist

## When to Use

- Before editing the gh-aw-guide skill, to ensure it's current
- When a new gh-aw stable release ships
- When asked to update or refresh gh-aw documentation
- Periodically to catch drift

## How to Run

### Step 1: Check for drift against tracked sources

```bash
pwsh ~/.agents/skills/gh-aw-guide-scraper/scripts/Check-Staleness.ps1
```

**What it checks:**
- **Doc pages** — fetches each tracked URL and computes a content hash
- **GitHub issues** — checks open/closed state against `resolution_expected`
- **Releases** — finds **stable** releases newer than `last_reviewed_release` (pre-releases and drafts are excluded)
- **Index crawling** — discovers new doc pages not yet tracked in the manifest
- **Recently closed issues** — finds closed issues (last 90 days) not in the manifest

**Output:** JSON report to stdout with `changes_detected: true/false`.

### Step 2: Extract every feature from release notes (CRITICAL)

For each new stable release found in Step 1, fetch the **full** release notes:

```bash
gh api "repos/github/gh-aw/releases/tags/vX.Y.Z" --jq '.body'
```

**This is the authoritative source — NOT the commit scanner.** Read every bullet under:
- "✨ What's New" — each bullet is a feature to document
- "🐛 Bug Fixes & Improvements" — each bullet may be a fix worth documenting
- "Breaking Changes" — each must be documented

**For every bullet, check:** Is this already in `~/.agents/skills/gh-aw-guide/SKILL.md`? If not, add it.

> ⚠️ **Do NOT skip this step or rely on the commit scanner alone.** The commit scanner (Step 3) uses keyword matching that misses entire feature categories (lint, firewall, billing, MCP, skills, progress notifications, etc.). Release notes are curated by the release author and contain features not discoverable from commit messages.

### Step 3: Run the commit scanner (supplementary)

The commit scanner is supplementary — it catches changes between releases that may not be in release notes yet:

```bash
pwsh ~/.agents/skills/gh-aw-guide-scraper/scripts/Scan-GhAwUpdates.ps1 -DryRun
```

Remove `-DryRun` to update the watermark file after a successful scan.

> **Known limitations of the commit scanner:**
> - Only matches 6 keyword categories: safe-outputs, triggers, compiler, security, breaking, engine
> - Misses: lint, firewall, billing, MCP (without "engine"), skills, progress, notifications, permissions, templates
> - Only reads commit messages, not release notes
> - Scans unreleased `main` commits — **never document a feature found only in the commit scanner until it appears in a stable release's notes**
> - Cannot cross-check what's already in the guide

### Step 4: Update the gh-aw-guide skill

Using the release notes from Step 2, update:

| File | What to update |
|------|---------------|
| `~/.agents/skills/gh-aw-guide/SKILL.md` | Version baseline, features, anti-patterns, safe-output types, trigger types |
| `~/.agents/skills/gh-aw-guide/references/architecture.md` | Execution model, security boundaries, new platform capabilities |
| `.github/instructions/gh-aw-workflows.sync.yaml` | Add new tracked URLs/issues, update `last_reviewed_release` |

**Update rules:**
- **Never include pre-release features** — only document what's in a stable release
- **Never remove divergence sections** (marked in the sync manifest)
- **Do NOT bump a version baseline** — the guide says "Assumes latest stable", not a specific version number
- **Do NOT add version tags to features** — the guide documents current behavior, not version history. New features are added without `(vX.Y.Z+)` annotations. Version-specific details go in `references/migrations.md` only.
- **Update `last_reviewed_release`** in the sync manifest

### Step 5: Verify

Re-run Step 1 to confirm drift is resolved:

```bash
pwsh ~/.agents/skills/gh-aw-guide-scraper/scripts/Check-Staleness.ps1
```

## Stable Release Identification

The GitHub Releases API distinguishes stable from pre-release. To check:

```bash
# Latest stable release (what users get by default)
gh api repos/github/gh-aw/releases/latest --jq '.tag_name'

# All stable releases
gh api "repos/github/gh-aw/releases?per_page=20" --jq '[.[] | select(.prerelease == false and .draft == false) | .tag_name]'
```

`gh extension install github/gh-aw` installs the **latest stable** release, not pre-releases.
