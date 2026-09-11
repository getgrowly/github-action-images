# Self-hosted GitHub Actions runner (ARC) with a GitOps toolchain.
#
# Base image (verified on 2.337.0 / Ubuntu 24.04) already provides:
#   docker CLI + buildx, git, curl, jq, python3, unzip, tar, gpg, ssh, sudo
# so none of those are installed again here. What it does NOT have, and
# what pipelines in this org reach for:
#   docker compose, helm, kubectl, gh, PyYAML, git-lfs, make, rsync, zip, xz
#
# Everything added here is something a pipeline actually runs. Resist
# "might be useful" tooling: this image is pulled on every cold runner pod,
# so its size is startup latency.
#
# Build locally:
#   docker build -t runner-devops runner/devops
#
# The RUNNER_VERSION default below only serves that local build. CI passes
# the value from .github/workflows/build-and-push.yml, which is also the
# tag the image is published under.

ARG RUNNER_VERSION=2.337.0
FROM ghcr.io/actions/actions-runner:${RUNNER_VERSION}

USER root
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
ENV DEBIAN_FRONTEND=noninteractive

# python3-yaml comes from apt on purpose. Ubuntu 24.04 marks the system
# interpreter as externally managed (PEP 668), so pip needs
# --break-system-packages and then quietly conflicts with apt on the next
# upgrade. The apt package is the same library without that fight.
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      git-lfs \
      make \
      python3-yaml \
      rsync \
      xz-utils \
      zip \
 && rm -rf /var/lib/apt/lists/*

# docker compose -- the base image ships buildx as its only CLI plugin, and
# `docker compose` is not a fallback for `docker-compose`: it resolves
# through the plugin directory or not at all. Pipelines here run their
# checks inside compose services, so without this the very first step dies
# with "docker: unknown command: docker compose".
ARG COMPOSE_VERSION=v5.5.1
RUN curl -fsSLo /usr/local/lib/docker/cli-plugins/docker-compose \
      "https://github.com/docker/compose/releases/download/${COMPOSE_VERSION}/docker-compose-linux-x86_64" \
 && chmod 0755 /usr/local/lib/docker/cli-plugins/docker-compose

# helm -- MATCH ARGO CD'S BUNDLED HELM, not "latest" and not "whatever we
# had". CI renders charts (gitops ci/render.py) to predict what Argo CD
# will apply; if the two binaries differ, CI can go green on a manifest the
# cluster renders differently -- the exact failure the render step exists
# to prevent.
#
# The version to match is whatever the running repo-server has:
#   kubectl -n argocd exec deploy/argocd-repo-server -- helm version
# Argo CD v3.5.1 -> helm v4.2.1 (checked 2026-09-11). Bump this when Argo
# CD is upgraded, in the same change.
ARG HELM_VERSION=v4.2.1
RUN curl -fsSL "https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz" \
      | tar -xz -C /tmp linux-amd64/helm \
 && install -m 0755 /tmp/linux-amd64/helm /usr/local/bin/helm \
 && rm -rf /tmp/linux-amd64

# kubectl -- used client-side only (`kubectl kustomize`). Kept on the
# cluster's minor so the same binary also works for a one-off against it;
# kubectl supports one minor of skew in either direction.
ARG KUBECTL_VERSION=v1.36.4
RUN curl -fsSLo /usr/local/bin/kubectl \
      "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl" \
 && chmod 0755 /usr/local/bin/kubectl

# gh -- `gh api` in scripts, PR comments, release notes.
ARG GH_VERSION=2.100.0
RUN curl -fsSL "https://github.com/cli/cli/releases/download/v${GH_VERSION}/gh_${GH_VERSION}_linux_amd64.tar.gz" \
      | tar -xz -C /tmp "gh_${GH_VERSION}_linux_amd64/bin/gh" \
 && install -m 0755 "/tmp/gh_${GH_VERSION}_linux_amd64/bin/gh" /usr/local/bin/gh \
 && rm -rf "/tmp/gh_${GH_VERSION}_linux_amd64"

# Fail the build here rather than in somebody's pipeline at 2am.
RUN git --version \
 && git-lfs --version \
 && python3 -c 'import yaml; print("pyyaml", yaml.__version__)' \
 && helm version --short \
 && kubectl version --client=true \
 && gh --version | head -1 \
 && docker --version \
 && docker buildx version \
 && docker compose version

USER runner
WORKDIR /home/runner
