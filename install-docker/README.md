# `install-docker` — Install Docker and Enable BuildKit

A composite action that installs Docker CE with the official `docker/setup-docker-action`, optionally registers QEMU for cross-platform builds, sets up Docker Buildx with BuildKit, and exports `DOCKER_BUILDKIT=1` for subsequent steps.

## Usage

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v4

  - name: Install Docker and BuildKit
    id: docker
    uses: hashemallaham963/gh-actions/install-docker@main

  - name: Build image
    run: docker build -t my-app:latest .
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `docker-version` | No | `latest` | Docker CE version passed to `docker/setup-docker-action` (e.g. `v24.0.9`) |
| `driver` | No | `docker-container` | Buildx driver: `docker`, `docker-container`, `kubernetes`, or `remote` |
| `builder-name` | No | `gh-actions-builder` | Name of the Buildx builder to create or reuse |
| `qemu` | No | `true` | Set to `false` to skip QEMU registration entirely |
| `platforms` | No | `linux/amd64,linux/arm64` | Comma-separated platforms QEMU should register (ignored when `qemu: false`) |

## Outputs

| Output | Description |
|--------|-------------|
| `status` | Result of the operation: `success` or `failed` |
| `docker_version` | Installed Docker version (e.g. `Docker version 24.0.7, build ...`) |
| `buildx_version` | Installed Buildx version (e.g. `github.com/docker/buildx v0.13.2 ...`) |
| `builder_name` | Name of the active Buildx builder |
| `qemu_platforms` | Platforms registered with QEMU; empty when `qemu: false` |

## Behavior

1. Runs the official `docker/setup-docker-action@v5` to install Docker CE at `docker-version` and start its daemon.
2. Unless `qemu: false`, runs `docker/setup-qemu-action@v3` to register the requested platforms for emulation. The step is skipped entirely when disabled.
3. Runs `docker/setup-buildx-action@v3` with `install: true`, which installs the Buildx plugin and creates/selects the named builder.
4. A final `if: always()` step reports the result, reads `docker --version` and `docker buildx version`, and exports `DOCKER_BUILDKIT=1` to `$GITHUB_ENV` so later steps use the BuildKit backend.

## Examples

### Use the default builder (recommended)

```yaml
- uses: hashemallaham963/gh-actions/install-docker@main
```

### Pin a Docker CE version

```yaml
- uses: hashemallaham963/gh-actions/install-docker@main
  with:
    docker-version: v24.0.9
```

### Use the daemon's built-in driver

```yaml
- uses: hashemallaham963/gh-actions/install-docker@main
  with:
    driver: docker
```

### Native builds only (skip QEMU)

```yaml
- uses: hashemallaham963/gh-actions/install-docker@main
  with:
    qemu: false
```

### Cross-platform multi-arch build

```yaml
- id: docker
  uses: hashemallaham963/gh-actions/install-docker@main
  with:
    platforms: linux/amd64,linux/arm64,linux/arm/v7

- run: |
    docker buildx build \
      --platform linux/amd64,linux/arm64,linux/arm/v7 \
      --tag my-app:latest .
```

### Combine with a registry login

```yaml
- id: docker
  uses: hashemallaham963/gh-actions/install-docker@main

- uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- run: docker build --push -t ghcr.io/${{ github.repository }}:latest .
```

## Notes

- All three setup actions (`setup-docker-action@v5`, `setup-qemu-action@v3`, `setup-buildx-action@v3`) are first-party Docker actions; no hand-written install scripts are used.
- GitHub-hosted Ubuntu runners already have Docker running. The official setup action is mainly useful here for pinning a Docker CE version, not because Docker is missing.
- The setup action downloads Docker CE from Docker's releases and starts its own daemon, so no `sudo` or apt access is needed. The runner needs network access to the Docker download endpoints; a GitHub token is used by default for the release lookup.
- `docker_version` is read with `docker --version` in the final step because `setup-docker-action` does not expose a version output.
- QEMU registration runs the `tonistiigi/binfmt` container and requires the runner to allow privileged containers. GitHub-hosted Ubuntu runners support this; hardened self-hosted runners may need an exception.
- Emulation is slower than native builds. Keep `platforms` to only what you actually ship, or set `qemu: false` for native-only pipelines.
- When `qemu: false`, the QEMU step is skipped, `qemu_platforms` is empty, and a skipped QEMU outcome does not fail the action.
- The `docker-container` driver supports advanced BuildKit features (cache export, multi-platform builds via QEMU). Use `driver: docker` if you only need the daemon's built-in builder.
- `DOCKER_BUILDKIT=1` is written to `$GITHUB_ENV`; export-based shell changes would not persist across composite action steps.
- If any step fails, `status=failed` is emitted by the final step via `if: always()`.
