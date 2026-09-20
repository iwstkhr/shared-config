# shared-config

Configuration templates shared across repositories.

## pre-commit

[.pre-commit-config.yaml](.pre-commit-config.yaml) is the canonical config
for all repositories. pre-commit has no config inheritance, so copy it into
each new repository.

```bash
cp ~/git/shared-config/.pre-commit-config.yaml .
pre-commit install
```

`default_install_hook_types` in the config makes `pre-commit install` set up
both the `pre-commit` and the `commit-msg` hook, so no extra flags are needed.
Repositories that ran `pre-commit install` before the `commit-msg` hook was
added need to run it once more.

After copying, Renovate (`:enablePreCommit`) keeps each repository's `rev`
values up to date.

### Included hooks

| Hook | Target |
| --- | --- |
| check-yaml | YAML syntax |
| end-of-file-fixer / trailing-whitespace | All files |
| double-quote-string-fixer | Python |
| actionlint | `.github/workflows/*.yml` |
| gitleaks | Secret detection |
| markdownlint | Markdown |
| shellcheck | Shell scripts |
| conventional-pre-commit | Commit messages (`commit-msg` stage) |

Hooks with no matching files are skipped, so unused hooks can stay in place.

### Conventional Commits

`conventional-pre-commit` rejects commit messages that do not follow
[Conventional Commits](https://www.conventionalcommits.org/), such as
`feat: add login form` or `fix(api): handle empty payload`.

It runs on the `commit-msg` stage, which has two consequences:

1. `pre-commit run --all-files`, including the CI workflow below, does not
   check commit messages. Enforcement happens locally at commit time, and
   `git commit --no-verify` bypasses it
2. To check a message by hand, pass the file explicitly

```bash
pre-commit run --hook-stage commit-msg --commit-msg-filename .git/COMMIT_EDITMSG
```

Allowed types default to `build`, `chore`, `ci`, `docs`, `feat`, `fix`,
`perf`, `refactor`, `revert`, `style`, and `test`, and `fixup!`/`squash!` and
merge commits pass as-is. Useful `args` per repository:

- `[feat, fix, docs, chore]` narrows the allowed types (`feat` and `fix` are
  always allowed)
- `[--force-scope]` requires a scope, `[--scopes, api,client]` restricts it
- `[--strict]` also rejects `fixup!`/`squash!` and merge commits

### Per-repository additions and adjustments

Add Biome to JS/TS repositories.

```yaml
  - repo: https://github.com/biomejs/pre-commit
    rev: v2.5.14
    hooks:
      - id: biome-check
```

Common adjustments:

- Change the `check-yaml` args to `[--unsafe]` for YAML with custom tags,
  such as CloudFormation templates
- Add `exclude:` for files that cause false positives, such as generated files
  (e.g. `exclude: ^src/content/` for markdownlint)

### GitHub Actions

[.github/workflows/pre-commit.yml](.github/workflows/pre-commit.yml) is a
reusable workflow that runs all hooks against all files. Call it from each
repository with `.github/workflows/pre-commit.yml`:

```yaml
name: pre-commit

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

This repository is private, so other repositories can call the workflow only
after enabling **Settings → Actions → General → Access → Accessible from
repositories owned by the user**.
