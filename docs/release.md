# Releasing ownCloud Classic apps

`release.yml` builds, signs and publishes an ownCloud Classic app on a tag push.

The workflow runs `make dist` with the app's `sign` variable overridden so the
bundle is signed by [`ocsign`](https://github.com/owncloud/ocsign) instead of
`occ integrity:sign-app`. That removes the need for a bootstrapped ownCloud
server, which is why app Makefiles used to skip signing silently in CI. Signing
is mandatory on ownCloud 11+, so a tarball without a valid
`appinfo/signature.json` now fails the run rather than being published unsigned.

No app Makefile change is required -- the override works with the
`ifdef CAN_SIGN` / `$(sign) --path=...` idiom every app already uses.

## Usage

```yaml
name: Release
on:
  push:
    tags: ['v*']
permissions:
  contents: read
jobs:
  release:
    permissions:
      contents: write        # required - a called workflow cannot elevate this
    uses: owncloud/reusable-workflows/.github/workflows/release.yml@main
    with:
      app-name: myapp
```

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `app-name` | repository name | Artifact name in dry-run mode |
| `app-repository` | current repository | Must be within `owncloud/` |
| `environment` | none | GitHub Environment providing the signing secrets (see below) |
| `ocsign-ref` | `7d83a313...` (v0.2.1) | Commit SHA of `owncloud/ocsign` to build the signing tool from |
| `artifact-glob` | `build/**/*.tar.gz` | Tarball(s) produced by `make dist` |
| `dry-run` | `false` | Skip the tag check and release creation; upload an artifact instead |

`ocsign-ref` must always be a full commit SHA with a `# vX.Y.Z` comment, never a
movable tag. Dependabot does not track a `repository:`/`ref:` checkout of a
sibling repo, so this pin is bumped manually.

## Secrets

Provide these as repository secrets on the app repository, or pass them from the
caller:

| Secret | Description |
| --- | --- |
| `SIGNING_KEY` | PEM encoded private key (EC P-384 or RSA) |
| `SIGNING_CERT` | PEM encoded leaf certificate -- its `CN` **must** equal the app id from `appinfo/info.xml` |
| `SIGNING_CHAIN` | PEM encoded intermediate chain, e.g. `intermediate-g2.crt` (optional) |

Certificates are issued via
[`owncloud/developer-certificates`](https://github.com/owncloud/developer-certificates).

### Protecting the key with an environment

To gate the signing key behind deployment protection rules -- required
reviewers, or a branch/tag restriction so only release tags can sign -- create a
[GitHub Environment](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments),
put the secrets there and name it via the `environment` input:

```yaml
    with:
      app-name: myapp
      environment: release
```

The environment must already exist on the app repository. Leave `environment`
unset to use plain repository secrets. Environment secrets take precedence over
caller-passed secrets of the same name.

## What the workflow enforces

- The tag, minus a leading `v`, matches `<version>` in `appinfo/info.xml`.
- `SIGNING_KEY` and `SIGNING_CERT` are non-empty before `make dist` runs.
- Every tarball matching `artifact-glob` contains an `appinfo/signature.json`
  that is a valid schema v2 envelope whose embedded leaf `CN` equals the app id.
- The glob matched at least one file, and the published release is not empty
  (`fail_on_unmatched_files`).

## Known app Makefile issues

- **`notes`, `calendar`** wrap the sign call in `mv $(configdir)/config.php ...`,
  a workaround for `occ` reading the app's own config. Without a core checkout
  that path does not exist and the `mv` fails -- delete both lines.
- **`testing`** has no signing block, so its `make dist` produces an unsigned
  tarball and the signature assertion fails by design.
