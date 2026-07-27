# Self-hosted Executor, single container, no external services.
# Build context is the repo root because Bun workspaces need every member:
#   docker build -f apps/host-selfhost/Dockerfile -t executor-selfhost .
#
# Runtime needs this container plus a volume for the data dir:
#   docker run -p 4788:4788 -e BETTER_AUTH_SECRET=$(openssl rand -hex 32) \
#     -e EXECUTOR_BOOTSTRAP_ADMIN_EMAIL=you@example.com \
#     -e EXECUTOR_BOOTSTRAP_ADMIN_PASSWORD=... \
#     -e EXECUTOR_WEB_BASE_URL=https://your.domain \
#     -v executor-data:/data executor-selfhost

ARG EXECUTOR_BUN_VERSION=1.3.11
ARG MISE_VERSION=2026.7.0

FROM oven/bun:${EXECUTOR_BUN_VERSION} AS prod-deps
WORKDIR /app
COPY . .
RUN bun install --frozen-lockfile --production --ignore-scripts --filter @executor-js/host-selfhost \
  && bun run apps/host-selfhost/scripts/package-runtime.ts

FROM oven/bun:${EXECUTOR_BUN_VERSION} AS build
WORKDIR /app
COPY . .
RUN apt-get update \
  && apt-get install --no-install-recommends --yes \
    build-essential \
    python3 \
  && rm -rf /var/lib/apt/lists/*
ENV PYTHON=/usr/bin/python3
RUN bun install --frozen-lockfile
RUN cd apps/host-selfhost && bun run build

FROM debian:12.11-slim@sha256:b1a741487078b369e78119849663d7f1a5341ef2768798f7b7406c4240f86aef AS runtime
ARG TARGETARCH
ARG MISE_VERSION
WORKDIR /app
LABEL org.opencontainers.image.source="https://github.com/ramarivera/executor-selfhost" \
      org.opencontainers.image.description="Single-container self-hosted Executor" \
      org.opencontainers.image.licenses="MIT"
ARG EXECUTOR_ALLOW_STDIO_MCP=false
ENV NODE_ENV=production \
    EXECUTOR_HOST=0.0.0.0 \
    PORT=4788 \
    EXECUTOR_DATA_DIR=/data \
    EXECUTOR_ALLOW_STDIO_MCP=${EXECUTOR_ALLOW_STDIO_MCP} \
    MISE_DATA_DIR=/opt/mise \
    MISE_CONFIG_FILE=/etc/mise/config.toml \
    MISE_TRUSTED_CONFIG_PATHS=/etc/mise \
    PATH=/opt/mise/shims:/opt/mise/bin:/usr/local/bin:/usr/local/sbin:/usr/sbin:/usr/bin:/sbin:/bin

# The runtime intentionally carries a trusted local studio toolchain. Stdio MCP
# is already an explicit opt-in in executor.config.ts; this gives those MCPs
# predictable interpreters and compilers without adding them to Executor's app.
# Cross-reference: apps/host-selfhost/mise.toml
RUN apt-get update \
  && apt-get install --no-install-recommends --yes \
    bash \
    build-essential \
    ca-certificates \
    curl \
    git \
    libssl3 \
    libstdc++6 \
    openssh-client \
    pkg-config \
    unzip \
    xz-utils \
  && rm -rf /var/lib/apt/lists/*

RUN set -eux; \
  case "${TARGETARCH}" in \
    amd64) mise_arch=x64 ;; \
    arm64) mise_arch=arm64 ;; \
    *) echo "Unsupported TARGETARCH: ${TARGETARCH}" >&2; exit 1 ;; \
  esac; \
  mise_archive="mise-v${MISE_VERSION}-linux-${mise_arch}.tar.gz"; \
  curl --fail --silent --show-error --location \
    --output /tmp/mise.tar.gz \
    "https://github.com/jdx/mise/releases/download/v${MISE_VERSION}/${mise_archive}"; \
  curl --fail --silent --show-error --location \
    --output /tmp/SHASUMS256.txt \
    "https://github.com/jdx/mise/releases/download/v${MISE_VERSION}/SHASUMS256.txt"; \
  mise_sha="$(awk -v archive="${mise_archive}" '$2 == archive || $2 == "*" archive { print $1 }' /tmp/SHASUMS256.txt)"; \
  test -n "${mise_sha}"; \
  printf '%s  %s\n' "${mise_sha}" /tmp/mise.tar.gz | sha256sum --check; \
  mkdir -p /tmp/mise /usr/local/bin /opt/mise; \
  tar --extract --gzip --file /tmp/mise.tar.gz --directory /tmp/mise; \
  install /tmp/mise/mise/bin/mise /usr/local/bin/mise; \
  rm -rf /tmp/mise /tmp/mise.tar.gz /tmp/SHASUMS256.txt

COPY apps/host-selfhost/mise.toml /etc/mise/config.toml
RUN mise install --yes \
  && mise reshim \
  && mise --version \
  && node --version \
  && bun --version \
  && python --version \
  && rustc --version \
  && cargo --version \
  && uv --version \
  && rm -rf /opt/mise/cache

COPY --from=prod-deps /app/.selfhost-runtime /app
COPY --from=build /app/apps/host-selfhost/dist /app/apps/host-selfhost/dist
WORKDIR /app/apps/host-selfhost
VOLUME ["/data"]
EXPOSE 4788
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=5 \
  CMD bun -e "fetch('http://127.0.0.1:4788/api/health').then(r=>process.exit(r.ok?0:1),()=>process.exit(1))"
CMD ["bun", "run", "dist-server/serve.js"]
