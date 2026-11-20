# `publish-docker-image` — Push a Built Image to a Registry

A small composite action that pushes one already-built image to its registry and reports the digest. Credentials are optional; omit them when the runner is already logged in.

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
      tags: ghcr.io/${{ github.repository }}:latest

  - name: Publish image
    id: publish
    uses: hashemallaham963/gh-actions/publish-docker-image@main
    with:
      image: ${{ steps.build.outputs.image }}
      registry: ghcr.io
      username: ${{ github.actor }}
      password: ${{ secrets.GITHUB_TOKEN }}

  - name: Show digest
    run: echo "${{ steps.publish.outputs.digest }}"
```

`image` is the fully qualified local tag that should be pushed — for example `ghcr.io/org/app:latest`. It includes the tag; there is no separate tag input.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `image` | **Yes** | — | Fully qualified local image reference to push (e.g. `ghcr.io/org/app:latest`) |
| `registry` | No | *(empty)* | Registry host to log in to; required when credentials are supplied |
| `username` | No | *(empty)* | Registry username; supply together with `registry` and `password` |
| `password` | No | *(empty)* | Registry password/token, passed via `docker login --password-stdin` |

## Outputs

| Output | Description |
|--------|-------------|
| `status` | `success` or `failed` |
| `digest` | `sha256:...` from the push output, or empty when unavailable |

## Behavior

1. Validates `image`, the presence of `docker`, and that credentials are either fully supplied (`registry`, `username`, `password`) or fully omitted.
2. When credentials are supplied, logs in to the explicit `registry` via `docker login --password-stdin`. Otherwise the runner's existing Docker login is reused.
3. Runs `docker push "$IMAGE"` and captures the last `sha256:...` value as `digest`.
4. Writes `status=success`, or `status=failed` on error.

## Examples

### Reuse an existing login

```yaml
- uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- uses: hashemallaham963/gh-actions/publish-docker-image@main
  with:
    image: ghcr.io/${{ github.repository }}:latest
```

### AWS ECR with the official login action

`aws-actions/amazon-ecr-login@v2` already logs Docker in, so no credentials are passed here:

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-region: us-east-1
    role-to-assume: arn:aws:iam::123456789012:role/github-actions

- uses: aws-actions/amazon-ecr-login@v2
  id: ecr

- uses: hashemallaham963/gh-actions/build-docker-image@main
  id: build
  with:
    tags: ${{ steps.ecr.outputs.registry }}/my-app:latest

- uses: hashemallaham963/gh-actions/publish-docker-image@main
  with:
    image: ${{ steps.build.outputs.image }}
```

### Log in explicitly with all required information

```yaml
- uses: hashemallaham963/gh-actions/publish-docker-image@main
  with:
    image: registry.example.com/team/my-app:v1.2.3
    registry: registry.example.com
    username: ${{ secrets.REGISTRY_USER }}
    password: ${{ secrets.REGISTRY_PASSWORD }}
```

## Notes

- The image must already exist in the local Docker daemon. Use `build-docker-image` with `load: true` and a fully qualified `tags` value.
- Credentials are never echoed: the password is piped to `docker login --password-stdin`, and log lines contain only the image reference.
- Login is optional. Pass all three of `registry`, `username`, and `password` to log in here; omit all three to reuse an existing login such as `docker/login-action@v3` or `aws-actions/amazon-ecr-login@v2`.
- The registry is never inferred from `image`, so an ECR URI, a private mirror, or a registry with a port all work as long as `registry` matches where you want to log in.
- Partial credentials fail fast: `username`/`password` must be paired, and both require `registry`.
- `digest` comes from the push output; registries that do not print one yield an empty string. It is informational, not a guaranteed attestation.
- Requires Docker with Buildx installed (run `install-docker` first) and a prior checkout when the image was built from repository sources.
