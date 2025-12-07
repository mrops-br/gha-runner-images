# GitHub Actions Runner Images

Custom container image for GitHub Actions pipelines, optimized for Kubernetes-based self-hosted runners.

Built with **multi-stage builds** for minimal size and **ArgoCD-centric GitOps** workflows.

## Available Images

| Image | Description | Size | Base |
|-------|-------------|------|------|
| `runner-base` | Optimized for GitOps pipelines with buildkit | ~400MB | `ubuntu:24.04` |

## Image Registry

Published to GitHub Container Registry:

```bash
ghcr.io/mrops-br/runner-base:latest
```

## Tags

Each image is tagged with:

- `latest` - Latest production release from main branch
- `dev-latest` - Latest development build from develop branch
- `dev-<sha>` - Development build with commit SHA
- `<sha>` - Git commit SHA (short format, immutable)
- `v1.0.0` - Semantic version (from git tags)
- `v1.0` - Major.minor version
- `v1` - Major version only
- `1.0.0-rc` - Release candidate (from release/* branches)
- `1.0.0-hotfix-rc` - Hotfix release candidate

## Tools Included

### Base Image (`runner-base`)

**Core Tools:**
- git, git-lfs
- curl, wget
- jq, yq (YAML processor)
- zip, unzip, tar

**GitOps & CI/CD:**
- argocd CLI v2.11.2 (for syncing deployments)
- GitHub CLI (gh) v2.50.0

**Security:**
- cosign v2.2.4 (keyless image signing)
- trivy v0.52.0 (vulnerability scanning)

**Runtime:**
- Python 3 + pip

**Intentionally Excluded:**
- ~~kubectl~~ - Use `azure/setup-kubectl` action if needed (ArgoCD handles deployments)
- ~~helm~~ - Use `azure/setup-helm` action if needed (ArgoCD handles deployments)
- ~~kustomize~~ - ArgoCD has built-in kustomize
- ~~Docker CLI~~ - Use buildkit for image builds (via `docker/build-push-action`)

## Usage in GitHub Actions

### Strategy: Hybrid Approach

We use a **hybrid approach** combining custom image with setup actions and buildkit:

- **Custom image** for slow-to-install tools (argocd, trivy, cosign)
- **Setup actions** for fast-installing tools (kubectl, helm) when needed
- **Buildkit** for container builds (no Docker-in-Docker needed)

### GitOps Workflows

```yaml
jobs:
  deploy:
    runs-on: [self-hosted, shared-runners]
    container:
      image: ghcr.io/mrops-br/runner-base:latest
      credentials:
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - uses: actions/checkout@v4

      # Tools already available: git, jq, yq, argocd, gh, cosign, trivy
      - name: Update GitOps repo
        run: |
          yq eval '.images[0].newTag = "${{ github.sha }}"' -i kustomization.yaml
          git commit -am "Update image tag"
          git push

      - name: Sync ArgoCD
        run: |
          argocd app sync my-app --wait
```

### Container Builds (with Buildkit)

```yaml
jobs:
  build:
    runs-on: [self-hosted, shared-runners]
    steps:
      - uses: actions/checkout@v4

      # Lint Dockerfile (using hadolint action)
      - uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile

      # Build and push with buildkit
      - uses: docker/setup-buildx-action@v3
        with:
          driver: remote
          endpoint: tcp://buildkitd.github-systems.svc.cluster.local:1234

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        id: build
        with:
          context: .
          push: true
          tags: ghcr.io/org/app:${{ github.sha }}
          provenance: true
          sbom: true

      # Sign image (cosign available in runner)
      - run: cosign sign --yes ghcr.io/org/app@${{ steps.build.outputs.digest }}

      # Scan for vulnerabilities (trivy available in runner)
      - run: trivy image ghcr.io/org/app:${{ github.sha }}
```

### Optional: Using Setup Actions

If you need kubectl/helm occasionally:

```yaml
jobs:
  deploy:
    runs-on: [self-hosted, shared-runners]
    container:
      image: ghcr.io/mrops-br/runner-base:latest
    steps:
      - uses: actions/checkout@v4

      # Install kubectl only when needed
      - uses: azure/setup-kubectl@v3
        with:
          version: 'v1.30.0'

      - run: kubectl get pods
```

## Architecture

### Multi-Stage Builds

All images use multi-stage builds for optimization:

```dockerfile
# Stage 1: Download tools
FROM ubuntu:24.04 AS builder
RUN mkdir /downloads
RUN curl ... -o /downloads/tool

# Stage 2: Minimal runtime
FROM ubuntu:24.04
COPY --from=builder /downloads/* /usr/local/bin/
```

**Benefits:**
- Smaller final images (no build tools)
- Faster pulls and startup
- Reduced attack surface

### Image Architecture

```
ubuntu:24.04
    └── runner-base (GitOps tools)
        - Buildkit used externally for container builds
        - No Docker-in-Docker complexity
```

## Adding New Images

1. Create a new directory under `images/`:
   ```
   images/
   └── your-image/
       └── Dockerfile
   ```

2. Use multi-stage build pattern (see existing images)

3. Create workflow `.github/workflows/build-your-image.yaml`:
   ```yaml
   name: Build Your Image
   on:
     push:
       branches: [develop]
       paths: ['images/your-image/**']
   # ... (copy from build-base.yaml or build-docker.yaml)
   ```

4. Add retag job to `release.yaml`:
   ```yaml
   retag-your-image:
     needs: [extract-version, retag-base]
     uses: ./.github/workflows/build-your-image.yaml
     with:
       mode: retag
       source_tag: ${{ needs.extract-version.outputs.source_tag }}
       version: ${{ needs.extract-version.outputs.version }}
   ```

5. Update this README

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
# Test base image
docker run -it --rm runner-base:local bash
> git --version
> jq --version
> yq --version
> argocd version --client
> gh --version
> cosign version
> trivy --version
> python3 --version
```

### Comparing image sizes

```bash
# Check layer sizes
dive runner-base:local

# Compare with previous version
docker images | grep runner
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
