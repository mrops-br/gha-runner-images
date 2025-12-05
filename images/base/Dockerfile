# Base image for GitHub Actions pipelines
# Contains common tools used across all pipeline types

FROM ubuntu:24.04

LABEL org.opencontainers.image.source="https://github.com/mrops-br/github-actions-runner-images"
LABEL org.opencontainers.image.description="Base image for GitHub Actions pipelines"
LABEL org.opencontainers.image.licenses="MIT"

# Set shell with pipefail for safer RUN commands
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Prevent interactive prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive

# Set locale
ENV LANG=C.UTF-8
ENV LC_ALL=C.UTF-8

# Install base packages
# hadolint ignore=DL3008
RUN apt-get update && apt-get install -y --no-install-recommends \
    # Essential tools
    ca-certificates \
    curl \
    wget \
    gnupg \
    lsb-release \
    software-properties-common \
    # Version control
    git \
    git-lfs \
    # Text processing
    jq \
    gettext \
    # Compression
    zip \
    unzip \
    tar \
    gzip \
    # Network tools
    openssh-client \
    dnsutils \
    # Build essentials
    build-essential \
    # Python (for scripts and tools)
    python3 \
    python3-pip \
    python3-venv \
    && rm -rf /var/lib/apt/lists/*

# Install yq (YAML processor)
ARG YQ_VERSION=v4.44.1
RUN curl -sSL "https://github.com/mikefarah/yq/releases/download/${YQ_VERSION}/yq_linux_amd64" -o /usr/local/bin/yq \
    && chmod +x /usr/local/bin/yq

# Install kubectl
ARG KUBECTL_VERSION=v1.30.0
RUN curl -sSL "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl" -o /usr/local/bin/kubectl \
    && chmod +x /usr/local/bin/kubectl

# Install Helm
ARG HELM_VERSION=v3.15.1
RUN curl -sSL "https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz" | tar xz -C /tmp \
    && mv /tmp/linux-amd64/helm /usr/local/bin/helm \
    && rm -rf /tmp/linux-amd64

# Install kustomize
ARG KUSTOMIZE_VERSION=v5.4.1
RUN curl -sSL "https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2F${KUSTOMIZE_VERSION}/kustomize_${KUSTOMIZE_VERSION}_linux_amd64.tar.gz" | tar xz -C /usr/local/bin

# Install ArgoCD CLI
ARG ARGOCD_VERSION=v2.11.2
RUN curl -sSL "https://github.com/argoproj/argo-cd/releases/download/${ARGOCD_VERSION}/argocd-linux-amd64" -o /usr/local/bin/argocd \
    && chmod +x /usr/local/bin/argocd

# Install GitHub CLI
ARG GH_VERSION=2.50.0
RUN curl -sSL "https://github.com/cli/cli/releases/download/v${GH_VERSION}/gh_${GH_VERSION}_linux_amd64.tar.gz" | tar xz -C /tmp \
    && mv /tmp/gh_${GH_VERSION}_linux_amd64/bin/gh /usr/local/bin/gh \
    && rm -rf /tmp/gh_${GH_VERSION}_linux_amd64

# Install cosign (for image signing)
ARG COSIGN_VERSION=v2.2.4
RUN curl -sSL "https://github.com/sigstore/cosign/releases/download/${COSIGN_VERSION}/cosign-linux-amd64" -o /usr/local/bin/cosign \
    && chmod +x /usr/local/bin/cosign

# Install trivy (security scanner)
ARG TRIVY_VERSION=0.52.0
RUN curl -sSL "https://github.com/aquasecurity/trivy/releases/download/v${TRIVY_VERSION}/trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz" | tar xz -C /tmp \
    && mv /tmp/trivy /usr/local/bin/trivy \
    && rm -rf /tmp/contrib

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
    && kubectl version --client \
    && helm version \
    && kustomize version \
    && argocd version --client \
    && gh --version \
    && cosign version \
    && trivy --version

CMD ["/bin/bash"]
