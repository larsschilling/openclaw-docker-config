###############################################################################
# OpenClaw Gateway – Docker Image (Multi-Stage Build, CPU-Optimized)
###############################################################################

# Stage 1: Builder - Python packages for CPU-only
FROM python:3.11-slim-bookworm AS python-builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl git build-essential \
    && rm -rf /var/lib/apt/lists/*

# Create virtual environment and install CPU-only packages
RUN python3 -m venv /opt/venv \
    && /opt/venv/bin/pip install --no-cache-dir --upgrade pip \
    && /opt/venv/bin/pip install --no-cache-dir \
       --extra-index-url https://download.pytorch.org/whl/cpu \
       faster-whisper

# Stage 2: Bun builder
FROM node:24-bookworm-slim AS bun-builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl ca-certificates unzip \
    && rm -rf /var/lib/apt/lists/*

USER node
RUN curl -fsSL https://bun.sh/install | bash

# Stage 3: Final minimal image
FROM node:24-bookworm-slim AS final


# renovate: datasource=npm depName=openclaw versioning=loose
ARG OPENCLAW_VERSION=2026.3.12

# Install minimal runtime dependencies + headless chromium
RUN apt-get update && apt-get install -y --no-install-recommends \
    git curl ca-certificates gnupg tini \
    python3 ffmpeg \
    # Headless chromium shell (kein GUI, ~60MB kleiner)
    chromium-headless-shell \
    chromium-common \
    && rm -rf /var/lib/apt/lists/*

# GitHub CLI
RUN curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
    | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
    > /etc/apt/sources.list.d/github-cli.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends gh \
    && rm -rf /var/lib/apt/lists/*

# Install npm packages
RUN npm install -g openclaw@${OPENCLAW_VERSION} clawhub

# Setup directories
RUN mkdir -p /home/node/.openclaw/workspace /.clawhub \
    && chown -R node:node /home/node/.openclaw /.clawhub

# Copy from builders
COPY --from=python-builder --chown=node:node /opt/venv /home/node/.venv
COPY --from=bun-builder --chown=node:node /home/node/.bun /home/node/.bun

# Copy configs
COPY --chown=node:node config/skills-manifest.txt /opt/config/
COPY --chown=node:node workspace-templates/ /opt/workspace-templates/
COPY --chown=node:node docker/entrypoint.sh /usr/local/bin/

USER node

ENV PATH="/home/node/.venv/bin:/home/node/.bun/bin:$PATH" \
    BUN_INSTALL="/home/node/.bun" \
    PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium-headless-shell

# Install QMD
RUN bun install -g https://github.com/tobi/qmd

ENTRYPOINT ["tini", "--", "entrypoint.sh"]
