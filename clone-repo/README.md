# `clone-repo` — Clone a Git Repository

A GitHub composite action that clones a Git repository into the current working directory on the runner. Supports authenticated clones via a personal access token, configurable branch, and shallow clone depth.

## Usage

```yaml
steps:
  - name: Clone repository
    id: clone
    uses: hashemallaham963/gh-actions/clone-repo@main
    with:
      repository_url: https://github.com/org/repo.git
      token: ${{ secrets.MY_PAT }}
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `repository_url` | **Yes** | — | Full HTTPS URL of the repository to clone |
| `token` | No | *(dummy — allows public repos)* | Personal access token (PAT) or `${{ secrets.GITHUB_TOKEN }}` for private repos |
| `branch` | No | `main` | Branch to clone |
| `depth` | No | `1` | Shallow clone depth. Set to `0` for a full clone |

## Outputs

| Output | Description |
|--------|-------------|
| `status` | Result of the clone operation: `success` or `failed` |

## Examples

### Public repository

No token needed for public repos:

```yaml
- uses: hashemallaham963/gh-actions/clone-repo@main
  with:
    repository_url: https://github.com/org/public-repo.git
```

### Private repository

```yaml
- uses: hashemallaham963/gh-actions/clone-repo@main
  with:
    repository_url: https://github.com/org/private-repo.git
    token: ${{ secrets.MY_PAT }}
```

### Specific branch and full clone

```yaml
- uses: hashemallaham963/gh-actions/clone-repo@main
  with:
    repository_url: https://github.com/org/repo.git
    token: ${{ secrets.MY_PAT }}
    branch: develop
    depth: 0
```

### Check the output status

```yaml
- name: Clone
  id: clone
  uses: hashemallaham963/gh-actions/clone-repo@main
  with:
    repository_url: https://github.com/org/repo.git
    token: ${{ secrets.MY_PAT }}

- name: Verify
  run: echo "Clone status: ${{ steps.clone.outputs.status }}"
```

## Notes

- The action installs `git` automatically if it is not present on the runner.
- The token is injected into the clone URL as `x-access-token` and is never logged.
- The clone lands in the **current working directory** of the runner — set `working-directory` on the step or a prior `cd` if you need a specific path.
- `depth: 1` (default) is a shallow clone. If you need the full history or tags, set `depth: 0`.
