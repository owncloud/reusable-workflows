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
- No tarball matching `artifact-glob` carries development files -- see
  [Development files](#development-files).
- The glob matched at least one file, and the published release is not empty
  (`fail_on_unmatched_files`).

## Development files

A release artifact must be the app's packaged output, never the working tree it
was built in. The workflow rejects a tarball that contains:

| rejected | scope |
|---|---|
| `.git` | any depth |
| `.git`, `.github`, `tests`, `build`, `vendor-bin` | the app's own top level |

The directory names are anchored to the app's top level on purpose, so a
vendored dependency shipping its own `tests/` is unaffected. Only `.git` is
matched at any depth, because `migrate_to_ocis` 3.0.0 shipped three of them
under `vendor/`.

If this fails, `artifact-glob` is almost certainly pointing at the build tree or
at the *source* artifact. Point it at the packaged one -- usually
`build/artifacts/appstore/<app>.tar.gz` or `build/dist/<app>.tar.gz`.

This check exists because
[owncloud/core#41824](https://github.com/owncloud/core/issues/41824): ownCloud
11.0.0 shipped 13 bundled apps as build working trees -- roughly 102 MB of
development material, 16 git repositories, and `files_antivirus`'s EICAR
acceptance data, which made anti-virus scans of the release tarball fail. Since
the signature above covers those files too, an administrator could not delete
them without breaking `occ integrity:check-app`. That is why this is a hard
failure at publish time rather than something to clean up afterwards.

It complements, rather than duplicates, the `ocsign` checkout guard: `ocsign`
refuses to sign a staging tree that holds a `.git` entry, while this inspects the
finished tarball and also covers `tests`, `build`, `.github` and `vendor-bin`,
which `ocsign` accepts.

## Known app Makefile issues

- **`notes`, `calendar`** wrap the sign call in `mv $(configdir)/config.php ...`,
  a workaround for `occ` reading the app's own config. Without a core checkout
  that path does not exist and the `mv` fails -- delete both lines.
- **`testing`** has no signing block, so its `make dist` produces an unsigned
  tarball and the signature assertion fails by design.
