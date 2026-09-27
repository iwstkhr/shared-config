# shared-config

Configuration templates shared across personal repositories.

## pre-commit

[mise.toml](mise.toml) pins the pre-commit version for this repository. With mise installed, set up the tools and Git hooks:

```bash
mise install
mise exec -- pre-commit install
```

[.pre-commit-config.yaml](.pre-commit-config.yaml) is the canonical config for personal repositories. pre-commit has no config inheritance, so copy it into each new repository, together with [.markdownlint-cli2.yaml](.markdownlint-cli2.yaml), the markdownlint-cli2 config that disables MD013 (line length) and MD029 (ordered list item prefix).

```bash
cp ~/git/shared-config/.pre-commit-config.yaml ~/git/shared-config/.markdownlint-cli2.yaml .
pre-commit install
```

`default_install_hook_types` in the config makes `pre-commit install` set up both the `pre-commit` and the `commit-msg` hook, so no extra flags are needed. Repositories that ran `pre-commit install` before the `commit-msg` hook was added need to run it once more.

After copying, Renovate (`:enablePreCommit`) keeps each repository's `rev` values up to date.

### Included hooks

| Hook | Target |
| --- | --- |
| check-yaml | YAML syntax |
| end-of-file-fixer / trailing-whitespace | All files |
| actionlint | `.github/workflows/*.yml` |
| ruff-check / ruff-format | Python |
| biome-check | JavaScript / TypeScript |
| gitleaks | Secret detection |
| markdownlint-cli2 | Markdown |
| shellcheck | Shell scripts |
| conventional-pre-commit | Commit messages (`commit-msg` stage) |

Hooks with no matching files are skipped, so unused hooks can stay in place.

### Conventional Commits

`conventional-pre-commit` rejects commit messages that do not follow [Conventional Commits](https://www.conventionalcommits.org/), such as `feat: add login form` or `fix(api): handle empty payload`.

It runs on the `commit-msg` stage, which has two consequences:

1. `pre-commit run --all-files`, including the CI workflow below, does not check commit messages. Enforcement happens locally at commit time, and `git commit --no-verify` bypasses it
2. To check a message by hand, pass the file explicitly

```bash
pre-commit run --hook-stage commit-msg --commit-msg-filename .git/COMMIT_EDITMSG
```

Allowed types default to `build`, `chore`, `ci`, `docs`, `feat`, `fix`, `perf`, `refactor`, `revert`, `style`, and `test`, and `fixup!`/`squash!` and merge commits pass as-is. Useful `args` per repository:

- `[feat, fix, docs, chore]` narrows the allowed types (`feat` and `fix` are always allowed)
- `[--force-scope]` requires a scope, `[--scopes, api,client]` restricts it
- `[--strict]` also rejects `fixup!`/`squash!` and merge commits

### Per-repository adjustments

- Change the `check-yaml` args to `[--unsafe]` for YAML with custom tags, such as CloudFormation templates
- Add `exclude:` for files that cause false positives, such as generated files (e.g. `exclude: ^src/content/` for markdownlint-cli2)

### GitHub Actions

[.github/workflows/pre-commit.yml](.github/workflows/pre-commit.yml) is a reusable workflow that runs all hooks against all files. It installs the same pre-commit version as [mise.toml](mise.toml), and Renovate updates both through `customManagers:githubActionsVersions` in the shared preset. When the calling repository has a `mise.toml`, the workflow also sets up [mise](https://mise.jdx.dev/) and installs the tools it pins, and uses the pre-commit from `mise.toml` if it is listed there. Otherwise it installs pre-commit with pip. Call it from each repository with `.github/workflows/pre-commit.yml`:

```yaml
name: Pre-commit

on:
  pull_request:
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  pre-commit:
    uses: iwstkhr/shared-config/.github/workflows/pre-commit.yml@main
```

This repository is public, so any repository can call the workflow without changing its Actions access settings.

## Renovate

[renovate-preset.json](renovate-preset.json) is a shared [Renovate](https://docs.renovatebot.com/) preset. Repositories apply the common dependency update rules by extending it in their Renovate config (e.g. `renovate.json`):

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>iwstkhr/shared-config:renovate-preset"]
}
```

Renovate reads the preset from the default branch, so changes take effect in every repository once they are merged into `main`.

### Extends

| Preset | Purpose |
| --- | --- |
| `config:recommended` | Renovate's recommended settings |
| `helpers:pinGitHubActionDigests` | Pin GitHub Actions to digests |
| `:enablePreCommit` | Enable updates for pre-commit hooks |
| `customManagers:biomeVersions` | Detect Biome versions with a custom manager |
| `customManagers:githubActionsVersions` | Update `_VERSION` environment variables marked with `# renovate:` comments in GitHub Actions workflows |

### Common options

- **labels**: Add the `dependencies` label to created PRs
- **timezone**: `Asia/Tokyo`
- **dependencyDashboard**: Enable the Dependency Dashboard issue
- **minimumReleaseAge**: Wait 7 days after a release before updating
- **rebaseWhen**: Rebase only when there are conflicts

### Lock file maintenance

Creates a lock file maintenance PR before 5:00 AM (Asia/Tokyo) every Monday and automerges it.

### packageRules

| Condition | Behavior |
| --- | --- |
| Major updates | Add the `breaking-change` label (no automerge) |
| Minor / patch updates | Group as `non-major dependencies` and automerge |
| GitHub Actions pin / digest updates | Automerge |

### Validation

[.github/workflows/renovate-validate.yml](.github/workflows/renovate-validate.yml) validates `renovate-preset.json` and `renovate.json` with the [Renovate config validator](https://docs.renovatebot.com/config-validation/) when either file changes. To run the same check locally:

```bash
npx --yes --package renovate -- renovate-config-validator --strict --no-global renovate-preset.json renovate.json
```

`--no-global` validates the files as repository config rather than self-hosted global config, and `--strict` also fails when an option needs migration. The validator does not fetch presets referenced in `extends`, so a wrong preset name only shows up on the Dependency Dashboard after Renovate runs.
