# syntax=docker/dockerfile:1
#
# Urkon docker sandbox image. Built on demand by `urkon watch` (docker backend):
# one stage per agent, selected with `docker build --target <agent>`. The image is
# tagged by this file's content hash, so urkon rebuilds automatically whenever you
# edit it (including a bumped *_VERSION below). Set [sandbox].image in
# .urkon/settings.toml to skip the build and run a prebuilt image instead.
#
# Each agent stage installs its CLI from the vendor's native installer (no npm) at a
# PINNED version, so builds are reproducible. Installs land in a world-readable /opt
# prefix symlinked onto PATH, so the binary runs under the host uid urkon passes via
# `docker run --user`. Bump a version by editing its ARG, then run `make
# sandbox-smoke`. `make sandbox-versions` reports which pins have gone stale
# against upstream. This file is yours to edit.

FROM debian:stable-slim AS base
RUN apt-get update \
 && apt-get install -y --no-install-recommends git ca-certificates curl unzip \
 && rm -rf /var/lib/apt/lists/*

# urkon runs the container with `--user <host-uid>:<host-gid>` so writes to the
# mounted worktree work — that overrides the USER below. urkon also mounts a
# persistent per-agent home (.urkon/sandbox/home/<agent>) and sets HOME to it, so
# agent settings/cache survive across runs; override HOME in [sandbox.env] if you
# want it elsewhere. [sandbox].paths_ro / paths_rw mounts bind at their absolute
# host path.
RUN useradd -m -d /home/urkon urkon
USER urkon
WORKDIR /workspace

# --- per-agent stages: each installs its pinned CLI onto PATH ---------------------

FROM base AS claude
# Claude Code — native installer, version = first arg. https://claude.ai/install.sh
ARG CLAUDE_VERSION=2.1.206
USER root
RUN export HOME=/opt/claude && mkdir -p /opt/claude \
 && curl -fsSL https://claude.ai/install.sh | bash -s -- "$CLAUDE_VERSION" \
 && ln -sf /opt/claude/.local/bin/claude /usr/local/bin/claude \
 && chmod -R a+rX /opt/claude
USER urkon
# Smoke test: fail the build if the CLI is broken. A throwaway HOME, removed after,
# so no build-user-owned cache is baked in (it would block a different --user uid).
RUN mkdir -p /tmp/smoke && HOME=/tmp/smoke claude --version && rm -rf /tmp/smoke

FROM base AS amp
# Amp — native installer (npm is deprecated); AMP_VERSION + AMP_HOME.
ARG AMP_VERSION=0.0.1783687353-g1c35ad
USER root
RUN export AMP_HOME=/opt/amp AMP_VERSION="$AMP_VERSION" \
 && curl -fsSL https://ampcode.com/install.sh | bash \
 && ln -sf /opt/amp/bin/amp /usr/local/bin/amp \
 && chmod -R a+rX /opt/amp
USER urkon
RUN mkdir -p /tmp/smoke && HOME=/tmp/smoke amp --version && rm -rf /tmp/smoke

FROM base AS opencode
# OpenCode — native installer; VERSION env, installs to $HOME/.opencode/bin.
ARG OPENCODE_VERSION=1.17.18
USER root
RUN export HOME=/opt/opencode VERSION="$OPENCODE_VERSION" && mkdir -p /opt/opencode \
 && curl -fsSL https://opencode.ai/install | bash \
 && ln -sf /opt/opencode/.opencode/bin/opencode /usr/local/bin/opencode \
 && chmod -R a+rX /opt/opencode
USER urkon
RUN mkdir -p /tmp/smoke && HOME=/tmp/smoke opencode --version && rm -rf /tmp/smoke
