# Docker image for building and pushing container images
# Based on the base image with Docker CLI and buildx

ARG BASE_IMAGE=ghcr.io/YOUR_ORG/runner-base:latest
FROM ${BASE_IMAGE}

LABEL org.opencontainers.image.source="https://github.com/YOUR_ORG/github-actions-runner-images"
LABEL org.opencontainers.image.description="Docker build image for GitHub Actions pipelines"
LABEL org.opencontainers.image.licenses="MIT"

# Switch to root for installations
USER root

# Install Docker CLI (not the daemon - we'll use Docker-in-Docker or host Docker socket)
RUN curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg \
    && echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        docker-ce-cli \
        docker-buildx-plugin \
    && rm -rf /var/lib/apt/lists/*

# Install crane (for image manipulation without Docker daemon)
ARG CRANE_VERSION=v0.19.1
RUN curl -sSL "https://github.com/google/go-containerregistry/releases/download/${CRANE_VERSION}/go-containerregistry_Linux_x86_64.tar.gz" | tar xz -C /tmp \
    && mv /tmp/crane /usr/local/bin/crane \
    && mv /tmp/gcrane /usr/local/bin/gcrane \
    && rm -rf /tmp/*

# Install skopeo (for copying images between registries)
RUN apt-get update \
    && apt-get install -y --no-install-recommends skopeo \
    && rm -rf /var/lib/apt/lists/*

# Install hadolint (Dockerfile linter)
ARG HADOLINT_VERSION=v2.12.0
RUN curl -sSL "https://github.com/hadolint/hadolint/releases/download/${HADOLINT_VERSION}/hadolint-Linux-x86_64" -o /usr/local/bin/hadolint \
    && chmod +x /usr/local/bin/hadolint

# Install dive (for analyzing image layers)
ARG DIVE_VERSION=0.12.0
RUN curl -sSL "https://github.com/wagoodman/dive/releases/download/v${DIVE_VERSION}/dive_${DIVE_VERSION}_linux_amd64.tar.gz" | tar xz -C /tmp \
    && mv /tmp/dive /usr/local/bin/dive \
    && rm -rf /tmp/*

# Add runner user to docker group (for socket access when mounted)
RUN groupadd -f docker && usermod -aG docker runner

# Switch back to non-root user
USER runner

# Verify installations
RUN echo "Verifying Docker tools..." \
    && docker --version \
    && crane version \
    && skopeo --version \
    && hadolint --version \
    && dive --version

CMD ["/bin/bash"]
