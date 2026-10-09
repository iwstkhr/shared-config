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

After copying, Renovate (`:enablePreCommit`) keeps each repository's `rev` values up to date. Each `rev` is pinned to a commit SHA with a `# frozen: <tag>` comment, and Renovate updates both the SHA and the tag. A bare SHA without the comment is treated as a version and is not updated correctly, so keep the comment when adding a hook. `pre-commit autoupdate --freeze` writes the same format.

### Included hooks

| Hook | Target |
| --- | --- |
| check-yaml | YAML syntax |
| end-of-file-fixer / trailing-whitespace | All files |
| actionlint | `.github/workflows/*.yml` |
| ruff-check / ruff-format | Python, including Bandit security rules (`S`) |
| biome-check | JavaScript / TypeScript |
| gitleaks | Secret detection |
| semgrep | Static analysis for security issues and bugs |
| trivyfs-docker | Dependency vulnerabilities and IaC misconfigurations (Trivy) |
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

### Semgrep

`semgrep` scans staged files with the [`p/default`](https://semgrep.dev/p/default) ruleset and fails the commit on any finding (`--error`). Some details to keep in mind:

- The ruleset is downloaded from the Semgrep Registry on every run, so the hook needs network access and fails offline
- `args` replace the hook's default args, so the config repeats `--skip-unknown-extensions`, `--disable-version-check`, and `--quiet`. Keep them when changing `args`
- The rulesets are named explicitly rather than using `--config auto`, which requires sending metrics to Semgrep
- To ignore a false positive, add a `# nosemgrep: <rule-id>` comment on the reported line

Useful changes per repository:

- Add rulesets for the languages in use, e.g. `[--config, p/default, --config, p/python, ...]`
- Point `--config` at a rules file in the repository, e.g. `.semgrep.yml`, to run without network access
- Add `.semgrepignore` to skip paths such as generated files or test fixtures

### Trivy

`trivyfs-docker` from [mxab/pre-commit-trivy](https://github.com/mxab/pre-commit-trivy) runs [Trivy](https://trivy.dev/) in the `aquasec/trivy` Docker image and fails the commit on any HIGH or CRITICAL finding. Some details to keep in mind:

- Docker must be running locally. GitHub-hosted Ubuntu runners have it preinstalled
- The hook scans the whole repository rather than the staged files, so it runs on every commit
- The vulnerability database is downloaded on the first run and when it goes stale, so the hook needs network access
- The cache is written to `.pre-commit-trivy-cache` in the repository. Add it to `.gitignore`
- `--scanners vuln,misconfig` skips Trivy's secret scanner, since gitleaks already covers secrets
- Trivy does not read `.gitignore`, so `--skip-dirs "**/node_modules"` and `--skip-dirs "**/.venv"` keep it out of installed dependencies at any depth. Lockfiles and `requirements.txt` are still scanned
- `args` replace the hook's default args, and the last one must be the path to scan (`.`)

```bash
echo ".pre-commit-trivy-cache/" >> .gitignore
```

Useful changes per repository:

- Widen `--severity`, e.g. `MEDIUM,HIGH,CRITICAL`, or add `--ignore-unfixed` to skip vulnerabilities with no fix
- Add more `--skip-dirs <dir>` before `.` to skip paths such as test fixtures
- Add `.trivyignore` with one CVE or check ID per line to ignore a false positive

### Ruff security rules

`ruff-check` runs with `--extend-select S`, which adds the [flake8-bandit](https://docs.astral.sh/ruff/rules/#flake8-bandit-s) rules, Ruff's port of [Bandit](https://bandit.readthedocs.io/), to whatever rules the repository's Ruff config selects. The flag extends rather than replaces the selection, so `select` and `extend-select` in `pyproject.toml` or `ruff.toml` keep working.

- To ignore a false positive, add a `# noqa: <rule>` comment on the reported line, e.g. `# noqa: S603`
- `S101` flags every `assert`, including those in pytest tests. Ignore it for tests in the repository's Ruff config:

```toml
[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101"]
```

### Per-repository adjustments

- Change the `check-yaml` args to `[--unsafe]` for YAML with custom tags, such as CloudFormation templates
- Add `exclude:` for files that cause false positives, such as generated files (e.g. `exclude: ^src/content/` for markdownlint-cli2)

### GitHub Actions

[.github/workflows/pre-commit.yml](.github/workflows/pre-commit.yml) is a reusable workflow that runs all hooks against all files. It installs the same pre-commit version as [mise.toml](mise.toml), and Renovate updates both through `customManagers:githubActionsVersions` in the shared preset. When the calling repository has a `mise.toml`, the workflow also sets up [mise](https://mise.jdx.dev/) and installs the tools it pins, and uses the pre-commit from `mise.toml` if it is listed there. Otherwise it installs pre-commit with pip. Call it from each repository with `.github/workflows/pre-commit.yml`:

```yaml
name: "repo - Pre-commit"

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

To get a Slack message when pre-commit fails, pass the Slack secrets described in [Slack notifications](#slack-notifications). The workflow calls [slack-notify.yml](.github/workflows/slack-notify.yml) only on failure, and skips it when either secret is missing, such as in pull requests from forks:

```yaml
jobs:
  pre-commit:
    uses: iwstkhr/shared-config/.github/workflows/pre-commit.yml@main
    secrets:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
      SLACK_CHANNEL_ID: ${{ secrets.SLACK_CHANNEL_ID }}
```

Runs in this repository notify the same way once these secrets are set here.

This repository is public, so any repository can call the workflow without changing its Actions access settings.

## Slack notifications

[.github/workflows/slack-notify.yml](.github/workflows/slack-notify.yml) is a reusable workflow that posts a workflow result to a Slack channel with the official [Slack GitHub Action](https://github.com/slackapi/slack-github-action) (`chat.postMessage`). Renovate keeps its pinned version up to date. The message links to the calling run and shows the repository, branch, commit, and actor, color-coded by result.

### Slack setup

1. Create a Slack app with the `chat:write` bot token scope and install it to the workspace
2. Invite the app to the target channel (`/invite @<app name>`)
3. Store the bot token (`xoxb-...`) and the channel ID (e.g. `C0123456789`) as secrets in each calling repository

### Calling the workflow

Add a job that runs after the jobs to report, and pass their result as `status`:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo build

  notify:
    needs: build
    if: ${{ always() }}
    uses: iwstkhr/shared-config/.github/workflows/slack-notify.yml@main
    with:
      status: ${{ needs.build.result }}
    secrets:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
      SLACK_CHANNEL_ID: ${{ secrets.SLACK_CHANNEL_ID }}
```

- Map each secret on the right-hand side to the name used in the calling repository. When the names already match, `secrets: inherit` works too
- Use `if: ${{ failure() }}` to notify only on failure
- When `needs` lists several jobs, combine their results, e.g. `status: ${{ contains(needs.*.result, 'failure') && 'failure' || 'success' }}`

| Input | Required | Description |
| --- | --- | --- |
| `status` | Yes | `success`, `failure`, or `cancelled` get a matching color and label; any other value is shown as a warning |
| `message` | No | Plain text shown under the title. Slack markup such as links and mentions is escaped |

If the Slack API returns an error, such as `not_in_channel` or `invalid_auth`, the job fails and the error appears in the log.

## AI review

[.github/workflows/ai-review.yml](.github/workflows/ai-review.yml) is a reusable workflow that reviews a pull request with [Claude Code GitHub Actions](https://github.com/anthropics/claude-code-action) when the `ai-review` label is added. Claude posts a tracking comment with progress and a summary, plus inline comments on specific issues. Renovate adds the label to major updates (see [packageRules](#packagerules)), so those PRs are reviewed automatically.

### AI review setup

1. Install the [Claude GitHub App](https://github.com/apps/claude) on the calling repository
2. Store one of these as a secret in the calling repository:
   - `CLAUDE_CODE_OAUTH_TOKEN`: generated with `claude setup-token` (uses a Claude subscription)
   - `ANTHROPIC_API_KEY`: an Anthropic API key
3. Create the `ai-review` label in the calling repository

### Calling the AI review workflow

Add `.github/workflows/ai-review.yml` to each repository:

```yaml
name: "repo - AI Review"

on:
  pull_request:
    types: [labeled]

permissions:
  contents: read
  pull-requests: write
  issues: write
  id-token: write

jobs:
  ai-review:
    uses: iwstkhr/shared-config/.github/workflows/ai-review.yml@main
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

- The caller must grant the permissions above, because a reusable workflow cannot exceed them
- Adding any other label does not start a review, and pull requests from forks are skipped because they cannot read the secrets
- Removing and adding the label again runs a new review, cancelling one still in progress

| Input | Default | Description |
| --- | --- | --- |
| `label` | `ai-review` | Label that triggers the review |
| `allowed_bots` | `renovate[bot]` | Comma-separated bot usernames allowed to trigger the review by adding the label. Users need write access to the repository |
| `extra_prompt` | `""` | Additional instructions appended to the review prompt, e.g. `Write the review in Japanese.` |

Runs in this repository review the same way once one of the secrets is set here.

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
| Major updates | Add the `breaking-change` and `ai-review` labels alongside `dependencies` and request review from `iwstkhr` when the PR is created (no automerge). The `ai-review` label starts the [AI review](#ai-review) |
| Minor / patch updates | Group as `non-major dependencies` and automerge |
| GitHub Actions pin / digest updates | Automerge |

### Validation

[.github/workflows/renovate-validate.yml](.github/workflows/renovate-validate.yml) validates `renovate-preset.json` and `renovate.json` with the [Renovate config validator](https://docs.renovatebot.com/config-validation/) when either file changes. To run the same check locally:

```bash
npx --yes --package renovate -- renovate-config-validator --strict --no-global renovate-preset.json renovate.json
```

`--no-global` validates the files as repository config rather than self-hosted global config, and `--strict` also fails when an option needs migration. The validator does not fetch presets referenced in `extends`, so a wrong preset name only shows up on the Dependency Dashboard after Renovate runs.
