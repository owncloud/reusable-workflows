# Auditing workflows with zizmor

`zizmor.yml` runs [zizmor](https://docs.zizmor.sh) over the calling repository's GitHub Actions
workflows, composite actions and Dependabot config. Any finding, at any severity, fails the job,
and zizmor annotates the offending line.

## Caller

```yaml
name: Zizmor

on:
  push:
    branches:
      - master
  pull_request:
    types:
      - opened
      - synchronize
      - reopened
  schedule:
    # Weekly: online audits (known-vulnerable actions, impostor commits) can
    # start failing without a code change.
    - cron: '0 4 * * 1'
  workflow_dispatch:

permissions:
  contents: read

# Grouped per event, so a push to the default branch can never cancel or
# displace the weekly run; only pull request runs cancel superseded ones.
concurrency:
  group: zizmor-${{ github.event_name }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  zizmor:
    name: Zizmor
    uses: owncloud/reusable-workflows/.github/workflows/zizmor.yml@main
```

Use the repository's default branch under `push:`. Do not add a `paths:` filter. The check is
meant to be required, and a required check that never reports blocks every merge.

## Configuration

Without a repository config, the workflow applies the org default:

```yaml
rules:
  unpinned-uses:
    config:
      policies:
        owncloud/*: ref-pin
        "*": hash-pin
```

That default allows `owncloud/...@main` and requires every other action to be pinned by commit
SHA. The org Actions allowlist enforces SHA pinning anyway.

A `zizmor.yml` or `.github/zizmor.yml` in the repository **replaces** this default; the two are
not merged. If you add one, copy the `unpinned-uses` block into it.

## Suppressing a finding

Fix the finding where you can. Where you cannot, suppress it on the flagged line and say why:

```yaml
        image: ${{ matrix.database }} # zizmor: ignore[unpinned-images] the tag comes from the caller's input
```

The comment must sit on a line inside the finding's span. For a `run:` script, that means a
YAML comment on the line above `run:`, not a shell comment inside the script.

## Running it locally

```sh
pipx run zizmor==1.31.1 --config org-default.yml .
```

Here `org-default.yml` holds the default config above. Without it, zizmor's own default flags
every `owncloud/...@main` reference. Leave out `--config` if the repository has its own config.

Use `--fix=all` to apply zizmor's auto-fixes, then review them. They are not always right: for
example, it writes `# master` as the version comment of a SHA that a stray `master` tag also
points to.
