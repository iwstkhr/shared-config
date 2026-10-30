# shared-config

Configuration templates shared across personal repositories.

## Git hooks (lefthook)

Git hooks run with [lefthook](https://lefthook.dev/). [lefthook-shared.yml](lefthook-shared.yml) is the canonical config for personal repositories, and each repository loads it through `remotes` instead of copying it. Every job runs a tool found on `PATH`, and [mise.toml](mise.toml) pins lefthook and those tools for this repository. With mise installed, set up the tools and Git hooks:

```bash
mise install
lefthook install
```

`lefthook install` sets up both the `pre-commit` and the `commit-msg` hook. This repository's own [lefthook.yml](lefthook.yml) loads the shared config with `extends`.

### Setting up a repository

Add a `lefthook.yml` that loads the shared config from this repository:

```yaml
remotes:
  - git_url: https://github.com/iwstkhr/shared-config
    ref: main
    configs:
      - lefthook-shared.yml
    # Pick up changes to the shared config once a day
    refetch_frequency: 24h
```

Copy the configs that the jobs read, pin the tools in the repository's `mise.toml` (see [Tools](#tools)), and install the hooks:

```bash
cp ~/git/shared-config/{.editorconfig,.yamllint.yml,.rumdl.toml} .
lefthook install
```

- [.editorconfig](.editorconfig) requires a final newline and no trailing whitespace, except in SVG files
- [.yamllint.yml](.yamllint.yml) keeps yamllint close to a syntax check, leaving whitespace to editorconfig-checker and allowing GitHub Actions' `on:` key
- [.rumdl.toml](.rumdl.toml) disables MD013 (line length) and MD029 (ordered list item prefix)

`ref: main` means changes to the shared config reach every repository at the next refetch, without a pull request in each repository. Keep changes to `lefthook-shared.yml` backward compatible, and announce breaking ones. If a fetch fails, lefthook warns and keeps using the last fetched config.

### Included jobs

| Job | Target |
| --- | --- |
| editorconfig-checker | Final newlines and trailing whitespace in all files (`.editorconfig`) |
| yamllint | YAML syntax (`.yamllint.yml`) |
| actionlint | `.github/workflows/*.yml` |
| gitleaks | Secret detection |
| rumdl | Markdown, fixing what it can in place (`--fix`) |
| semgrep | Static analysis for security issues and bugs |
| shellcheck | Shell scripts |
| trivy | Dependency vulnerabilities and IaC misconfigurations (Trivy) |
| biome-check | JavaScript / TypeScript |
| ruff-check / ruff-format | Python, including Bandit security rules (`S`) |
| cocogitto | Commit messages (`commit-msg` hook) |

Jobs with a `glob` and no matching staged files are skipped, so unused jobs can stay in place. The jobs that only report problems run in parallel in the `checks` group. The fixers (rumdl, Biome, Ruff) then run one by one, since several of them fix the same files in place.

`fail_on_changes: always` fails the commit when a job modifies files, such as rumdl, Biome, or Ruff fixing them in place. Review the fixes, stage them, and commit again. editorconfig-checker only reports problems and does not fix them.

### Tools

Repositories need lefthook and the tools for the jobs that match their files installed, for example by pinning them in their `mise.toml`:

```toml
[tools]
# Runtimes
python = "3.14.10"

# Tools
actionlint = "1.7.12"
biome = "2.6.2"
cocogitto = "7.0.0"
editorconfig-checker = "4.0.2"
gitleaks = "8.30.1"
lefthook = "2.2.2"
ruff = "0.17.2"
rumdl = "0.3.2"
semgrep = "1.182.1"
shellcheck = "0.11.0"
trivy = "0.76.1"
yamllint = "1.38.0"
```

`cocogitto`, `editorconfig-checker`, `gitleaks`, `semgrep`, and `trivy` run on every commit, so every repository needs them. `semgrep` and `yamllint` are installed into Python virtual environments, so pin `python` too; the [CI workflow](#github-actions) does not set up Python separately. `actionlint`, `biome`, `ruff`, `rumdl`, `shellcheck`, and `yamllint` are needed only in repositories with workflows, JavaScript / TypeScript files, Python files, Markdown files, shell scripts, or YAML files. The [CI workflow](#github-actions) installs the tools in `mise.toml`, so the same versions run there. Renovate updates the versions in `mise.toml`.

`gitleaks` scans only the staged changes (`--staged`), so it finds nothing in `lefthook run pre-commit --all-files`, including the CI workflow. Run `gitleaks git` to scan the whole history.

### Conventional Commits

The `cocogitto` job runs `cog verify` from [Cocogitto](https://docs.cocogitto.io/) and rejects commit messages that do not follow [Conventional Commits](https://www.conventionalcommits.org/), such as `feat: add login form` or `fix(api): handle empty payload`.

It runs on the `commit-msg` hook, which has two consequences:

1. `lefthook run pre-commit --all-files`, including the CI workflow below, does not check commit messages. Enforcement happens locally at commit time, and `git commit --no-verify` bypasses it
2. To check a message by hand, run `cog verify` on it

```bash
cog verify --file .git/COMMIT_EDITMSG
```

Allowed types default to `build`, `chore`, `ci`, `docs`, `feat`, `fix`, `perf`, `refactor`, `revert`, `style`, and `test`. The job passes `--ignore-merge-commits` and `--ignore-fixup-commits`, so merge commits and `fixup!`/`squash!`/`amend!` commits pass as-is. Per repository, adjust the rules in a `cog.toml` at the repository root:

- `[commit_types]` adds types, such as `hotfix = { changelog_title = "Hotfixes" }`
- `scopes = ["api", "client"]` restricts the allowed scopes
- Override the job's `run` in `lefthook-local.yml` without `--ignore-merge-commits` or `--ignore-fixup-commits` to also reject those commits

### Semgrep

`semgrep` scans staged files with the [`p/default`](https://semgrep.dev/p/default) ruleset and fails the commit on any finding (`--error`). Some details to keep in mind:

- The ruleset is downloaded from the Semgrep Registry on every run, so the job needs network access and fails offline
- `--skip-unknown-extensions` skips files whose extension Semgrep does not recognize, and `--disable-version-check` and `--quiet` keep the output short. Keep them when overriding `run`
- The rulesets are named explicitly rather than using `--config auto`, which requires sending metrics to Semgrep
- To ignore a false positive, add a `# nosemgrep: <rule-id>` comment on the reported line

Useful changes per repository:

- Add rulesets for the languages in use, e.g. `--config p/default --config p/python`
- Point `--config` at a rules file in the repository, e.g. `.semgrep.yml`, to run without network access
- Add `.semgrepignore` to skip paths such as generated files or test fixtures

### Trivy

The `trivy` job runs `trivy fs` with the [Trivy](https://trivy.dev/) found on `PATH` and fails the commit on any HIGH or CRITICAL finding. GitHub-hosted runners do not have Trivy preinstalled, so repositories that call the [CI workflow](#github-actions) must pin it in `mise.toml`. Some details to keep in mind:

- The job scans the whole repository rather than the staged files, so it runs once on every commit
- The vulnerability database is downloaded on the first run and when it goes stale, so the job needs network access. It is cached in Trivy's default cache directory (`~/.cache/trivy`), outside the repository. The [CI workflow](#github-actions) caches that directory with `actions/cache`, keyed by date, so it downloads the database at most once a day
- `--exit-code 1` makes findings fail the commit, since `trivy fs` exits with 0 by default
- `--scanners vuln,misconfig` skips Trivy's secret scanner, since gitleaks already covers secrets
- Trivy does not read `.gitignore`, so `--skip-dirs "**/node_modules"` and `--skip-dirs "**/.venv"` keep it out of installed dependencies at any depth. Lockfiles and `requirements.txt` are still scanned
- The last arg must be the path to scan (`.`)

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

lefthook merges configs in this order: `lefthook.yml`, `remotes`, then `lefthook-local.yml`. The shared config therefore overrides the repository's `lefthook.yml`, so put adjustments to shared jobs in a `lefthook-local.yml` committed to the repository. Jobs are merged by `name`, so list only the options to change. The jobs that only report problems (editorconfig-checker, yamllint, actionlint, gitleaks, semgrep, shellcheck, trivy) run in parallel inside the `checks` group, so nest overrides of those under the group. A top-level job with the same name would run as an extra job instead:

```yaml
pre-commit:
  jobs:
    # Skip generated files
    - name: rumdl
      exclude:
        - "src/content/*"
    # Add a ruleset for the language in use
    - name: checks
      group:
        jobs:
          - name: semgrep
            run: semgrep --config p/default --config p/python --error --skip-unknown-extensions --disable-version-check --quiet {staged_files}
```

- Allow custom YAML tags, such as in CloudFormation templates, by disabling the matching rules in the repository's `.yamllint.yml`
- Add jobs that only one repository needs to its `lefthook.yml` or `lefthook-local.yml`
- Skip a job for one commit with `LEFTHOOK_EXCLUDE=<job> git commit`, or every job with `LEFTHOOK=0`

### GitHub Actions

[.github/workflows/lefthook.yml](.github/workflows/lefthook.yml) is a reusable workflow that runs every `pre-commit` job against all files with `lefthook run pre-commit --all-files`. It sets up [mise](https://mise.jdx.dev/) and installs the tools pinned in the calling repository's `mise.toml`, including lefthook, so callers must have a `mise.toml`. `fail_on_changes` fails the run when a job would fix a file, and prints the diff. Call it from each repository with `.github/workflows/lefthook.yml`:

```yaml
name: "repo - Lefthook"

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
  lefthook:
    uses: iwstkhr/shared-config/.github/workflows/lefthook.yml@main
```

To get a Slack message when the run fails, pass the Slack secrets described in [Slack notifications](#slack-notifications). The workflow calls [slack-notify.yml](.github/workflows/slack-notify.yml) only on failure, and skips it when either secret is missing, such as in pull requests from forks:

```yaml
jobs:
  lefthook:
    uses: iwstkhr/shared-config/.github/workflows/lefthook.yml@main
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
| Major updates | Add the `breaking-change` label alongside `dependencies` and request review from `iwstkhr` when the PR is created (no automerge) |
| Minor / patch updates | Group as `non-major dependencies` and automerge |
| GitHub Actions pin / digest updates | Automerge |
| Python version in `mise.toml` | Keep below 3.15 (`allowedVersions: "<3.15"`), because semgrep fails on Python 3.15. Remove the rule once semgrep supports 3.15 |

### Validation

[.github/workflows/renovate-validate.yml](.github/workflows/renovate-validate.yml) validates `renovate-preset.json` and `renovate.json` with the [Renovate config validator](https://docs.renovatebot.com/config-validation/) when either file changes. To run the same check locally:

```bash
npx --yes --package renovate -- renovate-config-validator --strict --no-global renovate-preset.json renovate.json
```

`--no-global` validates the files as repository config rather than self-hosted global config, and `--strict` also fails when an option needs migration. The validator does not fetch presets referenced in `extends`, so a wrong preset name only shows up on the Dependency Dashboard after Renovate runs.
