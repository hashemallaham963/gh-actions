# `build-docker-image` — Build an Image with Docker Buildx

A small composite action that runs `docker buildx build` with configurable working directory, context, platforms, tags, load, and output. Builder setup is handled by `install-docker`, so this action only builds.

## Usage

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v4

  - name: Set up Docker
    uses: hashemallaham963/gh-actions/install-docker@main

  - name: Build image
    id: build
    uses: hashemallaham963/gh-actions/build-docker-image@main
    with:
      working-directory: apps/api
      platforms: ${{ matrix.target_arch }}
      tags: ghcr.io/${{ github.repository }}:latest

  - name: Show result
    run: echo "${{ steps.build.outputs.image }}"
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `working-directory` | No | `.` | Directory entered before the build |
| `context` | No | `.` | Build context path, relative to `working-directory` |
| `platforms` | No | `linux/amd64` | Comma-separated target platforms |
| `tags` | **Yes** | — | Image tag(s), one per line; the first becomes the `image` output |
| `load` | No | `true` | Add `--load` so the result is available in the local daemon |
| `output` | No | `type=image` | Buildx `--output` value; empty omits the flag |

## Outputs

| Output | Description |
|--------|-------------|
| `status` | `success` or `failed` |
| `image` | First tag from `tags` |

## Behavior

1. Validates `working-directory`, `tags`, the build `context`, and the presence of `docker`.
2. Enters `working-directory` and assembles `docker buildx build` from the input values.
3. Writes `image` (the first tag) and `status`, including `status=failed` on error.

Use fully qualified tags such as `ghcr.io/org/app:latest` if `publish-docker-image` will push the result. A local-only tag like `my-app:latest` cannot be pushed to GHCR without retagging.

## Examples

### Build for publishing

```yaml
- uses: hashemallaham963/gh-actions/build-docker-image@main
  with:
    tags: |
      ghcr.io/${{ github.repository }}:latest
      ghcr.io/${{ github.repository }}:${{ github.sha }}
    load: true
```

### Multi-platform image pushed by the build itself

```yaml
- uses: hashemallaham963/gh-actions/build-docker-image@main
  with:
    platforms: linux/amd64,linux/arm64
    tags: ghcr.io/${{ github.repository }}:latest
    load: false
    output: type=registry
```

## Notes

- Run `install-docker` first. It installs Docker, registers QEMU if needed, and creates the Buildx builder this action uses.
- `--load` imports a **single-platform** result into the local image store. For multi-platform builds, set `load: false` and use an output such as `type=registry` with a prior registry login.
- `type=image` together with `load: true` is valid: buildx adds a docker exporter so the image is also available locally.
- All inputs are passed through `env:` and the build flag array is built with proper quoting. The log line is a summary only, so image references are not expanded into a full command trace.
- A prior `actions/checkout@v4` is required when the build context lives in the repository.
