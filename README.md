# GitHub Actions Runner Images

Custom container images for GitHub Actions pipelines, optimized for Kubernetes-based self-hosted runners.

## Available Images

| Image | Description | Base |
|-------|-------------|------|
| `runner-base` | Common DevOps tools (kubectl, helm, kustomize, argocd, etc.) | `ubuntu:24.04` |

## Image Registry

All images are published to GitHub Container Registry:

```
ghcr.io/mrops-br/runner-base:latest
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

## Usage in GitHub Actions

### Using in a workflow

```yaml
jobs:
  deploy:
    runs-on: [self-hosted, shared-runners]
    container:
      image: ghcr.io/mrops-br/runner-base:latest
    steps:
      - uses: actions/checkout@v4
      - run: kubectl get pods
      - run: helm list
```

## Adding New Images

1. Create a new directory under `images/`:
   ```
   images/
   └── your-image/
       └── Dockerfile
   ```

2. Create a workflow file `.github/workflows/build-your-image.yaml` (copy from existing)

3. Add the new image to `release.yaml`

4. Update this README

## Security

- All images are scanned with Trivy for vulnerabilities
- Images are signed with cosign (keyless, using GitHub OIDC)
- SBOM (Software Bill of Materials) is generated for each build
- Non-root user (`runner`) is used by default

### Verifying image signatures

```bash
cosign verify ghcr.io/mrops-br/runner-base:latest \
  --certificate-identity-regexp='https://github.com/mrops-br/gha-runner-images/.github/workflows/*' \
  --certificate-oidc-issuer='https://token.actions.githubusercontent.com'
```

## Local Development

### Building locally

```bash
# Build base image
docker build -t runner-base:local images/base/
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
