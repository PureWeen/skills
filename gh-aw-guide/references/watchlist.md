# gh-aw Watchlist (Not Stable Guidance)

This file tracks upstream main-branch and experimental signals that should not be treated as current gh-aw behavior until a stable release note ships them. Stable guidance remains in `SKILL.md`, `architecture.md`, and `migrations.md`.

Last checked: 2026-06-25. Stable baseline: v0.80.9 (gh-aw is in [public preview](https://github.blog/changelog/2026-06-11-github-agentic-workflows-is-now-in-public-preview/) — drop any lingering "research preview" framing from reviewed workflows).

## Items resolved into v0.80.x stable (shipped in v0.80.9)

These previously sat on this watchlist or surfaced in the v0.80.x prerelease line and are now stable in v0.80.9; they are documented in `SKILL.md` / `migrations.md`:

- **`safe-outputs.mentions.allow-team-members` → `allowed-collaborators` rename** (old name kept as a deprecated alias) with codemod `add-allow-team-members-collaborators`, plus new `allowed-teams` team-authorization (needs `read:org` token scope). (v0.80.7 BREAKING)
- **`max-patch-size` default raised 1 MB → 4 MB (4096 KB).** (v0.80.0)
- **Default AWF firewall bumped to v0.27.7** (was 0.27.2). (v0.80.6)
- **`max-turn-cache-misses` runaway-loop guardrail** (maps to `apiProxy.maxCacheMisses`). (v0.80.7)
- **`normalize-closing-keywords`** on `create-issue` / `add-comment` / `create-pull-request`. (v0.80.1)
- **`required-category`** in `safe-outputs.create-discussion` (close only within a category). (v0.80.1)
- **OTel `cicd.automation.*` semantic conventions + W3C `TRACEPARENT` propagation** through engine steps, plus configurable OTLP resource attributes. (v0.80.0–v0.80.1)
- **Dedicated billing reference page** at `https://github.github.com/gh-aw/reference/billing/`. (v0.80.3)
- **`merge-pull-request` safe-output parity** — `samples` + cross-repo targeting (still `max: 1`, experimental). (v0.80.4)

## Items resolved into v0.79.x stable

These items previously sat on this watchlist and have now shipped in stable; they are documented in `SKILL.md` / `migrations.md`:

- **Effective-tokens → AI Credits (AIC) rename** with `effective-tokens-to-ai-credits` codemod (`gh aw fix --write`). New defaults: 1000 AIC/run, opt-in 5000 AIC/24h daily cap, 400 AIC threat-detection cap. (v0.79.4)
- **Daily AIC guardrail behavior** — opt-in via `max-daily-ai-credits`; skipped for `workflow_call`/`repository_dispatch`/`workflow_dispatch` carrying `aw_context` metadata. Cost-management docs now exist upstream. (v0.79.4–v0.79.8)
- **`safe-outputs.timeout-minutes` field** with default raised from 15 → 45 minutes. (v0.79.4)
- **Custom `models:` frontmatter overlay** for non-catalog model pricing. (v0.79.4)
- **`create-check-run.target`** PR-targeting field (`triggering` / `"*"` / explicit). (v0.79.4)
- **`features.dangerously-disable-sandbox-agent`** literal-string-justification requirement (boolean/expression rejected). (v0.79.4 BREAKING)
- **`features.user-invokable` / `features.disable-model-invocation` removed** from schema (validation error). (v0.79.4 BREAKING)
- **`engine.max-turns` → top-level `max-turns`** with `engine-max-turns-to-top-level` codemod. (v0.79.4)
- **AWF firewall upgraded to 0.27.2** + Go MCP server 4-process child-`gh` guardrail. (v0.79.6)
- **`gh-aw.aic` emitted as `doubleValue` on OTLP conclusion spans.** (v0.79.6)
- **`environment:` propagation to detection job** + `set_issue_field` GraphQL fix + `create_issue.labels` accepts comma-separated string + Copilot arbitrary `HOME` + `--gh-aw-ref` SHA pinning at compile time. (v0.79.8)

## Unreleased main-branch / prerelease items to watch

Do not copy these into stable guidance until they appear in stable release notes and are cross-checked against reference docs:

- **v0.81.0–v0.81.3 prereleases** — Newer than the v0.80.9 stable baseline but **not yet stable**; do not promote any v0.81.x-only behavior until it ships in a stable release and is cross-checked against reference docs.
- **`slash_command` command-name wildcard suffix `*`** — The v0.80.7 release notes mention a trailing-`*` family match for slash commands, but this could **not** be confirmed in the v0.80.9 `command-triggers.md` reference or the workflow schema (where the only `*` is the `events:` selector). Do not document a command-name wildcard until upstream docs/schema confirm it.
- **`auto_upgrade: true` in `aw.json`** → weekly `agentic-auto-upgrade.yml` — Release-noted in v0.80.7 but absent from the workflow JSON schema (it would live in `aw.json`, not frontmatter); confirm the exact key/location before documenting.
- **Compiler auto-pinning of unversioned `uses:` action refs** — Release-noted in v0.80.7 but not confirmable in the v0.80.9 `security.md` reference; verify the exact behavior (and whether compile fails closed on unresolvable pins) before promoting.
- **v0.79.5 / v0.79.7 prereleases** — Check release notes for any prerelease-only knobs before promoting; v0.79.5 / v0.79.7 carry incremental fixes that mostly landed in v0.79.6 / v0.79.8 stables.
- **Copilot SDK driver/harness** — `copilot_harness: drive Copilot via @github/copilot-sdk when copilot-sdk: true` plus SDK stdin/setup follow-ups. Still rolling out.
- **`timeout-minutes` templating support** beyond the main agent job (`workflow_call` input forwarding works today; expanded surfaces still TBD).
- **Safe-output token placeholder handling** — main keeps safe-output token placeholders out of runtime `config.json`.
- **GitHub OIDC/WIF detection permission** — main adds `id-token: write` to the detection job under `engine.auth: github-oidc`. Verify before relying on OIDC in detection.
- **Partial-clone / sparse-checkout** fixes and safe-output ref-fetch changes still landing.
- **Cross-repo `create_pull_request` validation** fixes and `pull-request-target-checkout-false` codemod safety fixes.
- **Designer / drift-audit tooling** — Portable `agentic-workflow-designer` skill and `designer-drift-audit` workflow continue to evolve in main; treat as experimental authoring tooling until stable.
- **Code Simplifier per-run hard budgets** — Internal codemod safety nets; surfaces may stabilize in a later release.

## GitHub Next / "next" signals

- No `gh-aw next` product documentation or release-note item was found.
- `githubnext` strings in the gh-aw repo currently refer to vendored imports or historical workflow sources, including `githubnext/agentics` reporting guidance and a `repo-mind-light` shared workflow import. Treat these as imported workflow components, not gh-aw stable schema or product behavior.
- The documentation blog references GitHub Next/agentics workflows such as Daily Documentation Updater, Glossary Maintainer, and Documentation Unbloat as add-wizard sources. These are workflow examples, not new gh-aw runtime semantics.

## Azure DevOps / AzDO signals

- No AzDO or Azure DevOps workflow/product documentation was found in stable release notes or tracked gh-aw workflow docs.
- Search hits were limited to hosted-runner inventory/research files listing the Azure CLI Azure DevOps extension and `pkgs.dev.azure.com` as a package-domain allowlist entry. These are not gh-aw Azure DevOps product features.
- The shared Azure MCP import is generic Azure resource discovery in read-only mode; do not describe it as Azure DevOps/AzDO support.
