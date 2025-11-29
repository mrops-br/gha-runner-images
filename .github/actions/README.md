# Composite Actions

This directory contains reusable composite actions for the runner image build workflows.

## Available Actions

### 1. lint-dockerfile

Lints a Dockerfile using hadolint.

**Inputs:**
- `dockerfile` (required): Path to the Dockerfile
- `failure-threshold` (optional): Failure threshold (error, warning, info, style, ignore, none). Default: `warning`

**Example:**
```yaml
- name: Lint Dockerfile
  uses: ./.github/actions/lint-dockerfile
  with:
    dockerfile: images/base/Dockerfile
    failure-threshold: warning
```

### 2. docker-build-push

Builds, tags, and pushes a Docker image with metadata, SBOM, and provenance.

**Inputs:**
- `context` (required): Build context directory
- `dockerfile` (required): Path to Dockerfile
- `registry` (required): Container registry
- `image-name` (required): Image name (without registry prefix)
- `push` (optional): Whether to push the image. Default: `false`
- `platforms` (optional): Target platforms. Default: `linux/amd64`
- `build-args` (optional): Build arguments (multiline)
- `registry-username` (optional): Registry username. Default: empty
- `registry-password` (optional): Registry password or token. Default: empty

**Outputs:**
- `digest`: Image digest
- `tags`: Image tags (newline separated)
- `image-ref`: Full image reference with digest

**Example:**
```yaml
- name: Build and push Docker image
  id: build
  uses: ./.github/actions/docker-build-push
  with:
    context: images/base
    dockerfile: images/base/Dockerfile
    registry: ghcr.io
    image-name: myorg/runner-base
    push: 'true'
    platforms: linux/amd64
    registry-username: ${{ github.actor }}
    registry-password: ${{ secrets.GITHUB_TOKEN }}
```

### 3. sign-image

Signs a container image using cosign with keyless signing.

**Inputs:**
- `image-digest` (required): Image digest to sign
- `image-tags` (required): Image tags (space or newline separated)

**Example:**
```yaml
- name: Sign image
  uses: ./.github/actions/sign-image
  with:
    image-digest: ${{ steps.build.outputs.digest }}
    image-tags: ${{ steps.build.outputs.tags }}
```

### 4. scan-image

Scans a container image for vulnerabilities using Trivy and optionally uploads results to GitHub Security.

**Inputs:**
- `image-ref` (required): Full image reference (with digest)
- `severity` (optional): Severities to scan for (comma separated). Default: `CRITICAL,HIGH`
- `format` (optional): Output format (table, json, sarif, etc.). Default: `sarif`
- `output` (optional): Output file path. Default: `trivy-results.sarif`
- `upload-sarif` (optional): Whether to upload SARIF results to GitHub Security. Default: `true`

**Example:**
```yaml
- name: Scan image for vulnerabilities
  uses: ./.github/actions/scan-image
  with:
    image-ref: ghcr.io/myorg/runner-base@sha256:abc123...
    severity: 'CRITICAL,HIGH'
```

## Complete Workflow Example

```yaml
name: Build Image
on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Lint Dockerfile
        uses: ./.github/actions/lint-dockerfile
        with:
          dockerfile: images/base/Dockerfile

  build:
    runs-on: [self-hosted, shared-runners]
    needs: lint
    permissions:
      contents: read
      packages: write
      id-token: write
      security-events: write
    steps:
      - uses: actions/checkout@v4

      # 1. Build and push
      - name: Build image
        id: build
        uses: ./.github/actions/docker-build-push
        with:
          context: images/base
          dockerfile: images/base/Dockerfile
          registry: ghcr.io
          image-name: myorg/runner-base
          push: ${{ github.event_name != 'pull_request' }}
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}

      # 2. Sign (only if pushed)
      - name: Sign image
        if: github.event_name != 'pull_request'
        uses: ./.github/actions/sign-image
        with:
          image-digest: ${{ steps.build.outputs.digest }}
          image-tags: ${{ steps.build.outputs.tags }}

      # 3. Scan (only if pushed)
      - name: Scan image
        if: github.event_name != 'pull_request'
        uses: ./.github/actions/scan-image
        with:
          image-ref: ${{ steps.build.outputs.image-ref }}
```

## Design Principles

1. **Composability**: Each action does one thing well
2. **Reusability**: Actions can be used across different workflows
3. **Maintainability**: Update once, benefits all workflows
4. **Security**: Built-in signing and scanning capabilities
5. **Git Flow Compatible**: Works with all Git Flow branches
