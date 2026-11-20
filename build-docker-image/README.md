# `build-docker-image` — Build an Image with Docker Buildx

A small composite action that runs `docker buildx build` with configurable working directory, context, platforms, tags, optional registry cache, load, and output. Builder setup is handled by `install-docker`, so this action only builds.

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
| `cache-ref` | No | *(empty)* | Registry cache reference (e.g. `ghcr.io/org/app:buildcache`); enables cache import/export |
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
3. When `cache-ref` is set, adds `--cache-from type=registry,ref=<cache-ref>` and `--cache-to type=registry,ref=<cache-ref>,mode=max`.
4. Writes `image` (the first tag) and `status`, including `status=failed` on error.

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

### Build with GHCR registry cache

```yaml
permissions:
  contents: read
  packages: write

steps:
  - uses: actions/checkout@v4
  - uses: hashemallaham963/gh-actions/install-docker@main
  - uses: docker/login-action@v3
    with:
      registry: ghcr.io
      username: ${{ github.actor }}
      password: ${{ secrets.GITHUB_TOKEN }}

  - uses: hashemallaham963/gh-actions/build-docker-image@main
    with:
      tags: ghcr.io/${{ github.repository }}:latest
      cache-ref: ghcr.io/${{ github.repository }}:buildcache
```

Use a cache tag that is separate from the image tags.

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
- `cache-ref` uses the BuildKit **registry** cache backend. For GHCR, grant `packages: write`, log in to `ghcr.io`, and use a distinct cache tag such as `:buildcache`. The cache is pushed even when `load: true`.
- `mode=max` exports cache layers from all stages, not just the final image. The cache manifest can be larger than the image manifest.
- GHCR image names must be lowercase. If your repository name has uppercase letters, use a lowercased cache ref instead of `${{ github.repository }}` directly.
- All inputs are passed through `env:` and the build flag array is built with proper quoting. The log line is a summary only, so image references are not expanded into a full command trace.
- A prior `actions/checkout@v4` is required when the build context lives in the repository.
