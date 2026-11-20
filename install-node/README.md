# `install-node` — Install Node.js and Project Dependencies

A composite action that sets up Node.js/npm and installs dependencies with `npm ci` when `package-lock.json` exists, falling back to `npm install` from `package.json` when it doesn't.

## Usage

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v4

  - name: Install dependencies
    id: deps
    uses: hashemallaham963/gh-actions/install-node@main

  - name: Show versions
    run: echo "${{ steps.deps.outputs.node_version }} / ${{ steps.deps.outputs.npm_version }}"
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `node-version` | No | `20` | Node.js version passed to `actions/setup-node@v4` (e.g. `18`, `20`, `lts/*`) |
| `working-directory` | No | `.` | Directory containing `package.json`, relative to the workspace root |
| `cache` | No | `true` | Enable npm caching. Only applied when `package-lock.json` exists |

## Outputs

| Output | Description |
|--------|-------------|
| `status` | Result of the operation: `success` or `failed` |
| `mode` | `ci` when `package-lock.json` was used, `install` for the `package.json` fallback |
| `node_version` | Installed Node.js version (e.g. `v20.11.1`) |
| `npm_version` | Installed npm version (e.g. `10.2.4`) |
| `cache_enabled` | `true` when npm caching was applied, `false` otherwise |

## Behavior

| Condition | Command used |
|-----------|--------------|
| `package-lock.json` found | `npm ci` |
| No `package-lock.json`, `package.json` found | `npm install` |
| No `package.json` | fails with `status=failed` |

## Examples

### Different Node version

```yaml
- uses: hashemallaham963/gh-actions/install-node@main
  with:
    node-version: '18'
```

### Monorepo package directory

```yaml
- uses: hashemallaham963/gh-actions/install-node@main
  with:
    working-directory: packages/web
```

### Use the resolved install mode

```yaml
- name: Install
  id: deps
  uses: hashemallaham963/gh-actions/install-node@main

- name: Verify
  run: |
    if [ "${{ steps.deps.outputs.mode }}" = "install" ]; then
      echo "⚠️ No lockfile — consider committing package-lock.json"
    fi
```

## Notes

- `actions/checkout@v4` must run before this action so `package.json` is available.
- `npm ci` requires `package-lock.json` to be in sync with `package.json`; if they drift, the install fails instead of silently resolving new versions.
- The `npm install` fallback will create a `package-lock.json` in the working directory. Commit it to get reproducible installs on subsequent runs.
- npm caching uses `actions/setup-node@v4`'s built-in cache, keyed on the resolved `package-lock.json`. It is enabled by default and skipped automatically when there is no lockfile, when `working-directory` does not contain one, or when `cache: false` is passed. The cache key also includes the Node.js version.
- The `npm install` fallback does not cache because there is no lockfile to key on. Once you commit the generated `package-lock.json`, subsequent runs get caching.
- Inputs are passed through `env:` and the working directory is quoted in the shell script.
