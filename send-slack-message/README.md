# `send-slack-message` — Send a Slack Message

A GitHub composite action that sends a message to a Slack channel via an [Incoming Webhook](https://api.slack.com/messaging/webhooks). Supports a custom bot username.

## Usage

```yaml
steps:
  - name: Notify Slack
    uses: hashemallaham963/gh-actions/send-slack-message@main
    with:
      webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
      message: "Deployment completed successfully!"
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `webhook-url` | **Yes** | — | Slack incoming webhook URL |
| `message` | No | `Workflow completed!` | Message text to send |
| `username` | No | — | Custom display name for the bot in Slack |

## Outputs

| Output | Description |
|--------|-------------|
| `status` | Result of the operation: `success` or `failed` |

## Examples

### Notify on deployment

```yaml
- name: Notify Slack
  uses: hashemallaham963/gh-actions/send-slack-message@main
  with:
    webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
    message: "🚀 ${{ github.repository }} deployed version ${{ needs.version.outputs.version }}"
    username: "Deploy Bot"
```

### Notify on failure

```yaml
- name: Notify on failure
  if: failure()
  uses: hashemallaham963/gh-actions/send-slack-message@main
  with:
    webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
    message: "❌ Workflow failed in ${{ github.repository }} on branch ${{ github.ref_name }}"
    username: "CI Bot"
```

### Rich message with Slack markdown

```yaml
- name: Notify team
  uses: hashemallaham963/gh-actions/send-slack-message@main
  with:
    webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
    username: "Release Bot"
    message: |
      *🚀 Deployment Report*

      • Status: ✅ *Success*
      • Repository: `${{ github.repository }}`
      • Branch: `${{ github.ref_name }}`
      • Commit: `${{ github.sha }}`

      <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Workflow>
```

### Check the output status

```yaml
- name: Send notification
  id: notify
  uses: hashemallaham963/gh-actions/send-slack-message@main
  with:
    webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
    message: "Build complete."

- name: Check result
  run: echo "Slack status: ${{ steps.notify.outputs.status }}"
```

## Setting up a Slack Incoming Webhook

1. Go to [api.slack.com/apps](https://api.slack.com/apps) and create (or select) an app.
2. Under **Features**, enable **Incoming Webhooks**.
3. Click **Add New Webhook to Workspace** and select a channel.
4. Copy the generated webhook URL.
5. Add it as a secret in your repository: **Settings → Secrets and variables → Actions → New repository secret**, name it `SLACK_WEBHOOK_URL`.

## Notes

- The action installs Python 3 and the `requests` library at runtime.
- The webhook URL is passed via environment variable and is never logged.
- The action exits with a non-zero code if Slack returns a non-200 response, which will fail the workflow step.
- Message text supports [Slack's mrkdwn formatting](https://api.slack.com/reference/surfaces/formatting) — use `*bold*`, `` `code` ``, and `<url|label>` links.
