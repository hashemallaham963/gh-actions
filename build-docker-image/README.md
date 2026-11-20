# `build-docker-image` — Build an Image with Docker Buildx

A composite action that creates (or reuses) a Buildx builder and runs `docker buildx build` with fully configurable working directory, context, Dockerfile, builder, driver, platforms, tags, load/push, and output options.

## Usage

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v4

  - name: Set up Docker
    uses: hashemallaham963/gh-actions/install-docker@main

  - name: Build image
    id: image
    uses: hashemallaham963/gh-actions/build-docker-image@main
    with:
      working-directory: apps/controller-dashboard
      platforms: ${{ matrix.target_arch }}
      tags: controller_dashboard_express_api_${{ matrix.target_lower }}:latest

  - name: Show result
    run: echo "${{ steps.image.outputs.image }} / ${{ steps.image.outputs.image_id }}"
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `working-directory` | No | `.` | Directory entered before running the build |
| `context` | No | `.` | Build context path passed to buildx, relative to `working-directory` |
| `dockerfile` | No | *(empty)* | Dockerfile path relative to the build context; empty uses `Dockerfile` |
| `builder-name` | No | `gh-actions-builder` | Name of the Buildx builder to create or reuse |
| `driver` | No | `docker-container` | Driver used when creating the builder |
| `platforms` | No | `linux/amd64` | Comma-separated target platforms, e.g. `linux/amd64,linux/arm64` |
| `tags` | **Yes** | — | Image tag(s), one per line; the first becomes the `image` output |
| `build-args` | No | *(empty)* | Build arguments, one `KEY=VALUE` per line |
| `target` | No | *(empty)* | Target build stage in the Dockerfile |
| `load` | No | `true` | Add `--load` to import a single-platform result into the local daemon |
| `push` | No | `false` | Add `--push` to send the result to the registry |
| `output` | No | `type=image` | Buildx `--output` value; empty omits the flag |
| `no-cache` | No | `false` | Add `--no-cache` |
| `progress` | No | `auto` | Buildx progress mode (`auto`, `plain`, `quiet`, `none`, `rawjson`, `tty`) |

## Outputs

| Output | Description |
|--------|-------------|
| `status` | `success` or `failed` |
| `image` | First tag from `tags` |
| `image_id` | Local image ID when `load: true`; empty otherwise |
| `platforms` | Platforms used for the build |
| `builder_name` | Builder used for the build |

## Behavior

1. Validates `working-directory`, `tags`, the `load`/`push` combination, and the presence of `docker`.
2. Reuses the named Buildx builder if it exists; otherwise creates it with `docker buildx create --name <builder-name> --driver <driver>`.
3. Enters `working-directory` and verifies the build `context` exists.
4. Builds the list of buildx flags from the inputs (`--platform`, `--tag`, `--build-arg`, `--file`, `--target`, `--no-cache`, `--progress`, `--load`, `--push`, `--output`) and runs `docker buildx build` against `context`.
5. Reads the local image ID when `load` is enabled and writes all outputs, including `status=failed` on error.

## Examples

### Reproduce the original matrix build

```yaml
- uses: hashemallaham963/gh-actions/build-docker-image@main
  with:
    working-directory: ${{ github.workspace }}/controller_dashboard_express_api
    builder-name: hashimBuilder
    driver: docker-container
    platforms: ${{ matrix.target_arch }}
    tags: controller_dashboard_express_api_${{ matrix.target_lower }}:latest
    load: true
    output: type=image
```

### Multiple tags and build args

```yaml
- uses: hashemallaham963/gh-actions/build-docker-image@main
  with:
    tags: |
      my-app:latest
      ghcr.io/${{ github.repository }}:${{ github.sha }}
    build-args: |
      NODE_ENV=production
      VERSION=1.2.3
    target: release
```

### Multi-platform image pushed to a registry

```yaml
- uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- uses: hashemallaham963/gh-actions/build-docker-image@main
  with:
    platforms: linux/amd64,linux/arm64
    tags: ghcr.io/${{ github.repository }}:latest
    load: false
    push: true
    output: type=image
```

## Notes

- Run `install-docker` first (or ensure Docker with Buildx and any needed QEMU platforms are already available). This action creates the builder but does not install Docker or register emulation platforms.
- `--load` imports a **single-platform** result into the local image store. For multi-platform builds, set `load: false` and either `push: true` or an explicit `output` such as `type=oci,dest=image.tar`.
- `load` and `push` are mutually exclusive; the action fails fast if both are `true`.
- `type=image` with `load: true` is valid: buildx adds a docker exporter so the image is also available locally, which lets `image_id` be read back.
- The builder is reused when it already exists, so the `driver` input only applies to the first build that creates it. Use a different `builder-name` to force a new driver.
- All input values are passed through `env:` and the build flag array is built with proper quoting, so values containing spaces are handled safely. The log line is a summary only, so `--build-arg` values are not printed.
- A prior `actions/checkout@v4` is required when the build context lives in the repository.
