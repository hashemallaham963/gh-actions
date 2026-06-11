# Send Slack Message (GitHub Action)

**Usage:**
```yaml
- uses: your-username/my-actions-repo/send-slack-message@v1
  with:
    webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
    message: |
      *🚀 Deployment Report*

      • Status: ✅ *Success*
      • Environment: `production`
      • Branch: `main`
      • Commit: ${{ github.sha }}

      *Test Results:*
      ✅ Unit tests: 120 passed
      ✅ Integration: 45 passed
      ✅ E2E tests: 12 passed

      <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Workflow Details>
    username: "Deployment Bot"
```
