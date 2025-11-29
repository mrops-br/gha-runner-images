# GitHub Actions Runner Images

Custom container images for GitHub Actions pipelines, optimized for Kubernetes-based self-hosted runners.

## Available Images

| Image | Description | Base |
|-------|-------------|------|
| `runner-base` | Common DevOps tools (kubectl, helm, kustomize, argocd, etc.) | `ubuntu:24.04` |
| `runner-docker` | Docker build tools (docker CLI, buildx, crane, skopeo) | `runner-base` |

## Image Registry

All images are published to GitHub Container Registry:

```
ghcr.io/YOUR_ORG/runner-base:latest
ghcr.io/YOUR_ORG/runner-docker:latest
```

## Tags

Each image is tagged with:

- `latest` - Latest build from main branch
- `<sha>` - Git commit SHA (short format)
- `v1.0.0` - Semantic version (from git tags)
- `v1.0` - Major.minor version
- `v1` - Major version only

## Tools Included

### Base Image (`runner-base`)

**Core Tools:**
- git, git-lfs
- curl, wget
- jq, yq
- zip, unzip, tar

**Kubernetes:**
- kubectl v1.30.0
- helm v3.15.1
- kustomize v5.4.1
- argocd v2.11.2

**Security:**
- cosign v2.2.4
- trivy v0.52.0

**CI/CD:**
- GitHub CLI (gh)
- Python 3

### Docker Image (`runner-docker`)

Inherits all tools from `runner-base`, plus:

- Docker CLI + Buildx
- crane / gcrane
- skopeo
- hadolint (Dockerfile linter)
- dive (image layer analyzer)

## Usage in GitHub Actions

### Using the base image

```yaml
jobs:
  deploy:
    runs-on: [self-hosted, shared-runners]
    container:
      image: ghcr.io/YOUR_ORG/runner-base:latest
    steps:
      - uses: actions/checkout@v4
      - run: kubectl get pods
```

### Using the docker image

```yaml
jobs:
  build:
    runs-on: [self-hosted, shared-runners]
    container:
      image: ghcr.io/YOUR_ORG/runner-docker:latest
      options: --privileged  # Required for Docker-in-Docker
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t myapp .
```

### With Docker socket mounting (alternative to privileged)

```yaml
jobs:
  build:
    runs-on: [self-hosted, shared-runners]
    container:
      image: ghcr.io/YOUR_ORG/runner-docker:latest
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t myapp .
```

## Adding New Images

1. Create a new directory under `images/`:
   ```
   images/
   └── your-image/
       └── Dockerfile
   ```

2. Create a workflow file `.github/workflows/build-your-image.yaml` (copy from existing)

3. Add the new image to `build-all.yaml`

4. Update this README

## Security

- All images are scanned with Trivy for vulnerabilities
- Images are signed with cosign (keyless, using GitHub OIDC)
- SBOM (Software Bill of Materials) is generated for each build
- Non-root user (`runner`) is used by default

### Verifying image signatures

```bash
cosign verify ghcr.io/YOUR_ORG/runner-base:latest \
  --certificate-identity-regexp='https://github.com/YOUR_ORG/github-actions-runner-images/.github/workflows/*' \
  --certificate-oidc-issuer='https://token.actions.githubusercontent.com'
```

## Local Development

### Building locally

```bash
# Build base image
docker build -t runner-base:local images/base/

# Build docker image (requires base to exist)
docker build -t runner-docker:local \
  --build-arg BASE_IMAGE=runner-base:local \
  images/docker/
```

### Testing locally

```bash
# Run base image
docker run -it --rm runner-base:local

# Test tools
kubectl version --client
helm version
kustomize version
```

## Maintenance

### Updating tool versions

Edit the `ARG` statements in the Dockerfiles:

```dockerfile
ARG KUBECTL_VERSION=v1.30.0
ARG HELM_VERSION=v3.15.1
```

Then push to trigger a rebuild.

### Scheduled rebuilds

Consider adding a scheduled workflow to rebuild images weekly for security patches:

```yaml
on:
  schedule:
    - cron: '0 0 * * 0'  # Weekly on Sunday
```

## License

MIT
