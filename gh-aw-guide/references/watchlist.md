# gh-aw Watchlist (Not Stable Guidance)

This file tracks upstream main-branch and experimental signals that should not be treated as current gh-aw behavior until a stable release note ships them. Stable guidance remains in `SKILL.md`, `architecture.md`, and `migrations.md`.

Last checked: 2026-06-04. Stable baseline: v0.77.5.

## Stable findings from this refresh

- No stable releases newer than v0.77.5 were available.
- The latest stable release notes contained no glossary-maintainer, documentation-maintainer, GitHub Next, Azure DevOps, or AzDO product changes.
- `.github/workflows/glossary-maintainer.md` is present in v0.77.5 and unchanged on main at the time of this refresh. It is a workflow exemplar, not a runtime feature.

## Unreleased main-branch items to watch

Do not copy these into stable guidance until they appear in stable release notes and are cross-checked against reference docs:

- Copilot SDK driver/harness work: `copilot_harness: drive Copilot via @github/copilot-sdk when copilot-sdk: true`, follow-up SDK stdin/setup fixes, and partial rollout to Copilot-backed workflows.
- Daily effective-token guardrail behavior after v0.77.5: main includes a change to emit the daily ET guardrail by default and disable only on explicit `-1`, which differs from the v0.77.5 release note that gated setup on explicit configuration.
- `timeout-minutes` templating support in schema/custom job compilation.
- Safe-output token placeholder handling: main keeps safe-output token placeholders out of runtime `config.json`.
- GitHub OIDC/WIF detection permission fix: main adds `id-token: write` to the detection job under `engine.auth: github-oidc`.
- Main-branch docs around `max-daily-effective-tokens` cost-management guidance.
- Checkout/push guidance around absent git credentials after checkout and `push_to_pull_request_branch` behavior in multi-checkout workflows.
- Partial-clone/sparse-checkout fixes and safe-output ref-fetch changes.
- Cross-repo `create_pull_request` validation fixes and `pull-request-target-checkout-false` codemod safety fixes.
- Safe-output completion hardening after v0.77.5: main contains several changes requiring workflows/agents to emit explicit terminal safe-output calls and to fall back when engine/tool permissions block normal output. Do not document exact behavior until a stable release describes it.
- Safe-output bundle/patch integrity fixes after v0.77.5: main includes fixes for patch/bundle desynchronization and a file-protection bypass via patch-parser differential. Treat as a high-priority stable-release review item when the next stable ships.
- Premium request / PRU removal: main removes premium-requests support from compiler, JS, and docs. Wait for stable release notes before changing Copilot billing/cost guidance.
- Agentic workflow designer skill and designer-drift-audit workflow: main contains a portable `agentic-workflow-designer` skill plus a designer-drift workflow. Treat as experimental workflow-design tooling until it appears in stable docs/releases.

## GitHub Next / "next" signals

- No `gh-aw next` product documentation or release-note item was found.
- `githubnext` strings in the gh-aw repo currently refer to vendored imports or historical workflow sources, including `githubnext/agentics` reporting guidance and a `repo-mind-light` shared workflow import. Treat these as imported workflow components, not gh-aw stable schema or product behavior.
- The documentation blog references GitHub Next/agentics workflows such as Daily Documentation Updater, Glossary Maintainer, and Documentation Unbloat as add-wizard sources. These are workflow examples, not new gh-aw runtime semantics.

## Azure DevOps / AzDO signals

- No AzDO or Azure DevOps workflow/product documentation was found in stable release notes or tracked gh-aw workflow docs.
- Search hits were limited to hosted-runner inventory/research files listing the Azure CLI Azure DevOps extension and `pkgs.dev.azure.com` as a package-domain allowlist entry. These are not gh-aw Azure DevOps product features.
- The shared Azure MCP import is generic Azure resource discovery in read-only mode; do not describe it as Azure DevOps/AzDO support.
