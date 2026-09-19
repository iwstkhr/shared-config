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

Hooks with no matching files are skipped, so unused hooks can stay in place.

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
