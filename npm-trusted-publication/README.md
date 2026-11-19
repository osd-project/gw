# npm trusted publication (bootstrap)

Reusable workflow that sets up [npm trusted publishing](https://docs.npmjs.com/trusted-publishers) (GitHub Actions OIDC) for public `@osd-project` npm packages.

Workflow file: [`.github/workflows/npm-trusted-publication.yml`](../.github/workflows/npm-trusted-publication.yml)


## What it does

npm requires a package to exist before a trusted publisher can be configured, and the first publish cannot use OIDC. This workflow handles both:

1. **Validate inputs** - the package manifest path must be relative, contain no `..`, end in `package.json` and exist, and `package-name` must equal the `name` in that file.
2. **Initial publish (new packages only)** - if the package is not on npm, it publishes version `0.0.0` once using `NPM_TOKEN` and a one-time password generated from `NPM_TOTP_DEVICE`.
3. **Configure the trusted publisher** - it registers the caller repository, publish workflow file and GitHub environment with npm (`POST /-/package/{package}/trust`).
   - Already matches the inputs: nothing is changed.
   - A different configuration exists: it is deleted and replaced with the requested one.

After this has run, the package is published by your own workflow with OIDC and no npm token.

## Prerequisites

- The repository belongs to the `osd-project` organization and has access to the org secrets `NPM_TOKEN` and `NPM_TOTP_DEVICE`.
- A GitHub environment (default `npm-publish`) exists in the repository. Recommended: required reviewers and a deployment-branch restriction.
- `package.json` has the right `name` and a `repository.url` that exactly matches the GitHub repository.
- The org's Actions policy allows `osd-project/gw/*`.

## Secrets (required)

| Secret | Description |
| --- | --- |
| `NPM_TOKEN` | npm granular access token from the osd-project publishing service account, read/write on the `@osd-project` scope |
| `NPM_TOTP_DEVICE` | Base32 TOTP secret of the same npm account (not a 6-digit code) |

The caller must pass them explicitly under `secrets:`, or use `secrets: inherit`.

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `package-name` | yes | | npm package name, must match `package.json` |
| `publish-workflow` | yes | | Filename (not path) of the workflow that will publish with OIDC, e.g. `npm-publish.yml` |
| `repository` | yes | | Caller repository in `owner/repo` form, normally `${{ github.repository }}` |
| `package-json-file-path` | no | `package.json` | Relative path to the package manifest |
| `github-environment` | no | `npm-publish` | GitHub environment registered with npm |

## Usage

### Single package

```yaml
name: npm trusted publishing

on:
  workflow_dispatch:

jobs:
  bootstrap:
    uses: osd-project/gw/.github/workflows/npm-trusted-publication.yml@main
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
      NPM_TOTP_DEVICE: ${{ secrets.NPM_TOTP_DEVICE }}
    with:
      package-name: '@osd-project/example-package'
      package-json-file-path: package.json
      publish-workflow: npm-publish.yml
      repository: ${{ github.repository }}
      github-environment: npm-publish
```

Pin to a release tag or commit SHA instead of `@main` for production use.

### Multiple packages

Trusted publishing is configured per package, so use a matrix:

```yaml
name: npm trusted publishing

on:
  workflow_dispatch:

jobs:
  bootstrap:
    strategy:
      matrix:
        include:
          - package-name: '@osd-project/package-a'
            package-json-file-path: packages/package-a/package.json
          - package-name: '@osd-project/package-b'
            package-json-file-path: packages/package-b/package.json
    uses: osd-project/gw/.github/workflows/npm-trusted-publication.yml@main
    secrets: inherit
    with:
      package-name: ${{ matrix.package-name }}
      package-json-file-path: ${{ matrix.package-json-file-path }}
      publish-workflow: publish.yml
      repository: ${{ github.repository }}
      github-environment: npm-publish
```

## After bootstrapping

Switch your publish workflow (the file named in `publish-workflow`) to OIDC:

- Node.js >= 22.14.0 and npm CLI >= 11.5.1 (Yarn Berry >= 4.10.3; Yarn Classic is not supported).
- Remove `NPM_TOKEN` and `NPM_TOTP_DEVICE` from that workflow and its environment.
- Run on a GitHub-hosted runner and use the same environment that was registered.
- Add:

```yaml
permissions:
  id-token: write   # required for OIDC trusted publishing
  contents: read    # use write only if the workflow commits, tags or creates releases
```

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| `Secret NPM_TOKEN is required, but not provided` | The caller does not pass the secrets. Add the `secrets:` block |
| `package-name does not match package.json name` | `package-name` differs from `name` in the file at `package-json-file-path` |
| `E404 Not Found` on publish | npm returns 404 for bad credentials. Check the token's scope and that both secrets are passed |
| `EOTP` | Wrong or missing TOTP secret. If it is sporadic, re-run after 30 seconds |
| Trusted publisher registered but OIDC publish fails | Repository, workflow filename or environment do not match what is registered (case-sensitive), or `repository.url` in `package.json` does not match the repo |

npm does not validate the trusted-publisher configuration when it is saved, and a new configuration expires if no successful publish happens within 2 days.


