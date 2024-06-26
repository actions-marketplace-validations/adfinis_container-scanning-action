# Container Scanning Action

Action for scanning containers.

## Examples

### Scheduled vuln scanning of containers

```yaml
---
name: Schedule

on:
  schedule:
    - cron: 11 15 * * *
  workflow_dispatch:

jobs:
  scan:
    name: Scan image
    runs-on: ubuntu-latest
    steps:
      - uses: adfinis/container-scanning-action@main
        with:
          image-ref: ghcr.io/adfinis/example # the image to scan
          token: ${{ secrets.GITHUB_TOKEN }}
          attest: true # create cosign security attestation
```

### Semantic release

```yaml
---
name: Release

on:
  push:
    branches: [main]

jobs:
  semrel:
    name: Semantic Release
    runs-on: ubuntu-latest
    permissions:
      actions: none
      checks: none
      contents: none
      deployments: none
      issues: none
      packages: write
      pull-requests: none
      repository-projects: none
      security-events: none
      statuses: none
      id-token: write # needed for signing the images with GitHub OIDC using cosign
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Semantic Release
        uses: go-semantic-release/action@v1
        id: semrel
        with:
          github-token: ${{ secrets.PAT }}

      - name: Bump Version
        if: steps.semrel.outputs.version != ''
        run: |
          pipx run poetry version "${{ steps.semrel.outputs.version }}"

      - name: Docker meta
        id: meta
        if: steps.semrel.outputs.version != ''
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/adfinis/example
          flavor: |
            latest=auto
          tags: |
            type=semver,pattern={{version}},value=${{ steps.semrel.outputs.version }}
            type=semver,pattern={{major}}.{{minor}},value=${{ steps.semrel.outputs.version }}
            type=semver,pattern={{major}},value=${{ steps.semrel.outputs.version }}
          labels: |
            org.opencontainers.image.title=${{ github.event.repository.name }}
            org.opencontainers.image.description=${{ github.event.repository.description }}
            org.opencontainers.image.url=${{ github.event.repository.html_url }}
            org.opencontainers.image.source=${{ github.event.repository.clone_url }}
            org.opencontainers.image.revision=${{ github.sha }}
            org.opencontainers.image.licenses=${{ github.event.repository.license.spdx_id }}

      - name: Login to GHCR
        if: steps.semrel.outputs.version != ''
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.repository_owner }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        if: steps.semrel.outputs.version != ''
        id: docker
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: |
            ${{ steps.meta.outputs.labels }}

      - name: Sign image and attach SBOM attestation
        uses: adfinis/container-scanning-action@main
        with:
          image-ref: ghcr.io/adfinis/example
          token: ${{ secrets.GITHUB_TOKEN }}
          digest: ${{ steps.docker.outputs.digest }}
          attest: true
```

## Workflow

![](./docs/workflow.png)

## License

Code released under the [GNU Lesser General Public License v3.0](./LICENSE)
