# `get-version` — Extract Version from Release

A GitHub composite action that extracts a semantic version from the current Git context. Supports two modes to handle different release workflows.

## Usage and the required permissions

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: read  # REQUIRED for hashemallaham963/gh-actions/get-version
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # required to fetches full tag history for hashemallaham963/gh-actions/get-version

      - name: Get version
        id: ver
        uses: hashemallaham963/gh-actions/get-version@main
        with:
          mode: advanced   # optional, default: advanced

      - run: echo "Releasing v${{ steps.ver.outputs.version }}"
```

Without this permission, the action will fail with a clear error message instead of silently falling back to a timestamp version.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `mode` | No | `advanced` | Extraction mode: `simple` or `advanced` |

## Outputs

| Output | Description |
|--------|-------------|
| `version` | Extracted semantic version (`MAJOR.MINOR.PATCH`). Also set as `VERSION` env var. |

## Modes

### `simple`

Picks the **highest semver tag** on the current commit (`sort -V | tail -1`), then falls back through:

1. Highest semver tag on commit
2. Release branch name (`release/x.y.z`, `rel-x.y.z`)
3. Latest GitHub release tag
4. Timestamp fallback (`0.0.YYYYMMDD`)

Use when your release workflow creates one tag per commit.

### `advanced` _(default)_

Prefers a tag **whose name matches the triggering branch exactly**, then falls back to the highest semver tag. Solves the case where an older tag and a newer tag both point to the same commit SHA.

Fallback chain:

1. Tag matching branch name exactly
2. Highest semver tag on commit
3. Release branch name
4. Latest GitHub release tag
5. `package.json` version field
6. Timestamp fallback (`0.0.YYYYMMDD`)

## Version normalization

All modes strip common prefixes (`v`, `release-v`, `release-`, `rel-v`, `rel-`) and normalize to bare `MAJOR.MINOR.PATCH`. If the result does not match the semver pattern, it falls back to `0.0.1`.

## Examples

**Simple mode:**

```yaml
- uses: ./.github/actions/get-version
  with:
    mode: simple
```

**Full release pipeline:**

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: read  # REQUIRED for hashemallaham963/gh-actions/get-version
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # REQUIRED

      - name: Extract version
        id: ver
        uses: hashemallaham963/gh-actions/get-version@main
        with:
          mode: advanced

      - name: Build and publish
        run: make build VERSION=${{ steps.ver.outputs.version }}
```

> **Note:** `fetch-depth: 0` is required on the checkout step. Without it, `git fetch --tags` may not resolve tags correctly on shallow clones.
>
> **Note:** The workflow job must have `permissions: contents: read` for the GitHub API calls to work. See [Required Permissions](#required-permissions) above.
