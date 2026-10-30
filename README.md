# shared-config

Configuration templates shared across personal repositories.

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
