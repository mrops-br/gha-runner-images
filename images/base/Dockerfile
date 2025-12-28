# Base image for GitHub Actions pipelines
# Multi-stage build for optimized size - ArgoCD-centric workflow
# Removed: kubectl and helm (ArgoCD handles deployments)

# ============================================
# Stage 1: Download all tools
# ============================================
FROM ubuntu:24.04 AS builder

# Set shell with pipefail for safer RUN commands
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Prevent interactive prompts
ENV DEBIAN_FRONTEND=noninteractive

# Install minimal tools needed for downloads
# hadolint ignore=DL3008
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    wget \
    && rm -rf /var/lib/apt/lists/*

# Tool versions
ARG YQ_VERSION=v4.44.1
ARG ARGOCD_VERSION=v3.2.0
ARG GH_VERSION=2.50.0
ARG COSIGN_VERSION=v2.2.4
ARG TRIVY_VERSION=0.52.0

# Create download directory
RUN mkdir -p /downloads

# Download yq
RUN curl -sSL "https://github.com/mikefarah/yq/releases/download/${YQ_VERSION}/yq_linux_amd64" \
    -o /downloads/yq && chmod +x /downloads/yq

# Download kustomize
ARG KUSTOMIZE_VERSION=v5.4.1
RUN curl -sSL "https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2F${KUSTOMIZE_VERSION}/kustomize_${KUSTOMIZE_VERSION}_linux_amd64.tar.gz" \
    | tar xz -C /tmp \
    && mv /tmp/kustomize /downloads/kustomize \
    && chmod +x /downloads/kustomize

# Download ArgoCD CLI
RUN curl -sSL "https://github.com/argoproj/argo-cd/releases/download/${ARGOCD_VERSION}/argocd-linux-amd64" \
    -o /downloads/argocd && chmod +x /downloads/argocd

# Download GitHub CLI
RUN curl -sSL "https://github.com/cli/cli/releases/download/v${GH_VERSION}/gh_${GH_VERSION}_linux_amd64.tar.gz" \
    | tar xz -C /tmp \
    && mv /tmp/gh_${GH_VERSION}_linux_amd64/bin/gh /downloads/gh \
    && chmod +x /downloads/gh

# Download cosign
RUN curl -sSL "https://github.com/sigstore/cosign/releases/download/${COSIGN_VERSION}/cosign-linux-amd64" \
    -o /downloads/cosign && chmod +x /downloads/cosign

# Download trivy
RUN curl -sSL "https://github.com/aquasecurity/trivy/releases/download/v${TRIVY_VERSION}/trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz" \
    | tar xz -C /tmp \
    && mv /tmp/trivy /downloads/trivy \
    && chmod +x /downloads/trivy

# ============================================
# Stage 2: Final runtime image
# ============================================
FROM ubuntu:24.04

LABEL org.opencontainers.image.source="https://github.com/mrops-br/github-actions-runner-images"
LABEL org.opencontainers.image.description="Optimized base image for GitOps pipelines with ArgoCD"
LABEL org.opencontainers.image.licenses="MIT"

# Set shell with pipefail
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Prevent interactive prompts
ENV DEBIAN_FRONTEND=noninteractive

# Set locale
ENV LANG=C.UTF-8
ENV LC_ALL=C.UTF-8

# Install only runtime packages (no build tools)
# hadolint ignore=DL3008
RUN apt-get update && apt-get install -y --no-install-recommends \
    # Essential tools
    ca-certificates \
    curl \
    wget \
    gnupg \
    # Version control
    git \
    git-lfs \
    # Text processing (jq from apt is fine, small package)
    jq \
    gettext \
    # Compression
    zip \
    unzip \
    tar \
    gzip \
    # Network tools
    openssh-client \
    # Python (for scripts)
    python3 \
    python3-pip \
    python3-venv \
    && rm -rf /var/lib/apt/lists/*

# Copy downloaded binaries from builder stage
COPY --from=builder /downloads/* /usr/local/bin/

# Create non-root user for security
RUN useradd -m -s /bin/bash runner \
    && mkdir -p /home/runner/.config \
    && chown -R runner:runner /home/runner

# Set working directory
WORKDIR /workspace

# Default to non-root user
USER runner

# Verify installations
RUN echo "Verifying installations..." \
    && git --version \
    && jq --version \
    && yq --version \
    && kustomize version \
    && argocd version --client \
    && gh --version \
    && cosign version \
    && trivy --version \
    && python3 --version

CMD ["/bin/bash"]
