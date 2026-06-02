# agents.md -- Reusable Workflows

## Repository Overview

Collection of reusable GitHub Actions workflow definitions for the ownCloud organization. No license file detected. Used across multiple repositories for standardized CI/CD.

- **Product family:** Infrastructure / Tooling
- **Primary language(s):** YAML

## Architecture & Key Paths

- `.github/workflows/` -- Reusable workflow definitions
- Each workflow file is a standalone reusable unit called by other repositories.
- Inputs and secrets are declared at the top of each workflow file.

## Development Conventions

- GitHub Actions YAML workflow syntax
- Called via `uses: owncloud/reusable-workflows/.github/workflows/<name>@<ref>`
- Pin callers to a specific commit SHA or tag for reproducibility.

## Build & Test Commands

No build commands -- these are declarative YAML workflow definitions.

## Important Constraints

- No license file detected. The OSPO is reviewing licensing for all repositories, with a goal of migrating to Apache 2.0 where possible. Repos with copyleft dependencies require auditing first.
- Do not introduce new **copyleft-licensed dependencies** (GPL, AGPL, LGPL, MPL) without explicit discussion in an issue first. This is especially important for repos that are migrating to or already under Apache 2.0, as copyleft dependencies would block or complicate that migration.
- Changes affect all repositories that reference these workflows.
- All contributions require a DCO sign-off.


## OSPO Policy Constraints

### GitHub Actions
- **Only** use actions owned by `owncloud`, created by GitHub (`actions/*`), verified on the GitHub Marketplace, or verified by the ownCloud Maintainers.
- Pin all actions to their full commit SHA (not tags): `uses: actions/checkout@<SHA> # vX.Y.Z`
- Never introduce actions from unverified third parties.

### Dependency Management
- Dependabot is configured for automated dependency updates.
- Review and merge Dependabot PRs as part of regular maintenance.
- Do not introduce new dependencies without discussion in an issue first.

### Git Workflow
- **Rebase policy**: Always rebase; never create merge commits. Use `git pull --rebase` and `git rebase` before pushing.
- **Signed commits**: All commits **must** be PGP/GPG signed (`git commit -S -s`).
- **DCO sign-off**: Every commit needs a `Signed-off-by` line (`git commit -s`).
- **Conventional Commits & Squash Merge**: Use the [Conventional Commits](https://www.conventionalcommits.org/) format where the repository enforces it. Many repos use squash merge, where the PR title becomes the commit message on the default branch — apply Conventional Commits format to PR titles as well. A reusable GitHub Actions workflow enforces this.

## Context for AI Agents

This repository contains shared GitHub Actions workflows. Modifying a workflow here may affect CI/CD across the entire ownCloud organization. Changes should be tested carefully and versioned.
