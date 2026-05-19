# skills

A collection of reusable agent skills for software-engineering tasks.

Each top-level directory is a self-contained skill — drop it into the right place for your runtime:

- **Copilot CLI** — copy into `~/.agents/skills/<name>/` (user-scoped) or `<repo>/.agents/skills/<name>/` (repository-scoped)
- **Claude Code** — copy into `~/.claude/skills/<name>/` or `<repo>/.claude/skills/<name>/`

All skills follow the standard layout:

```
<skill-name>/
├── SKILL.md           # Front-matter (name, description) + instructions
├── references/        # Optional: deeper docs the agent loads on demand
└── scripts/           # Optional: helper scripts the skill invokes
```

## Skills

### [`gh-aw-guide`](./gh-aw-guide/)

Comprehensive reference for building and maintaining [GitHub Agentic Workflows (`gh-aw`)](https://github.com/githubnext/gh-aw). Covers architecture, security boundaries, fork handling, safe outputs, anti-patterns, compilation, and troubleshooting. Targets the latest stable `gh-aw` release; migration tips for older workflows live in [`references/migrations.md`](./gh-aw-guide/references/migrations.md).

Use it when creating or editing `.github/workflows/*.md` files, writing safe-outputs configs, configuring fork PR handling, setting up integrity filtering, or debugging "why doesn't my workflow trigger".

### [`gh-aw-guide-scraper`](./gh-aw-guide-scraper/)

Scrapes the upstream `github/gh-aw` repo (releases, docs, commits) to detect what's changed since the `gh-aw-guide` skill was last updated, then helps refresh the guide. Only reports **stable, non-prerelease** releases — pre-release features should not be added to the guide until they ship in a stable release.

Use it when asked to "update gh-aw guide", "check for gh-aw changes", or "sync gh-aw knowledge".

### [`generic-adversarial-pr-reviewer`](./generic-adversarial-pr-reviewer/)

Multi-model adversarial PR code review. Dispatches three independent reviewers (different models) in parallel, runs cross-validation on disputed findings, and produces a single high-signal review. Works with any GitHub repository — no project-specific knowledge required.

Use it when asked to "review a PR", "code review", or "expert review".

## License

[MIT](./LICENSE) — see each skill's `SKILL.md` front-matter for its own attribution.
