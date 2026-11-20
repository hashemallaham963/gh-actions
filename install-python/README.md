# `install-python` — Install Python and Project Dependencies

A composite action that sets up Python/pip and installs dependencies from `requirements.txt`, falling back to the project metadata (`pyproject.toml` or `setup.py`) when no requirements file exists.

## Usage

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v4

  - name: Install dependencies
    id: deps
    uses: hashemallaham963/gh-actions/install-python@main

  - name: Show versions
    run: echo "${{ steps.deps.outputs.python_version }} / ${{ steps.deps.outputs.pip_version }}"
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `python-version` | No | `3.11` | Python version passed to `actions/setup-python@v5` |
| `working-directory` | No | `.` | Directory containing the dependency file, relative to the workspace root |
| `cache` | No | `true` | Enable pip caching when a dependency file exists |

## Outputs

| Output | Description |
|--------|-------------|
| `status` | Result of the operation: `success` or `failed` |
| `mode` | `requirements`, `pyproject`, `setup.py`, or empty when nothing was found |
| `python_version` | Installed Python version (e.g. `3.11.8`) |
| `pip_version` | Installed pip version |
| `cache_enabled` | `true` when pip caching was applied, `false` otherwise |

## Behavior

Priority order for the dependency source:

| File found | Command used | `mode` |
|------------|--------------|--------|
| `requirements.txt` | `python -m pip install -r requirements.txt` | `requirements` |
| `pyproject.toml` | `python -m pip install .` | `pyproject` |
| `setup.py` | `python -m pip install .` | `setup.py` |
| none | fails with `status=failed` | empty |

## Examples

### Different Python version

```yaml
- uses: hashemallaham963/gh-actions/install-python@main
  with:
    python-version: '3.12'
```

### Project in a subdirectory

```yaml
- uses: hashemallaham963/gh-actions/install-python@main
  with:
    working-directory: services/api
```

### Disable caching

```yaml
- uses: hashemallaham963/gh-actions/install-python@main
  with:
    cache: false
```

### Fail if no dependency file is present

```yaml
- name: Install
  id: deps
  uses: hashemallaham963/gh-actions/install-python@main

- name: Verify
  env:
    STATUS: ${{ steps.deps.outputs.status }}
  run: |
    if [ "$STATUS" != "success" ]; then
      echo "❌ dependency install failed"
      exit 1
    fi
```

## Notes

- `actions/checkout@v4` must run before this action so the dependency file is available.
- `requirements.txt` takes precedence. When both `requirements.txt` and `pyproject.toml` exist, only the requirements file is installed.
- The `pyproject.toml`/`setup.py` path installs the project itself with `pip install .`, so its dependencies come from the project metadata.
- pip caching uses `actions/setup-python@v5`'s built-in cache, keyed on the detected dependency file and Python version. It is skipped when no dependency file exists or when `cache: false` is passed.
- Dependencies are installed into the runner's Python environment. Add a virtual environment step first if your workflow needs isolation.
- Inputs are passed through `env:` and the working directory is quoted in the shell script.
