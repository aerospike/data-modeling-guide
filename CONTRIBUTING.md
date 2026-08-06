# Contributing to [PROJECT_NAME]

Thank you for your interest in contributing to this Aerospike project! We welcome contributions from the community.

## How to Contribute

### **Did you find a bug?**

- **Do not open up a GitHub issue if the bug is a security vulnerability**, and instead refer to our [security policy](SECURITY.md)

- If you're unable to find an open issue addressing the problem, be sure to include a **title and clear description**, as much relevant information as possible, and a **code sample** or an **executable test case** demonstrating the expected behavior that is not occurring.

### **Did you write a patch?**

- Open a new GitHub pull request with the patch.

- Ensure the PR description clearly describes the problem and solution. Include the relevant issue number if applicable.

## Development Setup

### Repo Tooling

Linting will be run on PRs; you can save yourself some time and annoyance by linting as you write.

If you use Visual Studio Code or a derivative, there are suggested extensions in the [.vscode](.vscode) directory.

### Trunk

Trunk can also be run as a CLI. Once installed, you can run `trunk git-hooks sync` to check and make sure that your code will pass CI.

### Linter notes

`kennylong.kubernetes-yaml-formatter`: **Do NOT install or enable this extension.** It is marked as unwanted in `.vscode/extensions.json` because it conflicts with Trunk and Prettier on YAML formatting rules. If you have yaml format-on-save enabled with kennylong's extension, `trunk check|fmt` will complain about it.

`streetsidesoftware.code-spell-checker`: This isn't enabled via trunk and you should run it in your editor of choice. Trunk marks all misspelled words as errors, when they should properly be notes (blue squiggles, not red squiggles).

### Pull Requests

With exceptions (see below) PR titles must follow conventional commit format:

```yaml
type(scope): description
```

#### Default Rules

- **type** is required and must be one of: `feat`, `fix`, `refactor`, `docs`, `test`, `ci`, `chore`, `build`, `perf`
- **scope** is optional, lowercase, in parentheses
- **description** starts with a lowercase letter

#### JIRA Ticket Requirement

A JIRA ticket in square brackets is required for these types: **feat, fix, docs, ci, refactor**.
This rule can be disabled by using the `skip-jira` label on the PR (commitlint still runs).

#### Examples

```text
feat(workflows): [INFRA-370] add integration test stage
fix: [ENG-123] correct routing logic for edge cases
docs(readme): [INFRA-451] update setup instructions
ci: [INFRA-400] switch to shared reusable workflows
chore(deps): bump shared-workflows to v3
test: add unit tests for auth module
```

#### Default Allowlisted Patterns

The following PR title patterns bypass **both** commitlint type validation and the JIRA ticket requirement:

- **Dependabot**: `chore(deps): bump ...` (any type with `deps` scope)
- **StepSecurity**: `[StepSecurity] ...`
- **Reverts**: `Revert "..."` or `revert: ...`
- **Dependency bumps**: `Bump ...`

#### Enforcement

This is enforced by the `pr-hygiene.yml` workflow which must pass before merge.
See the [pr-hygiene documentation](https://github.com/aerospike/shared-workflows/blob/main/.github/workflows/pr-hygiene/README.md)
for the full regex and configuration details.

## Contributor

This project adheres to the Contributor Covenant [Code of Conduct](CODE_OF_CONDUCT.md). By participating, you are expected to uphold this code.

## Questions?

Feel free to open an issue with your question.

## License

By contributing, you agree that your contributions will be licensed under the same license as the project (see [LICENSE](LICENSE)).
