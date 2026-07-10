# `publish-apt` — Publish to APT Repository

A GitHub composite action that uploads a `.deb` package to a [Sonatype Nexus](https://www.sonatype.com/products/nexus-repository) APT repository using `curl`. Supports basic authentication and a `Host` header override for IP-based Nexus deployments.

## Usage

```yaml
steps:
  - name: Publish package
    id: publish
    uses: hashemallaham963/gh-actions/publish-apt@main
    with:
      repository_url: https://nexus.example.com/repository/apt-releases/
      repository_username: ${{ secrets.NEXUS_USERNAME }}
      repository_password: ${{ secrets.NEXUS_PASSWORD }}
      package_path: /path/to/package_1.0.0_amd64.deb
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `repository_url` | **Yes** | — | Full URL of the Nexus APT repository endpoint |
| `repository_username` | **Yes** | `admin` | Nexus username |
| `repository_password` | **Yes** | — | Nexus password |
| `package_path` | **Yes** | — | Absolute path to the `.deb` file on the runner filesystem |
| `repository_hostname` | No | — | Hostname to send as the `Host` header — required when `repository_url` uses an IP address instead of a domain name |

## Outputs

| Output | Description |
|--------|-------------|
| `status` | Result of the publish operation: `success` or `failed` |

## Examples

### Standard deployment

```yaml
- name: Publish .deb to Nexus
  id: publish
  uses: hashemallaham963/gh-actions/publish-apt@main
  with:
    repository_url: https://nexus.example.com/repository/apt-hosted/
    repository_username: ${{ secrets.NEXUS_USERNAME }}
    repository_password: ${{ secrets.NEXUS_PASSWORD }}
    package_path: ${{ github.workspace }}/dist/myapp_1.2.3_amd64.deb
```

### IP-based Nexus with hostname override

Use `repository_hostname` when your Nexus instance is accessed via IP but requires a specific `Host` header (e.g. behind a reverse proxy or TLS terminator):

```yaml
- name: Publish .deb to Nexus (IP-based)
  id: publish
  uses: hashemallaham963/gh-actions/publish-apt@main
  with:
    repository_url: http://192.168.1.100:8081/repository/apt-hosted/
    repository_username: ${{ secrets.NEXUS_USERNAME }}
    repository_password: ${{ secrets.NEXUS_PASSWORD }}
    package_path: ${{ github.workspace }}/dist/myapp_1.2.3_amd64.deb
    repository_hostname: nexus.internal.example.com
```

### In a full release pipeline

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build .deb package
        run: make deb VERSION=${{ needs.version.outputs.version }}

      - name: Publish to APT
        id: publish
        uses: hashemallaham963/gh-actions/publish-apt@main
        with:
          repository_url: https://nexus.example.com/repository/apt-hosted/
          repository_username: ${{ secrets.NEXUS_USERNAME }}
          repository_password: ${{ secrets.NEXUS_PASSWORD }}
          package_path: ${{ github.workspace }}/dist/myapp_${{ needs.version.outputs.version }}_amd64.deb

      - name: Check result
        run: echo "Publish status: ${{ steps.publish.outputs.status }}"
```

## Notes

- `curl` must be available on the runner. The action exits with an error if it is not found.
- Credentials are passed via HTTP basic auth and are never echoed to logs.
- `package_path` must be an **absolute path** on the runner filesystem. Use `${{ github.workspace }}/...` to build it reliably.
- `repository_hostname` is only needed when the `repository_url` contains an IP address. Omit it for standard domain-based URLs.
