# Design: GitHub Workflow Deduplication via Composite Actions

**Date:** 2026-04-17  
**Goal:** Reduce maintenance burden by extracting duplicated workflow steps into composite actions — focusing on action version drift (primary risk) and complex conditional logic.

---

## Problem

`php-codestyle.yml`, `php-unit.yml`, and `js-unit.yml` share ~5 nearly-identical step clusters. `translation-sync.yml` and `calens.yml` duplicate token generation + PR creation. Key pain points:

- `shivammathur/setup-php` pinned by SHA — must be updated in 3 places
- `actions/create-github-app-token` already has version drift (v3.1.1 vs v1.11.6)
- Complex conditional checkout logic repeated verbatim in 3 workflows
- `occ maintenance:install` + core app enablement duplicated in 3 workflows

---

## Approach: 3 Composite Actions in `.github/actions/`

All composite actions live in this repo under `.github/actions/`. Callers reference them as `./github/actions/<name>`.

### Directory structure

```
.github/
  actions/
    setup-owncloud-env/
      action.yml
    setup-php-server/
      action.yml
    github-app-pr/
      action.yml
  workflows/
    php-codestyle.yml   (updated)
    php-unit.yml        (updated)
    js-unit.yml         (updated)
    translation-sync.yml (updated)
    calens.yml          (updated)
```

---

## Action 1: `setup-owncloud-env`

**File:** `.github/actions/setup-owncloud-env/action.yml`

**Purpose:** Clone owncloud/core + app + optional additional-app with correct refs. Consolidates the 3-step checkout cluster including `core-ref-php74` conditional logic and `app-repository` validation.

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `app-name` | yes | — | App directory name under `apps/` |
| `app-repository` | no | `''` | Override repo for app checkout (must be `owncloud/*`) |
| `core-ref` | no | `''` | Ref to use for owncloud/core checkout |
| `core-ref-php74` | no | `''` | Override core ref when PHP version is 7.4 |
| `php-version` | no | `''` | Current PHP version (used to select core ref) |
| `additional-app` | no | `''` | Optional second app to check out |
| `additional-app-github-token` | no | `''` | Token for private additional-app repo |

**Used by:** `php-codestyle`, `php-unit`, `js-unit`

**Steps consolidated:**
1. Validate `app-repository` format (`owncloud/*` namespace check)
2. `actions/checkout` → owncloud/core (with php74 ref conditional)
3. `actions/checkout` → `apps/${{ inputs.app-name }}`
4. `actions/checkout` → `apps/${{ inputs.additional-app }}` (conditional)

---

## Action 2: `setup-php-server`

**File:** `.github/actions/setup-php-server/action.yml`

**Purpose:** Install PHP, optional Linux packages, core dependencies, and ownCloud server. The single source of truth for the `shivammathur/setup-php` SHA pin.

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `php-version` | yes | — | PHP version to install |
| `additional-packages` | no | `''` | Extra `apt-get` packages |
| `database` | no | `sqlite` | One of: `sqlite`, `mysql`, `mariadb`, `postgres` |
| `database-name` | no | `owncloud` | DB name (ignored for sqlite) |
| `database-user` | no | `owncloud` | DB user (ignored for sqlite) |
| `database-pass` | no | `owncloud` | DB password (ignored for sqlite) |

**Used by:** `php-codestyle` (sqlite), `php-unit` (all databases), `js-unit` (sqlite)

**Steps consolidated:**
1. `shivammathur/setup-php@<sha>` with full extensions list and `memory_limit=1024M`
2. `sudo apt-get install $ADDITIONAL_PACKAGES` (conditional on non-empty input)
3. `make` (core dependencies)
4. `php occ maintenance:install` with database switching logic
5. Enable core apps: `files_sharing`, `files_trashbin`, `files_versions`, `provisioning_api`, `federation`, `federatedfilesharing`

---

## Action 3: `github-app-pr`

**File:** `.github/actions/github-app-pr/action.yml`

**Purpose:** Generate a GitHub App token and create/update a pull request. Fixes existing version drift between `translation-sync` (v3.1.1) and `calens` (v1.11.6) for `actions/create-github-app-token`.

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `app-id` | yes | — | GitHub App ID |
| `app-private-key` | yes | — | GitHub App private key |
| `branch` | yes | — | PR branch name |
| `commit-message` | yes | — | Commit message for the PR |
| `title` | yes | — | PR title |
| `body` | yes | — | PR body |
| `reviewers` | no | `''` | Comma-separated reviewer logins |
| `team-reviewers` | no | `''` | Comma-separated team slugs |
| `sign-commits` | no | `true` | Whether to sign commits |
| `condition` | no | `true` | Set to `false` to skip (caller controls branch conditions) |

**Used by:** `translation-sync`, `calens`

**Steps consolidated:**
1. `actions/create-github-app-token` (single pinned version)
2. `peter-evans/create-pull-request` (single pinned version)

The `condition` input lets callers pass `${{ github.ref_name == inputs.branch }}` so branch-guard logic stays in the calling workflow where it belongs semantically.

---

## What stays inline

These steps are **not** extracted — they are simple 1-2 line scripts with no action versions to drift:

- `make vendor` + `php occ a:e` (setup-app pattern)
- `make vendor` + `php occ a:e` for additional-app
- `make test-php-style`, `make test-php-phpstan`, `make test-php-phan`, `make test-php-unit`, etc.
- Translation-specific steps (l10n-read/push/pull/write/clean)

---

## Expected impact

| Metric | Before | After |
|--------|--------|-------|
| `actions/checkout` occurrences | 12 | ~5 (one per composite action + calens + translation-sync) |
| `shivammathur/setup-php` pins | 3 | 1 |
| `create-github-app-token` pins | 2 (drifted) | 1 |
| `create-pull-request` pins | 2 | 1 |
| Steps per PHP workflow | ~13 | ~7 |
