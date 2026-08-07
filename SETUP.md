# Repository Setup

This repository was created from the Aerospike repo template. It is a
**documentation repository** — markdown only, with no source to build and no
artifacts to publish — so the template's build, signing, and JFrog Artifactory
machinery has been removed: `setup.sh`, `cicd-standard.yaml`, and
`cicd-composable.yaml` are gone, along with the sections of this guide that
described them.

What remains is the setup that still applies.

## Remaining setup

1. Write `.github/CODEOWNERS` with the owning team (still the template
   placeholder).
2. **(Admin)** Optionally run `./setup-github-protection.sh --dry-run` to
   preview branch protection, then without the flag to apply it. See
   [Repository Protection](#repository-protection) for what it enforces and
   what to settle first.
3. Once branch protection is decided, `setup-github-protection.sh` and this
   file can be deleted.

## Dependabot

The template has Dependabot configured for GitHub Actions updates.
Uncomment the appropriate section in `.github/dependabot.yml` for your
package ecosystem (pip, npm, gomod, or docker).

## Shared Workflows Version

`pr-hygiene.yml` references
[aerospike/shared-workflows](https://github.com/aerospike/shared-workflows)
at a specific commit SHA. Dependabot will propose updates when new versions
are released.

## Repository Protection

The template includes a standalone script to configure branch protection and
repo-level merge settings via the GitHub API. It is kept separate because it
requires admin access and an authenticated `gh` CLI.

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
- Required status checks (`Trunk Check` + `validate-jira-ticket / hygiene-check`)
- Strict status checks (branch must be up to date)

Commit signature enforcement (`required_signatures`) was deliberately removed
from the template's ruleset. It attests authorship of commit objects, which
matters most where commits feed a build that ships to customers. This is an
internal documentation repository with named code owners and pull-request
review; requiring every contributor to configure a GPG or SSH signing key costs
more than it returns here. Artifact signing is a separate mechanism and was
removed along with the JFrog pipelines.

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

| Workflow         | Trigger           | Purpose                                    |
| ---------------- | ----------------- | ------------------------------------------ |
| `pr-hygiene.yml` | Pull requests     | Validate JIRA ticket reference in PR title |
| `trunk.yml`      | Push to main, PRs | Trunk Check linting                        |
