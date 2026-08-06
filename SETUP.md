# Template Setup Guide

## Quick Start

1. Create a new repository from this template
2. Clone your new repository
3. Run `./setup.sh` and follow the prompts
4. Review the remaining manual steps printed by the script
5. **(Admin)** Run `./setup-github-protection.sh --dry-run` to preview, then without the flag to apply branch protection and repo settings
6. Delete `setup.sh`, `setup-github-protection.sh`, and this file

## Template Values

The template ships with working test values for its own CI/CD pipeline.
When you run `setup.sh`, these are replaced with your project's values.

| Template Value      | Location                                                        | Description                            |
| ------------------- | --------------------------------------------------------------- | -------------------------------------- |
| `test`              | cicd-standard.yaml, cicd-composable.yaml (`jf-project`)         | JFrog Artifactory project name         |
| `test-build`        | cicd-standard.yaml, cicd-composable.yaml (`jf-build-name`)      | JFrog build identifier                 |
| `gh-dev-test`       | cicd-standard.yaml, cicd-composable.yaml (`oidc-provider-name`) | JFrog OIDC provider for GitHub Actions |
| `aerospike/testing` | cicd-standard.yaml, cicd-composable.yaml (`oidc-audience`)      | OIDC audience claim                    |
| `[PROJECT_NAME]`    | README.md, CONTRIBUTING.md                                      | Human-readable project name            |
| `[REPOSITORY_NAME]` | README.md                                                       | GitHub repository name                 |
| `CODEOWNERS`        | .github/CODEOWNERS                                              | Team/user who owns the code            |

## Required GitHub Secrets

Add these secrets in your repository settings (Settings > Secrets and variables > Actions):

### Required (artifact signing)

| Secret           | Description                           |
| ---------------- | ------------------------------------- |
| `GPG_SECRET_KEY` | GPG private key for signing artifacts |
| `GPG_PUBLIC_KEY` | GPG public key                        |
| `GPG_PASS`       | GPG key passphrase                    |

### Optional (NuGet package signing via SSL.com)

| Secret           | Description              |
| ---------------- | ------------------------ |
| `ES_USERNAME`    | SSL.com eSigner username |
| `ES_PASSWORD`    | SSL.com eSigner password |
| `CREDENTIAL_ID`  | SSL.com credential ID    |
| `ES_TOTP_SECRET` | SSL.com TOTP secret      |

## Customizing the Build Script

The `build-script` in the CI/CD workflow files creates a dummy artifact.
Replace it with your actual build commands:

```yaml
build-script: |
  set -euo pipefail
  # Your build commands here
  make build
```

The script runs with these environment variables available:

- `MATRIX_JSON`: JSON object with current matrix values (distro, arch, etc.)
- Any variables from the `build-env` input
- You **must** `export` variables for child processes (make, docker) to see them

## Multi-Platform Builds

To build for multiple distros/architectures, add `matrix-json` to the cicd job
(see the comments in `cicd-standard.yaml` for examples):

```yaml
matrix-json: >-
  {"include":[
    {"runs-on":"ubuntu-22.04","distro":"jammy","arch":"x86_64"},
    {"runs-on":"ubuntu-22.04","distro":"el9","arch":"x86_64"}
  ]}
```

## Versioning

The CI/CD pipeline uses the `extract-version-from-tag` action to determine the
artifact version. During setup you choose one of two strategies:

- **VERSION file** (default): The version is read from the `VERSION` file at
  the repo root. Update this file when you want to bump the version.
- **Git tags**: The version is extracted from git tags (e.g., `v1.2.3` becomes
  `1.2.3`). Push a tag to trigger a versioned release.

You can switch strategies at any time: delete the VERSION file to use tags, or
create one to override tags.

## Dependabot

The template has Dependabot configured for GitHub Actions updates.
Uncomment the appropriate section in `.github/dependabot.yml` for your
package ecosystem (pip, npm, gomod, or docker).

## Shared Workflows Version

The CI/CD workflows reference
[aerospike/shared-workflows](https://github.com/aerospike/shared-workflows)
at a specific commit SHA. Dependabot will propose updates when new versions
are released.

## Repository Protection

The template includes a standalone script to configure branch protection and
repo-level merge settings via the GitHub API. This is separate from `setup.sh`
because it requires admin access and an authenticated `gh` CLI.

### Usage

```bash
# Preview what will be applied:
./setup-github-protection.sh --dry-run

# Apply settings:
./setup-github-protection.sh
```

### Requirements

- `gh` CLI installed and authenticated (`gh auth login`)
- Admin access to the repository
- Token scopes: `repo` (classic) or `Administration: read/write` (fine-grained)

### What It Configures

**Repo-level ruleset (`protect_main`)**, applied on top of the org-level
baseline (`protect_default_branch_0001`). Only includes the delta:

- Required review thread resolution
- Squash-only merges
- Required commit signatures
- Required status checks (`Trunk Check` + `validate-jira-ticket / hygiene-check`)
- Strict status checks (branch must be up to date)

**Repository settings:**

- Auto-delete head branches after merge
- Only squash merges allowed (merge commits and rebase disabled)

### Manual Fallback

If you cannot run the script, configure these settings manually:

1. **Settings > Rules > Rulesets**: create a ruleset named `protect_main`
   targeting the default branch with the rules listed above
2. **Settings > General**: set "Allow squash merging" only, enable
   "Automatically delete head branches"

## Workflows Overview

| Workflow               | Trigger           | Purpose                                           |
| ---------------------- | ----------------- | ------------------------------------------------- |
| `cicd-standard.yaml`   | _(commented out)_ | Standard orchestrated build/sign/deploy pipeline  |
| `cicd-composable.yaml` | _(commented out)_ | Composable pipeline with custom inter-stage steps |
| `pr-hygiene.yml`       | Pull requests     | Validate JIRA ticket reference in PR title        |
| `trunk.yml`            | Push to main, PRs | Trunk Check linting                               |
