# `generate-changelog` — Generate a Changelog

A minimal GitHub composite action that turns a range of Git commits into a markdown bullet list.

Designed for **tag-driven releases**: with no inputs, it uses the previous semver tag and the tag that triggered the workflow. On a `release: created` event, it also uploads the changelog as a release asset named `CHANGELOG.md` and appends it to the release notes under a `changelog:` heading.

## Usage

```yaml
on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Generate changelog
        id: notes
        uses: hashemallaham963/gh-actions/generate-changelog@main

      - name: Use it
        run: echo "${{ steps.notes.outputs.changelog }}"
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `from` | No | *(see behavior)* | Starting tag/ref. Empty with an empty `to` means the previous semver tag |
| `to` | No | *(see behavior)* | Ending tag/ref. Empty with an empty `from` means the triggering tag, or `HEAD` off a tag event |

## Behavior

| `from` | `to` | Range used |
|--------|------|------------|
| empty | empty | previous semver tag `..` triggering tag (`HEAD` if not a tag event) |
| set | empty | `from..HEAD` |
| empty | set | all history up to `to` |
| set | set | `from..to` |

In the default mode, the previous tag is the highest version tag (`1.2.3` or `v1.2.3`) excluding the triggering tag, so re-running a workflow never includes the current tag as its own starting point.

## Outputs

| Output | Description |
|--------|-------------|
| `changelog` | Markdown bullet list: `- <commit subject> (<short SHA>)` |
| `asset_url` | Download URL of the uploaded `CHANGELOG.md`, or empty when not on a `release: created` event |
| `status` | Result of the operation: `success` or `failed` |

## Examples

### Release workflow with asset upload

```yaml
on:
  release:
    types: [created]

jobs:
  notes:
    runs-on: ubuntu-latest
    permissions:
      contents: write   # required to upload release assets
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - id: notes
        uses: hashemallaham963/gh-actions/generate-changelog@main

      - run: echo "Asset: ${{ steps.notes.outputs.asset_url }}"
```

The action detects `release` + `created` itself; no input is needed to enable the upload.

### Explicit range

```yaml
- uses: hashemallaham963/gh-actions/generate-changelog@main
  with:
    from: v1.0.0
    to: v1.1.0
```

Produces something like:

```markdown
- fix: handle empty response (a1b2c3d)
- feat: add dark mode (f9e8d7c)
```

### Feed the changelog into a Slack notification

```yaml
- name: Generate changelog
  id: notes
  uses: hashemallaham963/gh-actions/generate-changelog@main

- name: Notify
  uses: hashemallaham963/gh-actions/send-slack-notification@main
  with:
    status: '✅ Released'
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
    branch: ${{ github.ref_name }}
    commit_message: ${{ steps.notes.outputs.changelog }}
    author_name: ${{ github.actor }}
```

## Notes

- Requires `actions/checkout@v4` **before** this action. `fetch-depth: 0` is required for tag mode so the previous tag is present on the runner.
- Semver tags are selected in both forms: `1.2.3` and `v1.2.3`. Non-semver tags are ignored.
- If no previous semver tag exists, the action falls back to the full history up to `to` and logs `Tag mode: from '' to '...'`.
- Off a tag event (branch push, `workflow_dispatch`), `to` falls back to `HEAD` and `from` is the highest semver tag.
- Merge commits are excluded (`git log --no-merges`), so the list reflects actual work.
- Commit subjects are used verbatim. Conventional commits like `feat: ...` / `fix: ...` produce the most readable changelog.
- Asset upload and release-note updates need `permissions: contents: write` and `curl` + `jq` on the runner (both present on `ubuntu-latest`). They only fire on `release: created`; if you trigger on `published` instead, nothing is uploaded or appended.
- Release notes are appended to, not overwritten: existing text is kept and followed by a blank line, `changelog:`, then the generated list. Re-running the workflow will append the section again.
- If no commits fall in the range, `changelog` is empty but `status` is still `success`.
- Inputs are passed through `env:` and the range is quoted, so branch names and refs are safe to interpolate.
